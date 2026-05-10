# Architecture

NeorunBase is a distributed, sharded OLTP Lakebase that implements the PostgreSQL wire protocol. Any PostgreSQL-compatible client such as `psql`, JDBC drivers, or pgAdmin can connect to NeorunBase transparently without any modification.

NeorunBase provides horizontal scalability through hash-based sharding across multiple Data Nodes, fault tolerance via shard replication, and encryption at rest using envelope encryption with a built-in KMS. It also serves as:

- a **general-purpose vector database** with a pgvector-compatible `VECTOR` type and distributed HNSW approximate-nearest-neighbor search;
- a **distributed BM25 full-text search engine** with Lucene-backed per-shard inverted indexes;
- a **graph database** with a Compressed-Sparse-Row adjacency layer, PageRank / Personalized-PageRank, multi-hop reachability primitives, and single-SQL hybrid retrieval that blends BM25, vector, and graph signals.

It integrates with Apache Iceberg,
enabling automatic synchronization of transactional data to an open lakehouse format for downstream analytics by engines such as Apache Spark, Trino, and Hive. Additionally, NeorunBase supports streaming ingestion from Apache Kafka, allowing real-time data to flow directly into NeorunBase tables.

## NeorunBase Architecture

<img width="1200" src="../../images/architecture/neorunbase-architecture.png" align="center"/>

NeorunBase consists of two main deployable components: **Coordinator** and **Data Node**.

### Coordinator

The Coordinator is the SQL-facing entry point that clients connect to via the PostgreSQL wire protocol. It is responsible for:

- **SQL Parsing**: Parses incoming SQL statements using Apache Calcite (for DML) and a custom DDL parser.
- **Query Routing**: Determines the target shards using Murmur3 hash-based shard routing on the shard key, and prunes shards with Bloom filter caches.
- **Distributed Query Execution**: Scatters queries to the relevant Data Nodes in parallel via an internal binary NIO protocol (with Snappy compression and AES encryption), then merges partial results (sort-merge, aggregation, LIMIT).
- **Distributed Transactions**: Supports ACID transactions across shards using a two-phase commit protocol (PREPARE + COMMIT).
- **Cluster Metadata Management**: The leader Coordinator maintains table schemas and shard maps in an encrypted RocksDB-backed metadata store, and synchronizes them to non-leader Coordinators.
- **Admin API**: Exposes a Netty-based REST API and serves the React-based Admin UI for cluster management, IAM, metrics monitoring, and operational tasks.

### Data Node

The Data Node is the storage layer of NeorunBase. It is responsible for:

- **Shard Storage**: Each shard stores row data in append-only **heap segment files** (`{shardDir}/heap/seg-NNNNN.nrheap`, AES-256-GCM per record with a KMS-wrapped per-segment DEK), built for OLTP read latency at petabyte scale. A per-shard RocksDB `TransactionDB` holds the row → segment-pointer index (16 bytes per row), secondary indexes, and the change log — keeping LSM compaction off the row data path.
- **Query Execution**: Processes DML operations (INSERT, UPDATE, DELETE, SELECT) on local shards, including index scans and predicate pushdown.
- **Vector / ANN**: Each shard hosts per-table HNSW graphs as envelope-encrypted sidecar files outside RocksDB, serving local top-K vector search for the Coordinator to merge.
- **Full-Text Search**: Each shard owns a per-table Lucene inverted index, persisted as encrypted sidecar segments. BM25 search is scattered across shards and merged on the Coordinator.
- **Graph (CSR adjacency)**: For edge tables on the CSR enable list, each shard materialises a Compressed-Sparse-Row view of its local adjacency in encrypted segment files (`{shardDir}/csr/<table>/seg-NNNNN.nrcsr`). Per-`src_id` neighbor lookup is an O(1) pointer hop instead of a B-tree IN-list — the substrate for fast BFS and the BSP-distributed PageRank algorithm.
- **Write-Ahead Log (WAL)**: Maintains encrypted, segmented WAL for durability. Recovery halts on detected corruption (silent skip is treated as a bug, not an option) so the operator can investigate before any divergent state is created.
- **Change Log**: Records data changes for incremental Iceberg synchronization.
- **Bitrot Scrubber** (opt-in): A throttled background scanner periodically verifies the AES-GCM auth tag on every closed heap segment record and surfaces findings via the Admin API.
- **Kafka Consumer Service** (when enabled): Each Data Node hosts its own Kafka consumer worker. The leader Coordinator distributes consumer groups round-robin across Data Nodes; each worker forwards parsed INSERTs to the leader Coordinator over the internal protocol so the Coordinator's normal shard routing still applies.

### Cluster Coordination

NeorunBase uses Apache ZooKeeper (via Curator) for:

- **Service Discovery**: Coordinators and Data Nodes register as ephemeral nodes, enabling automatic detection of node joins and failures.
- **Leader Election**: Two separate leader elections — one for shard assignment (Controller) and one for metadata/KMS ownership (Coordinator Leader).
- **Shard Repair & Rebalancing**: Automatically detects dead Data Nodes and replicates shard replicas to healthy nodes; rebalances shards when nodes are added or removed.

### Graph Engine

NeorunBase folds graph traversal and analytics into the same SQL engine that owns relational, vector, and full-text data:

- **TVFs in `FROM`**: `GRAPH_NEIGHBORS`, `GRAPH_PATH_EXISTS`, `PAGERANK`, `PERSONALIZED_PAGERANK`, and `HYBRID_SEARCH` are exposed as table-valued functions. The Coordinator rewrites each call to a derived table at parse time, so JOIN / WHERE / ORDER BY / LIMIT downstream plan through the standard SQL pipeline.
- **CSR fast path**: Edge tables opted in via `neorunbase.csr.enabled.tables` get a per-shard adjacency view materialised by a leader-gated background compactor. CSR turns a BFS hop from a B-tree IN-list lookup into a sequential neighbor-block read — measured 256× speedup at 100 K edges in the Phase 1 microbenchmark.
- **Distributed PageRank (BSP)**: For graphs that don't fit in Coordinator memory, `PAGERANK(... distributed => true)` switches on a Pregel-lite superstep loop. The Coordinator broadcasts the rank vector (delta-only after the first iter), each Data Node walks its local CSR adjacency and returns sparse contribution maps, and the Coordinator applies teleport + dangling redistribution. Numerically equivalent to the centralised executor within `1e-6`.
- **Hybrid retrieval**: `HYBRID_SEARCH(...)` runs an FTS scatter and an ANN scatter in parallel and blends the two top-K lists either by min-max normalisation + linear combination or by Reciprocal Rank Fusion. JOINing its result with `PERSONALIZED_PAGERANK` and applying a `WHERE` Hard Filter is the SMBA Graph-RAG pipeline expressed in one SELECT.

See [Graph Traversal & Analytics](../features/graph.md) and [Hybrid Search](../features/hybrid-search.md) for the full TVF surface and example SQL.

### Iceberg Integration

NeorunBase synchronizes table data to Apache Iceberg for open lakehouse analytics:

- Connects to an Iceberg REST catalog (e.g., Polaris) with OAuth2 or static token authentication.
- Performs full snapshot sync on initial synchronization, followed by incremental sync via change logs using Iceberg RowDelta (equality deletes + data files).
- Writes Parquet files to S3-compatible object storage.
- Supports reading external Iceberg tables via distributed Parquet/ORC/Avro scan directly from S3.

### Kafka Integration

NeorunBase supports streaming data ingestion from Apache Kafka:

- Manages multiple Kafka consumer groups, each running as an independent consumer thread.
- Consumes JSON messages from Kafka topics and batch-inserts them into NeorunBase tables.

