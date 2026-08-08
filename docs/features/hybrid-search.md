# Hybrid Search (Text + Vector)

NeorunBase exposes hybrid retrieval — BM25 over a Lucene index plus ANN over an HNSW vector index — as a single SQL TVF, `HYBRID_SEARCH`. One call returns a blended top-K that composes naturally with `WHERE` (Hard Filter), `JOIN` (metadata enrichment), and other TVFs (graph re-ranking, fact-check). The text and the vector indexes already live inside the same engine that owns the relational data, so there is no out-of-band scoring service and no ETL to keep two systems in sync.

## Why a TVF

A planner-level expression form like `ORDER BY α·ts_rank(...) + β·(emb <=> :q)` was the obvious surface for a single-SQL hybrid search. We chose a TVF instead because:

- **Composability** — `FROM HYBRID_SEARCH(...) h JOIN entity_table d ON d.id = h.id WHERE …` is the SMBA Graph-RAG pipeline expressed in one SELECT. The TVF returns `(id, score)`; everything downstream is plain SQL.
- **Per-call tunables** — `α`, `β`, and the blending strategy (`min-max` / RRF) need to be visible at the call site so application authors can tune them per query without touching the planner.
- **Faster to ship correctly** — the TVF reuses the existing `FtsSearchExecutor` and `VectorSearchExecutor` scatter-gather paths verbatim. A planner-expression integration would have rewired the optimiser; the TVF is a parse-time text rewrite that drops into the existing pipeline.

## HYBRID_SEARCH — Surface

```sql
SELECT d.id, d.title, h.score
FROM HYBRID_SEARCH(table     => 'public.docs',
                   ts_query  => 'AI R&D',
                   ts_index  => 'idx_docs_text',
                   vec_query => '[0.9, 0.1, 0.0, 0.0]',
                   vec_index => 'idx_docs_emb',
                   alpha => 0.4, beta => 0.6, k => 50) h
JOIN docs d ON d.id = h.id
WHERE d.region = '경기도' AND d.revenue >= 100000000000
ORDER BY h.score DESC
LIMIT 10;
```

The TVF returns two columns — `id BIGINT, score DOUBLE` — sorted descending so a downstream `ORDER BY h.score DESC LIMIT k` is a no-op against the materialised result.

| Argument        | Required | Default | Meaning                                                                 |
|-----------------|----------|---------|-------------------------------------------------------------------------|
| `table`         | yes      |         | Fully qualified target table (e.g. `public.docs`)                       |
| `ts_query`      | yes      |         | BM25 / Lucene query string                                              |
| `ts_index`      | yes      |         | FTS index name on `table`                                               |
| `vec_query`     | yes      |         | Vector literal — `'[0.1, 0.2, …]'` (must match the index dimension)     |
| `vec_index`     | yes      |         | HNSW index name on `table`                                              |
| `alpha`         | no       | `0.5`   | FTS weight under min-max blending (ignored under RRF)                   |
| `beta`          | no       | `0.5`   | Vector weight under min-max blending (ignored under RRF)                |
| `k`             | no       | `50`    | Final top-K returned                                                    |
| `per_mode_k`    | no       | `k * 4` | Top-K each side fetches before blend; oversample preserves recall when one signal is stronger |
| `ef_search`     | no       | `0`     | HNSW ef_search parameter (0 = use index default)                        |
| `vec_op_class`  | no       | `''`    | Vector op class override (`vector_l2_ops`, `vector_cosine_ops`, `vector_ip_ops`); empty = index default |
| `blend`         | no       | `minmax` | `minmax` (linear combo of normalised scores) or `rrf` (Reciprocal Rank Fusion) |
| `rrf_k`         | no       | `60`    | RRF damping constant (only used when `blend => rrf`)                    |

Arguments are named (PostgreSQL `name => value` form). The vector literal is a quoted string parsed at rewrite time, so prepared-statement parameter substitution (`?`) for the vector itself is not yet supported — applications encode the embedding into the string before issuing the query.

## Score Blending — min-max vs RRF

Two blending strategies. Pick the one whose failure mode is least painful for your data.

### min-max (default)

Each side's top-K scores are min-max normalised into `[0, 1]`:

- **FTS BM25** — descending score; raw range is open, normalised to `1` for the top hit and `0` for the worst in the per-mode top-K.
- **Vector distance** — ascending; raw range is `[0, ∞)`. The codec inverts to similarity (`1 - dist_norm`) so higher is better in both modes.

Final score = `α · fts_norm + β · vec_norm`. Documents that appear in only one mode get `0` for the missing side, so co-presence in both lists wins ties — a desirable bias for hybrid retrieval.

### RRF (Reciprocal Rank Fusion)

Rank-based fusion that ignores the raw scores entirely:

```
score(doc) = Σ over modes of  1 / (rrf_k + rank_in_mode)
```

`rrf_k = 60` is the canonical constant. RRF is robust against pathological score distributions: if BM25 hands back a score of 1000.0 for a single outlier document and 0.5 for everyone else, min-max collapses every other document to ~0 and the outlier dominates blindly. RRF reads only the rank, so the outlier's "rank 0 in FTS" still has to compete with "rank 0 in vector" for the top slot.

When to choose RRF:

- The two modes have wildly different score scales (e.g. dense vector cosine in `[0, 1]` vs BM25 in `[0, 30+]`).
- A few outlier scores skew min-max normalisation.
- You want a strategy that is invariant to score-space changes (e.g. swapping the embedding model).

When to stay with min-max:

- The α / β knobs need to map onto raw signal weight — RRF discards the score magnitudes.
- Score distributions are well-behaved (typical for recent BM25 + cosine on the same corpus).

```sql
-- RRF blend.
SELECT d.id, d.title, h.score
FROM HYBRID_SEARCH(table     => 'public.docs',
                   ts_query  => 'AI R&D',
                   ts_index  => 'idx_docs_text',
                   vec_query => :emb_vector,
                   vec_index => 'idx_docs_emb',
                   blend     => 'rrf',
                   k         => 50) h
JOIN docs d ON d.id = h.id
ORDER BY h.score DESC
LIMIT 10;
```

## Composing with Hard Filter, Graph Re-rank, and Fact-Check

The whole point of putting hybrid search in SQL is that the surrounding query plans normally. Three patterns cover most agent-side workloads.

### 1. Hard Filter + Hybrid

Pre-compute the candidate set with `HYBRID_SEARCH`, then apply structured constraints in `WHERE` against the metadata table that owns the rest of the row. NeorunBase pushes the predicate down through the JOIN so per-shard scans evaluate the filter against the candidate ids only:

```sql
SELECT d.id, d.title, h.score
FROM HYBRID_SEARCH(table => 'public.docs', ts_query => 'AI R&D',
                   ts_index => 'idx_docs_text',
                   vec_query => :emb, vec_index => 'idx_docs_emb',
                   alpha => 0.4, beta => 0.6, k => 100) h
JOIN docs d ON d.id = h.id
WHERE d.region = '경기도' AND d.revenue >= 100000000000
ORDER BY h.score DESC
LIMIT 10;
```

### 2. Graph-aware re-ranking — the SMBA pattern

JOIN the hybrid candidates with `PERSONALIZED_PAGERANK` (see [Graph Traversal & Analytics](graph.md)), seeded on the user's query target. Final score blends both:

```sql
SELECT d.id, d.title,
       0.6 * h.score + 0.4 * p.rank AS final_score
FROM HYBRID_SEARCH(table => 'public.docs', ts_query => 'AI', ts_index => 'idx_docs_text',
                   vec_query => :emb, vec_index => 'idx_docs_emb',
                   alpha => 0.5, beta => 0.5, k => 50) h
JOIN PERSONALIZED_PAGERANK(table => 'public.doc_relations',
                            seed  => :user_target_id,
                            k     => 200) p ON p.node_id = h.id
JOIN docs d ON d.id = h.id
ORDER BY final_score DESC
LIMIT 10;
```

### 3. RIG fact-check at query time

`GRAPH_PATH_EXISTS` returns a single-row BOOLEAN, so you can fold it directly into `WHERE` to drop documents whose retrieval the agent's claim cannot justify:

```sql
SELECT d.id, d.title, h.score
FROM HYBRID_SEARCH(table => 'public.programs', ts_query => :q,
                   ts_index => 'idx_programs_text',
                   vec_query => :emb, vec_index => 'idx_programs_emb', k => 50) h
JOIN programs d ON d.id = h.id
WHERE (SELECT path_exists FROM GRAPH_PATH_EXISTS(
         edge_table => 'public.eligibility',
         src        => :company_id,
         dst        => d.id,
         max_depth  => 2,
         edge_filter => 'eligible_for') v)
ORDER BY h.score DESC
LIMIT 10;
```

The agent submits a candidate set whose every member has a graph-verified eligibility path to the user's company — no hallucinated programs survive the WHERE.

## How It Runs

1. The coordinator's TVF rewriter detects `HYBRID_SEARCH(...)` *before* SQL parsing, parses the named arguments, and resolves the table's schema (`pkTypeId`, shard map, owning data nodes).
2. The `HybridSearchExecutor` issues two parallel scatter-gather searches — one over the FTS index, one over the HNSW index — at oversampled depth (`per_mode_k`). Both reuse the standard `FtsSearchExecutor` / `VectorSearchExecutor` paths, so encryption / IAM / shard pruning all apply unchanged. The Hybrid path propagates one per-stage timeout, `neorunbase.search.scatter.stage.timeout.ms` (default 30000), down to both inner executors so all three backends share a single wall-clock budget.
3. The executor blends the two top-K lists into a single sorted hybrid top-K according to the chosen strategy.
4. The blended result is inlined back into the original SQL as a derived table (`(SELECT 1 AS id, 0.92 AS score UNION ALL …) AS h`). The rest of the query — JOINs, WHERE, ORDER BY, LIMIT — plans through NeorunBase's normal pipeline.

Because the blend happens once on the coordinator and the result is materialised before the outer query runs, there is no per-row scoring overhead downstream and the planner can prune shards based on the candidate id set just like any IN-list predicate.

## Bounds and Roadmap

- **Single FTS + single ANN index per call.** Multi-index hybrid (e.g. one ANN per modality) currently requires UNION-ing multiple `HYBRID_SEARCH` calls.
- **No prepared-statement vector binding.** The vector literal is a string at rewrite time. Direct `?`-binding for the vector is on the roadmap.
- **Per-mode oversample is k×4 by default.** Underlying recall tradeoff — set `per_mode_k` larger for queries where one signal dominates.
- **No global IDF for FTS.** BM25 scoring is shard-local (matches OpenSearch's `query_then_fetch`). A global-DF mode is planned for cases where strict cross-shard score comparability matters.
- **Score normalisation is min-max.** RRF is available as an opt-in for outlier-prone distributions; learned-fusion modes (e.g. logistic regression over per-mode signals) are not in scope.
