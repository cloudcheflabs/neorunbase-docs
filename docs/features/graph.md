# Graph Traversal & Analytics

NeorunBase exposes graph queries — neighbour expansion, PageRank, reachability fact-checks, and weighted re-ranking — as first-class parts of the engine, so ontology, knowledge-graph, and Graph-RAG workloads compose naturally with BM25 search, vector ANN, and the rest of the relational planner. There is no separate Neo4j or Apache AGE process; an existing edge table answers graph questions from the same SQL surface.

## Why It's Built In

Most agent / RAG stacks bolt a graph database next to their analytical store, then need an ETL pipeline to keep the two in sync and a glue layer at query time to join graph hops with vector hits. NeorunBase folds the graph engine into the coordinator so:

- **One transaction** writes a relation; the same row is visible to graph traversal, BM25 search, vector ANN, and PageRank immediately.
- **One SQL statement** can walk *n* hops, run PageRank, JOIN with the entities table, filter by BM25 / vector, and rank — no glue layer.
- **One operational surface** — encryption, IAM, replication, and Iceberg auto-sync apply unchanged because the edge table is just a table.
- **Bounded blast radius** — every traversal carries explicit `max_depth` / `max_results`; PageRank carries `max_iter`; the coordinator can't be hung by a hub node.

## Edge Table Shape

The traversal expects an edge table with three named columns; any other columns become first-class edge properties (Phase 2.1) that ride along into the CSR adjacency view and are available to weighted PageRank and downstream JOINs.

| Column     | Type   | Required | Meaning                                  |
|------------|--------|----------|------------------------------------------|
| `src_id`   | BIGINT | ✓        | source node primary key (also the shard key) |
| `dst_id`   | BIGINT | ✓        | destination node primary key             |
| `rel_type` | TEXT   | ✓        | optional relationship type (`'is_a'`, …); per-row may be NULL when the graph is untyped |
| `weight`   | DOUBLE | optional | edge weight consumed by weighted PageRank |
| *anything else* | any | optional | arbitrary property columns — encoded into the CSR neighbor block alongside `rel_type` |

```sql
CREATE TABLE relations (
    edge_id  BIGINT PRIMARY KEY,
    src_id   BIGINT NOT NULL,
    dst_id   BIGINT NOT NULL,
    rel_type TEXT,
    weight   DOUBLE,
    valid_from TIMESTAMP
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

## CSR Adjacency — Phase 1 Acceleration Layer

A naive BFS hop is `WHERE src_id IN (1,2,3,…)` — one B-tree lookup per source vertex per hop. For graphs with millions of edges this becomes the dominant cost. NeorunBase materialises a per-shard **Compressed Sparse Row** view of selected edge tables that turns each hop into a sequential neighbor-block read.

### Enabling CSR

CSR is opt-in per table and configured via three properties:

```properties
# Master toggle.
neorunbase.csr.enabled=true

# Comma-separated list of qualified edge tables to materialize.
neorunbase.csr.enabled.tables=public.relations,public.eligibility

# Background compaction interval — rebuilds CSR from edge rows.
neorunbase.csr.compaction.interval.seconds=3600
```

Enabling CSR for a table does three things:

1. Creates a metadata column family `_csr.<table>` mapping `src_id` (8-byte BE) to a 16-byte CSR pointer.
2. Allocates a per-(shard, table) `CsrSegmentManager` that owns encrypted segment files under `{shardDir}/csr/<table>/seg-NNNNN.nrcsr`. Per-segment AES-256-GCM DEKs match the heap layer's envelope encryption — at-rest protection is identical.
3. Schedules a leader-gated background compactor that scans edge rows, groups them by `src_id`, and writes one neighbor block per source. Property columns (anything beyond `src_id`/`dst_id`/`rel_type`/PK) ride into a per-file dictionary so per-edge `weight`, `valid_from`, etc. are available to readers without a separate row lookup.

### Speedup

Phase 1 microbenchmark, 100 000 edges across 10 000 source vertices, single shard, BFS-hop frontier of 100 source ids:

| Path                | p50 latency | p99 latency | Speedup |
|---------------------|-------------|-------------|---------|
| Row-scan (`WHERE src_id IN (...)`) | **400 ms** | 414 ms | baseline |
| CSR pointer lookup + sequential block read | **1.56 ms** | 11.20 ms | **256×** |

The same answer set comes back either way (951 distinct destinations) — CSR is purely an acceleration of the existing query path, falling back to row-scan when a `src_id` is not yet materialised (eg. just-inserted vertices between compaction passes).

### Admin Endpoints

```bash
# Force a CSR rebuild — useful right after bulk-loading edges.
curl -H "Authorization: Bearer $TOKEN" \
     -X POST https://master:8080/admin/graph/csr/compact \
     -d '{"table":"public.relations"}'

# Snapshot of CSR state across the cluster.
curl -H "Authorization: Bearer $TOKEN" \
     https://master:8080/admin/graph/csr/status
```

The status response lists per-(shard, table) entries with segment count, active segment id, and the number of source vertices currently materialised.

### How a CSR-Accelerated Hop Runs

1. The coordinator's `GraphTraversalExecutor` checks whether the target table is on `csr.enabled.tables`. If yes, it dispatches a `CSR_NEIGHBORS_REQ` per data node holding part of the frontier (scatter-gather, same shape as ANN search).
2. Each data node looks up `_csr.<table>[src_id]`, deref's the pointer through `CsrSegmentManager.read`, decrypts the neighbor block, and returns the dst-id list (and any requested property values).
3. The coordinator merges per-node results into the next BFS frontier. `src_id` values not present in the CSR view fall back to a single row-scan, so the answer is always complete even mid-compaction.

## PAGERANK / PERSONALIZED_PAGERANK

Two TVFs expose the relevance signal that "Graph RAG" papers usually call *graph-based re-ranking*:

```sql
-- Global PageRank — every node gets a rank reflecting its overall centrality.
SELECT node_id, rank
FROM PAGERANK(table     => 'public.relations',
              damping   => 0.85,
              max_iter  => 30,
              k         => 200) p
ORDER BY rank DESC;

-- Personalized PageRank — random walk teleports to a query-specific seed.
-- Use this to rank "everything else by relevance to entity X".
SELECT e.name, p.rank
FROM PERSONALIZED_PAGERANK(table   => 'public.relations',
                           seed    => :company_id,
                           damping => 0.85,
                           max_iter => 30,
                           k       => 100) p
JOIN entities e ON e.id = p.node_id
ORDER BY p.rank DESC;
```

Both functions return `(node_id BIGINT, rank DOUBLE)`. The TVF rewriter runs the algorithm at parse time and inlines the result as a derived table, so JOINs and ORDER BY downstream plan through NeorunBase's standard SQL pipeline.

| Argument       | Required          | Meaning                                                  |
|----------------|-------------------|----------------------------------------------------------|
| `table`        | yes               | Fully qualified edge table                               |
| `seed`         | personalised only | Seed node id — random walk teleports here                |
| `damping`      | no (0.85)         | Damping factor in (0, 1)                                 |
| `max_iter`     | no (20)           | Power-iteration cap                                      |
| `epsilon`      | no (1e-6)         | L1 convergence threshold — algorithm exits early on convergence |
| `k`            | no (100)          | Top-K results returned                                   |
| `weight_col`   | no                | Column name to use as edge weight; absent = uniform      |
| `distributed`  | no (`auto`)       | `true` forces BSP distributed; `false` / `auto` use centralised in-memory |

### Weighted Edges

When `weight_col` is set, contribution along an edge `u → v` scales by `weight(u, v) / Σ weights leaving u` instead of the uniform `1 / outdegree(u)`. With every weight equal to 1.0 the algorithm collapses to the textbook unweighted form, so adding a `weight` column is non-breaking.

```sql
SELECT node_id, rank
FROM PAGERANK(table      => 'public.relations',
              weight_col => 'weight',
              k          => 50) p;
```

### Centralised vs. Distributed

| Mode         | When to use                                  | How it runs                                                |
|--------------|----------------------------------------------|------------------------------------------------------------|
| Centralised  | Edge count fits in coordinator memory (~10 M edges typical) | Coordinator fetches all edges via `SELECT src_id, dst_id, weight FROM table`, builds in-memory adjacency, runs power iteration |
| Distributed  | Larger graphs; multi-shard parallelism       | BSP / Pregel-lite — coordinator broadcasts the rank vector, each data node walks its local CSR adjacency, returns a sparse contribution map; the coordinator applies teleport + dangling redistribution and converges |

The distributed mode produces ranks that match the centralised result within `1e-6` (max abs diff) on identical edge sets — the algorithm is mathematically the same, only the data movement changes. Subsequent supersteps send only **deltas** (entries that moved by more than `1e-3 × epsilon` since the last broadcast), which collapses the wire footprint as ranks converge.

```sql
-- Forces the BSP distributed path.
SELECT node_id, rank
FROM PAGERANK(table       => 'public.relations',
              distributed => true,
              weight_col  => 'weight',
              k           => 100) p;
```

The distributed path requires CSR to be enabled on the target table; tables without CSR fall back to centralised with a logged warning.

## GRAPH_PATH_EXISTS — RIG Fact-Check Primitive

Retrieval-Interleaved Generation (RIG) needs a fast "does this claim hold against the graph?" check so an agent can decide whether to ground or block a generated answer. `GRAPH_PATH_EXISTS` answers it in one TVF call:

```sql
-- Did the agent's claim "company X is eligible for program Y within 2 hops via 'eligible_for' edges" hold?
SELECT path_exists FROM GRAPH_PATH_EXISTS(
    edge_table  => 'public.eligibility',
    src         => :company_id,
    dst         => :program_id,
    max_depth   => 2,
    edge_filter => 'eligible_for') v;
```

Returns a single-row, single-column `BOOLEAN`. The function reuses the same BFS engine as `GRAPH_NEIGHBORS`, so cycle detection / depth bound / `max_results` cap all apply. Because the rewriter materialises the result inline (`(SELECT TRUE AS path_exists) AS v` or `… FALSE …`), it composes naturally inside `WHERE EXISTS (…)`, scalar subqueries, or alongside other TVFs.

## Hybrid Retrieval — Graph + BM25 + Vector

The SMBA Graph-RAG specification calls for a single SELECT that runs Hard Filtering (SQL `WHERE`), semantic+keyword retrieval, knowledge-graph re-ranking, and RIG fact-check together. Each component is a TVF or a standard SQL clause; combined they look like this:

```sql
SELECT d.id, d.name,
       0.5 * h.score + 0.3 * p.rank AS final_score
FROM HYBRID_SEARCH(table     => 'public.programs',
                   ts_query  => 'AI R&D',
                   ts_index  => 'idx_programs_text',
                   vec_query => :user_query_embedding,
                   vec_index => 'idx_programs_emb',
                   alpha => 0.4, beta => 0.6, k => 50) h
JOIN PERSONALIZED_PAGERANK(table => 'public.eligibility',
                            seed  => :company_id,
                            k     => 200) p ON p.node_id = h.id
JOIN programs d ON d.id = h.id
JOIN companies c ON c.id = :company_id
WHERE c.region = '경기도'                       -- Hard Filter
  AND c.revenue >= 100000000000                  -- Hard Filter
  AND d.status = 'open'                          -- Hard Filter
  AND (SELECT path_exists FROM GRAPH_PATH_EXISTS(  -- RIG fact-check
         edge_table => 'public.eligibility',
         src => :company_id, dst => d.id,
         max_depth => 2, edge_filter => 'eligible_for') v)
ORDER BY final_score DESC
LIMIT 10;
```

`HYBRID_SEARCH` is documented on its own page — see [Hybrid Search](hybrid-search.md). The combination here is what folds Graph RAG into a single SELECT.

## Admin REST — `POST /admin/graph/neighbors`

For SDK and agent paths that don't want to embed the TVF in a SELECT, the BFS is exposed over the admin HTTP surface:

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
- **PageRank `max_iter` defaults to 20** — sufficient for damping = 0.85 to converge within `1e-6` on most graphs. Distributed mode will still exit early on convergence even when `max_iter` is generous.
- **Cycles are broken by a visited set** — back-edges and self-loops can't grow the frontier indefinitely.
- **`edge_filter` is single-quote escaped** before it hits the per-hop SQL, so a malicious filter value can't break out of the string literal.
- **All TVFs are rewritten on the coordinator side** *before* the planner runs. Quoted strings and SQL comments are skipped verbatim, so a literal like `'GRAPH_NEIGHBORS(...)'` inside a column value is left alone.
- **CSR fallback is automatic** — a `src_id` not yet materialised (added between compaction passes) is served by the original row-scan path, so traversals never miss freshly inserted edges.

## Ontology Agent Cookbook

Agents over a domain ontology benefit from a few standard moves on top of the TVFs:

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

### 2. Re-ranking by Personalised PageRank

Start with the user's anchor entity, score every other entity by graph proximity, and use that score as one term in the final ranking — the SMBA pattern in miniature:

```sql
SELECT d.id,
       0.6 * ts_rank(d.body, plainto_tsquery(:q))
     + 0.3 * (1 / (1 + (d.embedding <=> :emb)))
     + 0.1 * p.rank                                AS score
FROM PERSONALIZED_PAGERANK(table => 'public.ontology',
                            seed  => :concept_id,
                            k     => 200) p
JOIN documents d ON d.id = p.node_id
WHERE d.body @@ plainto_tsquery(:q)
ORDER BY score DESC
LIMIT 10;
```

### 3. Multi-seed fan-out

`GRAPH_NEIGHBORS` takes one seed; for multiple seeds, run several calls and `UNION` (or `UNION ALL` if you want depth weighting per seed):

```sql
WITH n1 AS (SELECT id, depth FROM GRAPH_NEIGHBORS(... seed => :a ...)),
     n2 AS (SELECT id, depth FROM GRAPH_NEIGHBORS(... seed => :b ...))
SELECT id, MIN(depth) AS depth FROM (SELECT * FROM n1 UNION ALL SELECT * FROM n2) GROUP BY id;
```

The shortest distance to *any* seed is usually a better relevance signal than per-seed depth.

## Limits and Roadmap

- **Edges are directional.** If your ontology is undirected (`synonym`), insert both `(a, b)` and `(b, a)` rows. A symmetric edge type is on the roadmap.
- **No path enumeration yet.** `GRAPH_NEIGHBORS` returns reachable nodes and their hop count; it does not return the path. A `GRAPH_PATH` TVF that returns explicit edge sequences is planned.
- **No Cypher dialect.** The current surface is SQL-only. A Cypher front-end is demand-driven; raise an issue if you need it.
- **Distributed PageRank bootstraps the node set via SQL.** A one-time `SELECT DISTINCT src_id ∪ dst_id` runs before the BSP loop. For multi-billion-edge graphs this becomes the dominant cost; a follow-up will push node-set discovery down to the data nodes.
- **Single-edge-type filter.** `GRAPH_NEIGHBORS` filters on one `rel_type` value at a time. Multi-type filtering (`IN (…)`) is a small extension on the roadmap.
