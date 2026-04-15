# Vector Database Capability

NeorunBase is a general-purpose, scalable vector database. It stores high-dimensional embeddings alongside regular relational data and serves approximate nearest neighbor (ANN) search through the same PostgreSQL wire protocol as the rest of the database. The surface area is pgvector-compatible, so existing applications and libraries built for pgvector connect without modification.

## VECTOR Data Type

NeorunBase adds a `VECTOR(dim)` data type to the standard PostgreSQL type system. Vectors are fixed-dimension arrays of 32-bit floats and use the pgvector literal form `'[v1, v2, ..., vN]'`.

```sql
CREATE TABLE documents (
    id        BIGINT PRIMARY KEY,
    content   TEXT,
    embedding VECTOR(1536)
) SHARD KEY (id) SHARDS 8;

INSERT INTO documents (id, content, embedding)
VALUES (1, 'hello world', '[0.12, -0.04, 0.91, ...]');
```

VECTOR column values are stored in RocksDB like any other column type and inherit the per-shard envelope encryption automatically, so embeddings are protected at rest on disk.

## Distance Operators and Functions

NeorunBase implements the pgvector operator and function set so that SELECT expressions, ORDER BY clauses, and computed columns work with either form.

| Operator | Function          | Distance       |
|----------|-------------------|----------------|
| `<->`    | `l2_distance`     | Euclidean (L2) |
| `<=>`    | `cosine_distance` | Cosine         |
| `<#>`    | `inner_product`   | Negative inner product |

```sql
-- Top-K nearest neighbors using cosine distance.
SELECT id, content, embedding <=> '[...]' AS distance
FROM documents
ORDER BY embedding <=> '[...]'
LIMIT 10;
```

Helper functions `VECTOR_DIMS(v)` and `VECTOR_NORM(v)` are also available.

## HNSW Index for Approximate Nearest Neighbor Search

Without an index, NeorunBase can evaluate distance operators with a full scan, which is sufficient for small tables but not for production-scale workloads. For low-latency top-K search at scale, NeorunBase builds an **HNSW (Hierarchical Navigable Small World)** index on a VECTOR column.

Index creation uses the pgvector DDL syntax, with an opclass that selects the distance metric and a `WITH` clause for tuning parameters.

```sql
CREATE INDEX idx_doc_embedding
  ON documents
  USING hnsw (embedding vector_cosine_ops)
  WITH (m = 16, ef_construction = 64);
```

### Supported Opclasses

| Opclass              | Distance metric |
|----------------------|-----------------|
| `vector_l2_ops`      | L2 distance     |
| `vector_cosine_ops`  | Cosine distance |
| `vector_ip_ops`      | Inner product   |

### Tunables

- **`m`** and **`ef_construction`** — graph build parameters, set at index creation time.
- **`ef_search`** — query-time parameter controlling recall/latency trade-off.

### Query-Time Planner Integration

When a query has the form `ORDER BY vector_column <op> constant LIMIT K` and a matching HNSW index exists, the planner rewrites it into a top-K ANN search that pushes the per-shard work down to each Data Node. Unrelated `WHERE` predicates in the same query are pushed to the Data Node as well and evaluated against ANN candidates before the candidates are shipped back to the Coordinator, which reduces wire traffic for selective filters.

## Distributed ANN Across Shards

Because vector search runs inside the same sharded storage layer as the rest of the database, it scales the same way:

- A VECTOR column on a sharded table has **one HNSW graph per shard**.
- On a top-K query, the Coordinator scatters a `VECTOR_SEARCH_REQ` to every shard in parallel.
- Each Data Node runs a local HNSW traversal, applies pushdown filters, and returns its local top results.
- The Coordinator merges the per-shard results into the global top K and returns them to the client.

An over-fetch factor is applied per shard to compensate for the recall loss that comes from shard-local ANN.

## Storage Layout — Encrypted Sidecar

An HNSW index cannot live inside RocksDB efficiently: graph traversal performs hundreds of random hops per query, and a RocksDB `get` per hop would destroy latency. NeorunBase stores each HNSW graph as a **sidecar file** in the shard directory — outside RocksDB but inside the shard's encryption boundary.

- The sidecar uses a chunked envelope-encrypted file format. Each chunk carries its own IV and GCM authentication tag.
- Each sidecar has its own KMS-wrapped Data Encryption Key stored in the file header, isolated from the shard's primary DEK.
- On shard open, chunks are decrypted into an in-memory plaintext buffer; traversal runs on that buffer.
- Atomic replacement on rebuild follows the same `*.tmp → fsync → rename` pattern as the WAL.
- The sidecar is **derived state** — if it is missing or stale, it is rebuilt from authoritative RocksDB rows.

## Freshness, Flush, and Recovery

Rewriting the sidecar on every row change would be ruinously slow, so NeorunBase keeps the authoritative ANN graph in memory during uptime and flushes back to the encrypted sidecar on a schedule.

- **Incremental writes** — inserts, updates, and deletes update the in-memory graph as part of regular DML. No extra client-visible step.
- **Periodic flush** — a background scheduler writes dirty sidecars back to disk at a configurable interval (`neorunbase.ann.flush.interval.seconds`, default 30s).
- **Flush on shard close** — the shard close path flushes all of its cached ANN indexes so a graceful shutdown never loses in-memory state.
- **Lazy rebuild on stale sidecar** — each sidecar records the WAL sequence number it was flushed at. On the next write, the Data Node compares that sequence number to the current WAL seqno and rebuilds the sidecar from RocksDB rows if it is behind (or missing entirely, e.g. on a freshly repaired replica). No admin action is needed to converge after an ungraceful shutdown as long as the shard eventually sees a write.
- **On-demand rebuild** — `REBUILD INDEX idx_name ON table_name` (SQL) or the admin REST endpoint `POST /admin/ann/rebuild` forces an unconditional rebuild on every replica of every shard owning the table. This covers read-only or lightly-written shards whose lazy path may not fire.

## Primary Key Requirement for HNSW-Indexed Tables

HNSW nodes are addressed by a numeric id. To keep search hits round-tripping directly through the primary key store without a secondary id→pk map, NeorunBase requires HNSW-indexed tables to have a single-column numeric primary key (`INT`, `BIGINT`, or `SMALLINT`). Attempts to create an HNSW index on a table with a non-numeric or composite primary key are rejected at DDL time.

## Observability

Vector search is instrumented alongside the rest of NeorunBase in the built-in Codahale/Prometheus metrics pipeline. Exposed metrics include search / insert / update / delete / flush / rebuild meters, an error counter, and latency timers for search, flush, and rebuild (p50 / p99 / max). The admin endpoint `GET /admin/ann/status` fans out across all Data Nodes and returns one row per cached `(shard, index)` pair with size, dirty flag, pending WAL seqno, and last flushed seqno — enough to tell at a glance which sidecars are behind.

## Compatibility Notes

- **pgvector parity**: the type literal, operators, functions, opclasses, and `CREATE INDEX ... USING hnsw (col opclass) WITH (...)` syntax match pgvector, so JDBC / psycopg / pgvector client libraries work without changes.
- **VECTOR type OID**: NeorunBase uses a stable custom OID (`100001`) outside the reserved PostgreSQL range. Clients that treat unknown types as opaque text round-trip VECTOR values correctly via the `[v1, v2, ...]` literal form.
