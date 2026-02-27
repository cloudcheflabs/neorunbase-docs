# Architecture

NeorunBase is a distributed, sharded OLTP Lakebase that implements the PostgreSQL wire protocol. Any PostgreSQL-compatible client such as `psql`, JDBC drivers, or pgAdmin can connect to NeorunBase transparently without any modification.

NeorunBase provides horizontal scalability through hash-based sharding across multiple Data Nodes, fault tolerance via shard replication, and encryption at rest using envelope encryption with a built-in KMS. It also integrates with Apache Iceberg, 
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

- **Shard Storage**: Each shard is backed by a RocksDB `TransactionDB` instance with per-table column families, supporting data-at-rest encryption via envelope encryption (per-shard DEK).
- **Query Execution**: Processes DML operations (INSERT, UPDATE, DELETE, SELECT) on local shards, including index scans and predicate pushdown.
- **Write-Ahead Log (WAL)**: Maintains encrypted, segmented WAL for durability.
- **Change Log**: Records data changes for incremental Iceberg synchronization.

### Cluster Coordination

NeorunBase uses Apache ZooKeeper (via Curator) for:

- **Service Discovery**: Coordinators and Data Nodes register as ephemeral nodes, enabling automatic detection of node joins and failures.
- **Leader Election**: Two separate leader elections — one for shard assignment (Controller) and one for metadata/KMS ownership (Coordinator Leader).
- **Shard Repair & Rebalancing**: Automatically detects dead Data Nodes and replicates shard replicas to healthy nodes; rebalances shards when nodes are added or removed.

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

