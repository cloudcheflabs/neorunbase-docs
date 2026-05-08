# Full-Text Search

NeorunBase ships a built-in BM25 full-text search engine alongside its vector and relational paths. The surface is PostgreSQL-compatible — `tsvector`, `tsquery`, the `@@` match operator, and the `ts_rank` family — so existing client code works unchanged. Underneath, every shard owns its own Lucene-based inverted index, persisted as an encrypted sidecar in the same way the HNSW vector index is.

## Why It's Built In

Most stacks bolt search onto a separate Elasticsearch / OpenSearch cluster, which forces a synchronisation pipeline (Debezium, Kafka, custom batch jobs) and gives up cross-system ACID. NeorunBase does it inside the same database so:

- **One transaction** writes a document and its searchable form atomically.
- **One SQL statement** can `JOIN` searchable text with relational metadata, filter by `WHERE`, and rank by BM25 — no glue layer.
- **One operational surface** — the same backup, replication, IAM, and encryption story already in place for OLTP applies to search.
- **Iceberg auto-sync** carries searchable text into the lakehouse with no extra work.

## TSVECTOR / TSQUERY Types

Two PG-native types are added with their canonical OIDs (`3614` / `3615`) so JDBC, psycopg, and ORM clients recognise them without driver changes:

```sql
-- tsvector / tsquery flow as opaque text on the wire; Lucene tokenises
-- on the data node when CREATE INDEX is built and when search runs.
SELECT to_tsvector('english', 'machine learning fundamentals');
SELECT plainto_tsquery('korean', '한국어 검색');
```

The function set tracks PostgreSQL: `to_tsvector`, `plainto_tsquery`, `phraseto_tsquery`, `websearch_to_tsquery`, `ts_rank`, `ts_rank_cd`. The match operator `a @@ b` is recognised before the SQL reaches the planner and lowered to `tsmatch(a, b)` — no driver-side transformation needed.

## CREATE INDEX … USING fts

A single DDL line creates the per-shard inverted index. Configuration sits in the standard `WITH` clause:

```sql
CREATE TABLE articles (
    id   BIGINT PRIMARY KEY,
    body TEXT
) SHARD KEY (id) SHARDS 16;

-- English: the default Lucene Standard analyzer.
CREATE INDEX idx_body
  ON articles
  USING fts (body);

-- Korean: Lucene's official Nori morphological analyzer.
CREATE INDEX idx_body_ko
  ON articles
  USING fts (body)
  WITH (lang = 'korean');
```

Supported `lang` values include `english` (default), `simple`, and `korean`. Adding a new language is one analyzer wiring — the rest of the path is language-agnostic.

### Primary Key Requirement

Like HNSW, FTS addresses documents by a numeric primary key (`INT`, `BIGINT`, or `SMALLINT`). This keeps a search hit round-tripping straight back to the row store via PK lookup — no secondary `id → pk` map.

## Querying — `@@` and `ts_rank`

```sql
-- Simple match — boolean filter only.
SELECT id, body
FROM articles
WHERE body @@ 'machine learning'
LIMIT 10;

-- Ranked retrieval — the planner recognises this shape and routes it
-- through the per-shard FTS scatter-gather, returning the global top-K
-- by BM25 score.
SELECT id, body, ts_rank(body, 'machine learning') AS score
FROM articles
WHERE body @@ 'machine learning'
ORDER BY score DESC
LIMIT 10;
```

Lucene query syntax is accepted in the right operand: plain words, `AND` / `OR` / `NOT`, `"quoted phrases"`, `field:term`, and `+required` / `-excluded` prefixes.

## Distributed BM25 Across Shards

A FTS-indexed table has **one Lucene index per shard**. On a top-K query, the Coordinator scatters an `FTS_SEARCH_REQ` to each shard owner; every Data Node runs its local Lucene search, applies `WHERE` pre-filter pushdown if present, and returns its local top results; the Coordinator merges them into the global top-K by descending BM25.

NeorunBase uses **shard-local BM25** scoring by default — each shard scores against its own document frequency. This matches OpenSearch's default `query_then_fetch` mode and trades a small recall hit on heavily-skewed data distributions for one fewer network round-trip. A future `dfs_query_then_fetch` mode (coordinator collects global df first, then re-scores) is on the roadmap; the wire protocol leaves room for it without breaking changes.

## Hybrid Retrieval — BM25 + Vector in One SQL

Because vector search and full-text search are both first-class operations on the same table, hybrid retrieval is just an arithmetic blend in the SELECT:

```sql
SELECT
    d.id, d.title, d.body,
    ts_rank(d.body, plainto_tsquery('한국어 검색'))    AS text_score,
    d.embedding <=> :query_embedding                  AS vec_distance,
    -- α=0.4 BM25 + β=0.6 vector similarity (1 / (1 + distance))
    ts_rank(d.body, plainto_tsquery('한국어 검색')) * 0.4
      + (1 / (1 + (d.embedding <=> :query_embedding))) * 0.6 AS rank
FROM documents d
JOIN tenants t ON d.tenant_id = t.id
WHERE t.id = :tenant_id
  AND d.body @@ plainto_tsquery('한국어 검색')
ORDER BY rank DESC
LIMIT 10;
```

This single statement enforces tenant isolation (`JOIN` + `WHERE`), restricts via BM25 (`@@`), and ranks by a hybrid of BM25 score and vector similarity — all in one ACID transaction over the same indexes.

## Storage Layout — Encrypted Sidecar

A Lucene index can't live efficiently inside RocksDB (per-segment file format, frequent random reads). NeorunBase stores each shard's Lucene directory as an **encrypted sidecar** in the shard's `sidecars/` directory — the same pattern used by HNSW.

- Chunked envelope encryption with per-chunk IV + GCM tag.
- KMS-wrapped Data Encryption Key per sidecar, isolated from the shard's primary DEK.
- On shard open, the sidecar is decrypted into an in-memory Lucene `ByteBuffersDirectory`.
- Atomic replacement on rebuild follows the standard `*.tmp → fsync → rename` flow.

The same mechanics that protect at-rest row data therefore protect the inverted index.

## Freshness, Flush, and Backup Safety

Rewriting the sidecar on every row mutation would be ruinous, so NeorunBase keeps the authoritative Lucene state in memory at runtime and flushes back to disk on a schedule.

- **Incremental writes** — every INSERT updates the in-memory index as part of regular DML.
- **Periodic flush** — `FtsFlushScheduler` writes dirty sidecars back at a configurable interval (`neorunbase.fts.flush.interval.seconds`, default 30 s).
- **Flush on shard close** — the close path persists every cached entry so a graceful shutdown never loses recent writes.
- **Pre-backup flush hook** — every `BACKUP_RUN_REQ` triggers an explicit `flushTick()` on the Data Node before the shard tree is uploaded, so the backup captures the live in-memory state, not yesterday's snapshot.

The result: a backup taken seconds after an INSERT contains that INSERT.

## Observability

The same Codahale / Prometheus pipeline used elsewhere in NeorunBase. The admin endpoint `GET /admin/fts/status` (mirrors `/admin/ann/status`) returns one row per cached `(shard, index)` with size, dirty flag, pending WAL seqno, and last flushed seqno.

## Current Limitations

- **MVP is INSERT-only** — UPDATE / DELETE on FTS-indexed tables update the row store but don't tombstone in Lucene yet. Append-only workloads (logs, articles, knowledge bases) are unaffected; row-mutation-heavy schemas should wait for the tombstone follow-up.
- **Shard-local BM25** — see the "Distributed BM25" section. Adequate for evenly-distributed data; less precise on heavily skewed shards. `dfs_query_then_fetch` mode is on the roadmap.
- **No phrase positions in scoring** — `"quoted phrase"` matching works (positional postings are written) but proximity-aware scoring is left to a follow-up.

## Compatibility Notes

- **PostgreSQL FTS shape** — `tsvector` / `tsquery` types, `@@` operator, `ts_rank` family, `to_tsvector(config, text)` with a per-language `config` argument. Standard pg JDBC / psycopg / ORM clients see them as native.
- **Lucene 9.11** — analyzer + index format. The Nori artifact (`lucene-analysis-nori`) bundles `mecab-ko-dic`, so Korean tokenisation works out of the box without external dictionary install.
- **CREATE INDEX … USING fts** — internal name. Existing PostgreSQL DDL with `USING gin (col tsvector_ops)` aliases to the same FTS index path on the roadmap.
