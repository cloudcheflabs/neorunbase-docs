# Graph Traversal

NeorunBase exposes graph queries — neighbour expansion over a relational edge table — as a first-class part of the engine, so ontology, knowledge-graph, and agent-retrieval workloads compose naturally with BM25 search, vector ANN, and the rest of the relational planner. There is no separate Neo4j or Apache AGE process; an existing edge table answers graph questions from the same SQL surface.

## Why It's Built In

Most agent / RAG stacks bolt a graph database next to their analytical store, then need an ETL pipeline to keep the two in sync and a glue layer at query time to join graph hops with vector hits. NeorunBase folds the BFS into the coordinator so:

- **One transaction** writes a relation; the same row is visible to graph traversal, BM25 search, and vector ANN immediately.
- **One SQL statement** can walk `n` hops out from a seed entity, JOIN the result with the entities table, filter by BM25 (`@@`) or vector similarity (`<=>`), and rank — no glue layer.
- **One operational surface** — encryption, IAM, replication, and Iceberg auto-sync apply unchanged because the edge table is just a table.
- **Bounded blast radius** — every traversal carries explicit `max_depth` and `max_results`; the coordinator can't be hung by a hub node.

## Edge Table Shape

The traversal expects an edge table with three named columns:

| Column     | Type   | Meaning                                  |
|------------|--------|------------------------------------------|
| `src_id`   | BIGINT | source node primary key                  |
| `dst_id`   | BIGINT | destination node primary key             |
| `rel_type` | TEXT   | optional relationship type (`'is_a'`, …) |

```sql
CREATE TABLE relations (
    src_id   BIGINT NOT NULL,
    dst_id   BIGINT NOT NULL,
    rel_type TEXT,
    PRIMARY KEY (src_id, dst_id, rel_type)
) SHARD KEY (src_id) SHARDS 16;
```

Sharding by `src_id` lines up with the per-hop expansion query (`WHERE src_id IN (…)`), so each hop fans out only to the shards that own the current frontier.

## GRAPH_NEIGHBORS — SQL Table-Valued Function

The traversal is exposed as a TVF you can use anywhere a table reference is legal — usually inside `FROM` and JOINed against the entities table:

```sql
SELECT e.id, e.name, n.depth
FROM GRAPH_NEIGHBORS(edge_table  => 'public.relations',
                     seed        => 42,
                     max_depth   => 3,
                     edge_filter => 'is_a',
                     max_results => 200) n
JOIN entities e ON e.id = n.id
ORDER BY n.depth, e.name
LIMIT 50;
```

The function returns two columns — `id BIGINT, depth INT` — where `depth = 0` is the seed itself and `depth = k` is reached on the *k*-th hop. Cycles are broken by the visited set; `depth` is therefore the shortest hop-count, not the number of paths.

| Argument        | Required | Meaning                                                      |
|-----------------|----------|--------------------------------------------------------------|
| `edge_table`    | yes      | Fully qualified edge table, e.g. `public.relations`          |
| `seed`          | yes      | Starting node primary key                                    |
| `max_depth`     | yes      | Hop limit — `1` returns direct neighbours only               |
| `edge_filter`   | no       | Restrict expansion to one `rel_type` value                   |
| `max_results`   | no       | Visited-set cap (default 1000); coordinator memory guardrail |

Arguments are named (PostgreSQL `name => value` form). Positional calls are intentionally rejected so the call site stays readable as the schema evolves.

## How It Runs

The coordinator implements the BFS directly, one hop at a time, against the edge table:

```
SELECT DISTINCT dst_id FROM <edge_table>
WHERE src_id IN (<frontier>)
  [AND rel_type = '<edge_filter>']
```

Each hop reuses the standard distributed planner: shard pruning, encryption, IAM checks, and metrics all apply. There is no new RPC opcode and no external graph database to operate. The `GRAPH_NEIGHBORS(...)` call is rewritten in place to `(VALUES (id, depth), …) AS alias(id, depth)` before parsing, so the rest of the query — JOIN, WHERE, ORDER BY, BM25 / vector operators — plans through NeorunBase's normal SQL pipeline.

Because the result is materialised before the outer query runs, traversal cost is controlled by `max_depth` and `max_results`, not by the size of the entities table or the JOIN downstream.

## Hybrid Retrieval — Graph + BM25 + Vector

Knowledge-graph agents typically need three retrieval signals at once: ontology proximity, lexical match, and semantic similarity. A single SQL statement covers all three:

```sql
SELECT e.id,
       e.name,
       ts_rank(e.description, plainto_tsquery('korean', '한국어 검색')) AS lex,
       1 / (1 + (e.embedding <=> :q))                              AS sem,
       n.depth
FROM GRAPH_NEIGHBORS(edge_table => 'public.relations',
                     seed       => 42,
                     max_depth  => 3,
                     edge_filter => 'is_a',
                     max_results => 200) n
JOIN entities e ON e.id = n.id
WHERE e.description @@ plainto_tsquery('korean', '한국어 검색')
ORDER BY (0.3 * lex + 0.5 * sem + 0.2 / (1 + n.depth)) DESC
LIMIT 10;
```

No CDC, no extra service, no eventual consistency window — the ontology, the BM25 index, and the vector index see the same row at the same time.

## Admin REST — `POST /admin/graph/neighbors`

For SDK and agent paths that don't want to embed the TVF in a SELECT, the same BFS is exposed over the admin HTTP surface:

```bash
curl -H "Authorization: Bearer $TOKEN" \
     -H "Content-Type: application/json" \
     -X POST https://master:8080/admin/graph/neighbors \
     -d '{
           "edgeTable":   "public.relations",
           "seed":        42,
           "maxDepth":    3,
           "edgeFilter":  "is_a",
           "maxResults":  200
         }'
```

Response:

```json
{
  "hits": [
    { "id": 42, "depth": 0 },
    { "id": 7,  "depth": 1 },
    { "id": 8,  "depth": 1 },
    { "id": 11, "depth": 2 }
  ],
  "count": 4
}
```

Validation errors return `400`; anything else returns the same `(id, depth)` shape the TVF produces.

## Java Helper — `NeorunGraphClient`

`neorunbase-common` ships a thin HTTP wrapper for Java callers (Mium, ad-hoc tools) that don't want to assemble JSON by hand:

```java
NeorunGraphClient g = new NeorunGraphClient("https://master:8080", jwtToken);

List<NeorunGraphClient.Hit> hits =
        g.neighbors("public.relations", 42L, 3, "is_a", 200);

for (NeorunGraphClient.Hit h : hits) {
    System.out.println(h.id + " @ depth " + h.depth);
}
```

The class only depends on `java.net.http` and Jackson — no extra runtime to deploy.

## Bounds and Safety

- **`max_depth` is required and must be ≥ 1.** A traversal with no upper bound is an outage waiting to happen, so the API refuses the call.
- **`max_results` is required-with-default (`1000`) and capped at the call site.** The visited set lives in coordinator memory; the cap is a guardrail, not a recommendation.
- **Cycles are broken by a visited set** — back-edges and self-loops can't grow the frontier indefinitely.
- **`edge_filter` is single-quote escaped** before it hits the per-hop SQL, so a malicious filter value can't break out of the string literal.
- **The TVF is rewritten on the coordinator side** *before* the planner runs. Quoted strings and SQL comments are skipped verbatim, so a literal like `'GRAPH_NEIGHBORS(...)'` inside a column value is left alone.

## Ontology Agent Cookbook

Agents over a domain ontology benefit from a few standard moves on top of the TVF:

### 1. Concept expansion before a search

When the user says "find me documents about *neural network*", expand the concept node along `is_a` and `synonym` edges before doing BM25 — so a doc tagged `transformer` still matches:

```sql
WITH expanded AS (
    SELECT id FROM GRAPH_NEIGHBORS(edge_table => 'public.ontology',
                                   seed       => :concept_id,
                                   max_depth  => 2,
                                   edge_filter => 'is_a',
                                   max_results => 100)
    UNION
    SELECT id FROM GRAPH_NEIGHBORS(edge_table => 'public.ontology',
                                   seed       => :concept_id,
                                   max_depth  => 1,
                                   edge_filter => 'synonym',
                                   max_results => 100)
)
SELECT d.*
FROM documents d
JOIN document_concepts dc ON dc.doc_id = d.id
WHERE dc.concept_id IN (SELECT id FROM expanded)
  AND d.body @@ plainto_tsquery('english', :query)
ORDER BY ts_rank(d.body, plainto_tsquery('english', :query)) DESC
LIMIT 20;
```

### 2. Re-ranking by ontology distance

When two documents tie on BM25 / vector score, prefer the one closer in the ontology — a strong signal that the agent is on topic:

```sql
SELECT d.id,
       0.6 * ts_rank(d.body, plainto_tsquery(:q))
     + 0.3 * (1 / (1 + (d.embedding <=> :emb)))
     + 0.1 / (1 + n.depth)                            AS score
FROM GRAPH_NEIGHBORS(edge_table => 'public.ontology',
                     seed       => :concept_id,
                     max_depth  => 3) n
JOIN document_concepts dc ON dc.concept_id = n.id
JOIN documents d           ON d.id = dc.doc_id
WHERE d.body @@ plainto_tsquery(:q)
ORDER BY score DESC
LIMIT 10;
```

### 3. Multi-seed fan-out

The TVF takes one seed; for multiple seeds, run several calls and `UNION` (or `UNION ALL` if you want depth weighting per seed):

```sql
WITH n1 AS (SELECT id, depth FROM GRAPH_NEIGHBORS(... seed => :a ...)),
     n2 AS (SELECT id, depth FROM GRAPH_NEIGHBORS(... seed => :b ...))
SELECT id, MIN(depth) AS depth FROM (SELECT * FROM n1 UNION ALL SELECT * FROM n2) GROUP BY id;
```

The shortest distance to *any* seed is usually a better relevance signal than per-seed depth.

## Limits and Roadmap

- **Edges are directional.** If your ontology is undirected (`synonym`), insert both `(a, b)` and `(b, a)` rows. A symmetric edge type is on the roadmap.
- **No path enumeration yet.** `GRAPH_NEIGHBORS` returns reachable nodes and their hop count; it does not return the path. A `GRAPH_PATH` TVF that returns explicit edge sequences is planned.
- **No Cypher dialect.** The current surface is SQL-only. A Cypher front-end (similar to Apache AGE) is demand-driven; raise an issue if you need it.
- **Single-column edge filter.** Only `rel_type` is filtered today. Property-graph filters (e.g. edge weight thresholds) can be added once a use case lands.
