# Configuration

NeorunBase is configured through `conf/neorunbase.properties` in the release archive. Every property has a sensible default; you typically only override cluster identity, ZooKeeper connection, advertised hostnames, ports, and whichever feature toggles (Iceberg, Kafka, encryption, scrubber) your deployment uses.

## Configuration Sources & Precedence

NeorunBase resolves each configuration key from three sources, **in order**:

1. **JVM system property** — e.g. `-Dneorunbase.coordinator.pg.port=5433` on the command line.
2. **Environment variable** — e.g. `NEORUNBASE_COORDINATOR_PG_PORT=5433`. The mapping is **uppercase with dots replaced by underscores**.
3. **Properties file** — `conf/neorunbase.properties`, the on-disk default.

A higher source always wins. This lets the start scripts in `bin/` override individual settings on the command line (the example launcher uses this to start two Data Nodes on different ports from a single config file), and lets containerized or systemd-managed deployments inject ports, hostnames, and key material via environment variables without rewriting the file.

Property values can reference other properties with `${...}` substitution. For example:

```properties
neorunbase.base.data.dir=./data
neorunbase.datanode.data.dir=${neorunbase.base.data.dir}/shards
neorunbase.wal.directory=${neorunbase.base.data.dir}/wal
```

!!! note "Container persistence"
    `neorunbase.base.data.dir` is the root of all on-disk state (shards, WAL, KMS, IAM, metadata, logs, metrics). In containers it **must** be a persistent mount, otherwise all cluster state is lost on restart.

## Master Key

Every NeorunBase server process reads a master key from the environment variable named by `neorunbase.kms.master.key.env` (default `NEORUNBASE_MASTER_KEY`). The master key:

- Must be **at least 32 characters** long.
- Must be **identical across all Coordinators and Data Nodes** in the cluster.
- Is the root of the envelope-encryption hierarchy. PBKDF2 with `neorunbase.kms.pbkdf2.iterations` iterations (default `200000`) derives the master encryption key, which wraps the per-key DEKs used for KMS, internal protocol, metadata, WAL, heap segments, and at-rest shard data.
- Is also the secret used to derive the JWT signing key for the admin HTTP server (SHA-256 of the master key); rotating the master key invalidates outstanding admin tokens.

Losing the master key makes all encrypted on-disk state and KMS-wrapped DEKs unrecoverable. Keep it in a secret manager and rotate per your security policy.

## Common Cluster

| Property | Default | Description |
| --- | --- | --- |
| `neorunbase.cluster.id` | `neorunbase-cluster` | Logical name of this cluster. All nodes that form one cluster share the same id; it tags the cluster's ZooKeeper namespace so two clusters can share a ZK ensemble. |
| `neorunbase.zookeeper.server.list` | `localhost:2181` | ZooKeeper ensemble used for leader election, node discovery, and coordination. Comma-separated `host:port` list reachable from every node. |
| `neorunbase.zookeeper.root.path` | `/neorunbase` | Chroot/root znode under which all NeorunBase znodes are created. Must start with `/`. |
| `neorunbase.zookeeper.session.timeout.ms` | `30000` | ZooKeeper session timeout. Too low triggers spurious failovers on GC pauses; too high slows failure detection. |
| `neorunbase.zookeeper.connection.timeout.ms` | `10000` | Maximum time to wait when first establishing a TCP connection to a ZooKeeper server before trying another endpoint. |
| `neorunbase.base.data.dir` | `./data` | Root directory for all on-disk state. Most other path properties derive from this. Must be a persistent mount in containers. |

## Coordinator Node

| Property | Default | Description |
| --- | --- | --- |
| `neorunbase.coordinator.host` | `0.0.0.0` | Bind address for the PG-wire server and the coordinator internal NIO server. `0.0.0.0` binds all interfaces. |
| `neorunbase.coordinator.pg.port` | `5432` | TCP port for PostgreSQL wire protocol clients (psql, JDBC). The primary SQL endpoint. |
| `neorunbase.coordinator.advertised.host` | `localhost` | Hostname/IP this coordinator publishes to ZooKeeper. Must be resolvable/reachable by peers and clients even when the bind host is `0.0.0.0`. |
| `neorunbase.coordinator.worker.threads` | `235` | NIO worker pool backing the coordinator internal server and the query scatter pool. Code default is `8` if unset. |
| `neorunbase.coordinator.internal.port` | `7100` | TCP port for coordinator-to-coordinator RPC (metadata/IAM/KMS broadcast, bootstrap fetch). Distinct from the data-node port (7000) and PG-wire port (5432). |

## Data Node

| Property | Default | Description |
| --- | --- | --- |
| `neorunbase.datanode.host` | `0.0.0.0` | Bind address for the data node's internal NIO server. `0.0.0.0` binds all interfaces. |
| `neorunbase.datanode.internal.port` | `7000` | TCP port for the internal binary protocol (shard reads/writes, scatter, DDL fan-out). Not client-facing. |
| `neorunbase.datanode.advertised.host` | `localhost` | Hostname/IP this data node publishes to ZooKeeper. Must be reachable by coordinators. |
| `neorunbase.datanode.data.dir` | `${neorunbase.base.data.dir}/shards` | Directory (or comma-separated directories) holding this node's RocksDB shard data. Multiple entries spread shards across physical disks. |
| `neorunbase.datanode.worker.threads` | `256` | NIO worker pool servicing inbound shard read/write/scatter work. Code default is `8` if unset. |

## Default Settings

| Property | Default | Description |
| --- | --- | --- |
| `neorunbase.default.num.shards` | `16` | Number of shards a new table is split into when `CREATE TABLE` does not specify one. More shards = more parallelism but more metadata and fan-out. |
| `neorunbase.default.replication.factor` | `2` | Physical copies kept of each shard. `2` = one primary plus one replica; the cluster needs at least this many data nodes to place a shard. |

## Authentication

| Property | Default | Description |
| --- | --- | --- |
| `neorunbase.admin.user` | `admin` | Bootstrap superuser, seeded on first start. Used for PG-wire auth and the admin HTTP UI login. |
| `neorunbase.admin.password` | `admin` | Bootstrap superuser password. **Change before any non-local deployment** — it grants full cluster control. |

## Logging & Monitoring

| Property | Default | Description |
| --- | --- | --- |
| `neorunbase.log.mode` | `ASYNC_FILE` | `ASYNC_FILE` (async append to rolling files under `log.path`; recommended for production), `RING_BUFFER` (in-memory ring buffer only, surfaced in the admin UI; suits ephemeral containers), or `NOP` (console only). |
| `neorunbase.log.path` | `${neorunbase.base.data.dir}/logs` | Directory where rolling log files are written when `log.mode=ASYNC_FILE`. |
| `neorunbase.log.level` | `INFO` | Root log level for NeorunBase loggers. `TRACE < DEBUG < INFO < WARN < ERROR`. |
| `neorunbase.log.ring.buffer.capacity` | `1000` | Number of most-recent log lines retained in the in-memory ring buffer (used by `RING_BUFFER` and the admin UI log tail). |

## KMS (Key Management Service)

| Property | Default | Description |
| --- | --- | --- |
| `neorunbase.kms.enabled` | `true` | Master switch for the built-in KMS, the root of trust for all envelope encryption. Disabling it turns off all encryption features. |
| `neorunbase.kms.rocksdb.path` | `${neorunbase.base.data.dir}/kms` | RocksDB store persisting the KMS key hierarchy (wrapped DEKs) on the leader coordinator. Must be durable and backed up. |
| `neorunbase.kms.master.key.env` | `NEORUNBASE_MASTER_KEY` | Name of the OS environment variable from which the master-key passphrase is read at startup. The same secret must be present on every node. |
| `neorunbase.kms.pbkdf2.iterations` | `200000` | PBKDF2 iteration count for deriving the master key from the passphrase. Must stay consistent so the same passphrase derives the same key. |
| `neorunbase.kms.encrypt.internal.protocol` | `true` | Encrypt the coordinator-to-data-node internal NIO payloads with a KMS key. Only effective when `kms.enabled=true`. |
| `neorunbase.kms.internal.protocol.key.id` | `internal-protocol` | Logical KMS key id used to encrypt internal-protocol messages. |

## Admin HTTP Server

| Property | Default | Description |
| --- | --- | --- |
| `neorunbase.admin.http.port` | `8080` | TCP port for the admin HTTP server (management REST API + static admin UI). Non-leaders proxy leader-only calls to the leader over this port. |
| `neorunbase.admin.ui.static.path` | `./admin-ui` | Filesystem path to the built React admin-ui assets served as static files. |
| `neorunbase.admin.worker.threads` | `256` | Thread pool executing admin HTTP requests. Code default is `4` if unset. |
| `neorunbase.admin.jwt.expiry.ms` | `86400000` | Lifetime of an issued admin UI JWT (24h). Tokens are signed with SHA-256 of the master key, so rotating it invalidates outstanding tokens. |
| `neorunbase.admin.http.max.content.length` | `10485760` | Maximum accepted admin HTTP request body size (10 MiB). |
| `neorunbase.admin.shutdown.timeout.seconds` | `5` | Grace period the admin HTTP server waits for in-flight requests on shutdown before forcing termination. |
| `neorunbase.admin.http.proxy.connect.timeout.ms` | `5000` | Connect timeout for HTTP proxy calls between admin servers (e.g. leader-only endpoints forwarded from non-leaders). |
| `neorunbase.admin.http.proxy.read.timeout.ms` | `30000` | Read timeout for the same proxy calls. Increase for large log-tail responses or slow links. |

## Metrics Collection

| Property | Default | Description |
| --- | --- | --- |
| `neorunbase.metrics.collector.interval.ms` | `15000` | Period between metrics-collection sweeps. The leader polls every node this often. |
| `neorunbase.metrics.collector.initial.delay.ms` | `10000` | Delay after startup before the first metrics sweep, giving nodes time to register. |
| `neorunbase.metrics.retention.days` | `7` | Days of historical metric samples to keep before the retention-cleanup task deletes them. |
| `neorunbase.metrics.rocksdb.path` | `${neorunbase.base.data.dir}/metrics` | RocksDB store holding the metrics time series. |
| `neorunbase.metrics.collector.connect.timeout.ms` | `3000` | TCP connect timeout when the collector opens an HTTP connection to scrape a node. |
| `neorunbase.metrics.collector.read.timeout.ms` | `5000` | Read/response timeout for each metrics-scrape HTTP call. |
| `neorunbase.metrics.retention.cleanup.initial.delay.minutes` | `60` | Delay after startup before the first metrics-retention cleanup run. |
| `neorunbase.metrics.retention.cleanup.interval.minutes` | `360` | Period between metrics-retention cleanup runs (every 6h). |

## Metadata Store (Coordinator)

| Property | Default | Description |
| --- | --- | --- |
| `neorunbase.metadata.rocksdb.path` | `${neorunbase.base.data.dir}/metadata` | Leader coordinator's encrypted metadata store. Holds the authoritative catalog (schemas + shard placement). Must be durable. |
| `neorunbase.metadata.kms.key.id` | `metadata-encryption` | Logical KMS key id used to envelope-encrypt the metadata store at rest. |

## Data At-Rest Encryption

| Property | Default | Description |
| --- | --- | --- |
| `neorunbase.data.encryption.enabled` | `true` | Encrypt RocksDB shard data at rest. Each shard gets its own DEK wrapped by the KMS key below. Requires `kms.enabled=true`. |
| `neorunbase.data.encryption.kms.key.id` | `data-encryption` | Logical KMS key id that wraps the per-shard DEKs for data-at-rest encryption. |

## Protocol Compression

| Property | Default | Description |
| --- | --- | --- |
| `neorunbase.protocol.compression.enabled` | `true` | Snappy-compress internal-protocol payloads before sending over the NIO wire. Usually a win for large scatter/result payloads. |
| `neorunbase.protocol.compression.min.bytes` | `256` | Payloads smaller than this are sent uncompressed even when compression is enabled. Only applies when compression is enabled. |

## Iceberg Catalog Integration

NeorunBase can synchronize tables to an Apache Polaris Iceberg REST catalog. See [Iceberg Integration](../features/iceberg-integration.md) for the full feature description.

!!! note "Catalog backend support"
    `neorunbase.iceberg.catalog.type` accepts only **`none`** (disabled) or **`polaris`**. The generic `neorunbase.iceberg.catalog.rest.*` security/OAuth keys below are **legacy and ignored** under the polaris path; the admin UI hides them. Polaris runtime config is normally set from the admin UI (persisted in the metadata store), not from this file. Other Iceberg-REST flavors are intentionally not supported.

| Property | Default | Description |
| --- | --- | --- |
| `neorunbase.iceberg.catalog.type` | `none` | Catalog backend selector. `none` disables all Iceberg integration; `polaris` enables the Apache Polaris REST-catalog bootstrap. |
| `neorunbase.iceberg.catalog.rest.uri` | (empty) | *(legacy, ignored under polaris)* Generic REST catalog base URI. |
| `neorunbase.iceberg.catalog.rest.security` | `NONE` | *(legacy, ignored under polaris)* REST catalog auth mode: `NONE` or `OAUTH2`. |
| `neorunbase.iceberg.catalog.rest.token-endpoint` | (empty) | *(legacy, ignored under polaris)* OAuth2 token endpoint for client-credentials flow. |
| `neorunbase.iceberg.catalog.rest.client-id` | (empty) | *(legacy, ignored under polaris)* OAuth2 client id. |
| `neorunbase.iceberg.catalog.rest.client-secret` | (empty) | *(legacy, ignored under polaris)* OAuth2 client secret. |
| `neorunbase.iceberg.catalog.rest.token` | (empty) | *(legacy, ignored under polaris)* Static bearer token alternative to OAuth2. |
| `neorunbase.iceberg.catalog.rest.extra.properties` | (empty) | *(legacy, ignored under polaris)* Free-form extra REST-catalog properties (comma-separated `key=val`). |
| `neorunbase.iceberg.catalog.rest.warehouse` | `s3://warehouse` | Iceberg warehouse identifier. For a Polaris catalog this is the catalog name (e.g. `quickstart_catalog`); for an S3-path warehouse it is an `s3://` URI. |
| `neorunbase.iceberg.s3.endpoint` | `http://localhost:9000` | S3 (or S3-compatible) endpoint used by Iceberg FileIO for Parquet/manifests. |
| `neorunbase.iceberg.s3.access.key` | `admin` | S3 access key id for Iceberg object storage (credential). |
| `neorunbase.iceberg.s3.secret.key` | `admin123` | S3 secret access key for Iceberg object storage (credential). |
| `neorunbase.iceberg.s3.region` | `us-east-1` | AWS region string sent with S3 requests. Any value the gateway accepts works for S3-compatible stores. |
| `neorunbase.iceberg.s3.path.style.access` | `true` | Use path-style S3 addressing. Required for MinIO/ShannonStore and most S3-compatible gateways. |
| `neorunbase.iceberg.s3.sse.type` | `none` | Server-side encryption for Parquet files written by the Iceberg sink: `none`, `s3` (SSE-S3, AES-256), or `kms` (SSE-KMS). |
| `neorunbase.iceberg.s3.sse.key` | (empty) | KMS key id/ARN when `sse.type=kms`. |
| `neorunbase.iceberg.sync.interval.ms` | `60000` | Period between CDC-sync passes that push changed rows into Iceberg tables. `0` disables the periodic loop (manual sync still works). |
| `neorunbase.iceberg.sync.thread.pool.size` | `4` | Threads used to sync multiple tables in parallel during one sync pass. |
| `neorunbase.iceberg.sync.auto.start` | `false` | Whether the periodic CDC-sync loop starts automatically at boot. When `false`, sync must be started explicitly (e.g. from the admin UI). |
| `neorunbase.iceberg.sync.table.filters` | (empty) | Comma-separated table filters (`*` wildcard supported). Empty = sync all dirty tables. |
| `neorunbase.iceberg.sync.changelog.batch.size` | `500` | Batch size for reading changed PKs from the changelog during incremental sync. |
| `neorunbase.iceberg.sync.task.timeout.ms` | `300000` | Per-table sync task timeout. |
| `neorunbase.iceberg.default.namespace` | `lakebase` | Default Iceberg namespace, auto-created on catalog init. |
| `neorunbase.iceberg.parquet.compression.codec` | `snappy` | Parquet codec used by NeorunBase's own Iceberg writers (data + delete files): `snappy`, `gzip`, `zstd`, `lz4`, `none`. |
| `neorunbase.iceberg.cdc.partition.unit` | `days` | Hidden-partition transform on the synthetic `_neorun_synced_at` column for CDC-synced tables: `hours`, `days`, or `months`. |
| `neorunbase.iceberg.parquet.read.prefetch.threshold.bytes` | `67108864` | Iceberg-data Parquet read pre-fetch threshold (64 MiB). Files at or below this size are pulled in a single GET to avoid per-chunk seeks. |
| `neorunbase.iceberg.load.batch.size` | `1000` | Bulk-load batch size used when caching an external Iceberg table into LakeBase. Larger batches amortize per-INSERT overhead at the cost of memory. |

## S3 Backup

Scheduled cluster backup to S3-compatible storage. See [Backup & Restore](../features/backup-restore.md) for the full feature description.

!!! note "Runtime values live in the admin UI"
    Most operators set the runtime backup values (enabled, endpoint, bucket, keys, retention) from the admin UI; those are persisted (encrypted) in the metadata store and override anything in this file. The properties below only control the *internals* of the backup loop.

| Property | Default | Description |
| --- | --- | --- |
| `neorunbase.backup.tick.interval.seconds` | `60` | How often the backup ticker wakes up to evaluate interval, leadership, and enabled state. |
| `neorunbase.backup.history.cap` | `200` | Cap on the in-memory recent-history list returned by the admin UI. The S3-side history index is independent and trimmed only by retention days. |
| `neorunbase.backup.rpc.timeout.ms` | `1800000` | Per-node fan-out timeout for `BACKUP_RUN_REQ` / `BACKUP_RESTORE_REQ` (30 min). |

## Kafka Consumer

NeorunBase can ingest JSON messages directly from Kafka into target tables. See [Kafka Integration](../features/kafka-integration.md).

| Property | Default | Description |
| --- | --- | --- |
| `neorunbase.kafka.consumer.enabled` | `false` | Master switch for the built-in Kafka ingestion consumers. |
| `neorunbase.kafka.consumer.groups` | (empty) | Comma-separated list of consumer-group names to start. Each expands to a per-group block (below). Empty = no consumers even if enabled. |
| `neorunbase.kafka.consumer.leader.rpc.timeout.ms` | `30000` | RPC timeout for the data-node consumer when it forwards a batch INSERT/DDL to the leader coordinator. Sits above the default selector latency budget because one batch can scatter to many shards. |

For each group `<name>` listed in `neorunbase.kafka.consumer.groups`, configure a per-group block. The defaults shown are what the loader falls back to if a key is omitted:

```properties
neorunbase.kafka.consumer.group.<name>.brokers=localhost:9092
neorunbase.kafka.consumer.group.<name>.topics=topic1,topic2
neorunbase.kafka.consumer.group.<name>.group-id=neorunbase-<name>
neorunbase.kafka.consumer.group.<name>.target-table=my_table
neorunbase.kafka.consumer.group.<name>.id-columns=
neorunbase.kafka.consumer.group.<name>.num-shards=2
neorunbase.kafka.consumer.group.<name>.format=json
neorunbase.kafka.consumer.group.<name>.batch-size=100
neorunbase.kafka.consumer.group.<name>.poll-timeout-ms=1000
neorunbase.kafka.consumer.group.<name>.auto-offset-reset=latest
neorunbase.kafka.consumer.group.<name>.extra.properties=max.poll.records=500
neorunbase.kafka.consumer.group.<name>.max-consumers=3
```

| Per-group key | Default | Description |
| --- | --- | --- |
| `.brokers` | `localhost:9092` | Kafka bootstrap servers. |
| `.topics` | (empty) | Comma-separated topics to consume. |
| `.group-id` | `neorunbase-<name>` | Kafka consumer `group.id`. |
| `.target-table` | (empty) | NeorunBase table to ingest into. |
| `.id-columns` | (empty) | Primary-key / upsert columns. |
| `.num-shards` | `2` | Shards if the target table is auto-created. |
| `.format` | `json` | Record format. |
| `.batch-size` | `100` | Records buffered per INSERT batch. |
| `.poll-timeout-ms` | `1000` | `KafkaConsumer.poll` timeout. |
| `.auto-offset-reset` | `latest` | `earliest` or `latest`. |
| `.extra.properties` | (empty) | Raw `KafkaConsumer` props as `k=v,k=v`. |
| `.max-consumers` | `3` | Max parallel consumer threads. |

The leader Coordinator distributes consumer groups round-robin across the Data Nodes; each Data Node hosts the consumer worker for the groups assigned to it.

## IAM (Identity & Access Management)

| Property | Default | Description |
| --- | --- | --- |
| `neorunbase.iam.rocksdb.path` | `${neorunbase.base.data.dir}/iam` | RocksDB store for IAM state (users, roles, grants, policies, STS temp credentials). Owned by the leader, replicated to peers. Must be durable. |
| `neorunbase.iam.sts.cleanup.interval.hours` | `1` | How often to sweep expired STS temporary credentials from the AuthManager keystore. |

## Write-Ahead Log (WAL) & Encryption

| Property | Default | Description |
| --- | --- | --- |
| `neorunbase.wal.encryption.enabled` | `true` | Envelope-encrypt WAL segments (per-segment DEK wrapped by the KMS key below). Requires `kms.enabled=true`. |
| `neorunbase.wal.encryption.kms.key.id` | `wal-encryption` | Logical KMS key id that wraps the per-segment WAL DEKs. |
| `neorunbase.wal.directory` | `${neorunbase.base.data.dir}/wal` | Base WAL directory; per-shard subdirectories are created automatically. Must be durable. |
| `neorunbase.wal.segment.size` | `67108864` | Maximum size a WAL segment grows to before rotation (64 MiB). |
| `neorunbase.wal.sync.interval.ms` | `1000` | How often buffered WAL writes are fsync'd to disk. Smaller = stronger durability, more IO. |

## Row-Data Storage Mode

| Property | Default | Description |
| --- | --- | --- |
| `neorunbase.storage.row.data.mode` | `heap` | `heap` (default — values appended to `{shardDir}/heap/seg-*.nrheap`, RocksDB holds only a 16-byte pointer per row) or `rocksdb` (legacy rollback escape hatch; values live inside RocksDB). Indexes, changelog, and system column families always stay in RocksDB. |
| `neorunbase.heap.segment.size` | `67108864` | Maximum size a heap row-data segment grows to before rotation (64 MiB). Only relevant when `storage.row.data.mode=heap`. |
| `neorunbase.heap.encryption.kms.key.id` | `heap-encryption` | Logical KMS key id that wraps the per-segment DEKs for heap row-data files. |

## Bitrot Scrubber

| Property | Default | Description |
| --- | --- | --- |
| `neorunbase.scrubber.enabled` | `false` | Enable the periodic background scrubber that walks closed heap segments and verifies every record's AES-GCM auth tag. When disabled, bitrot is still detected on read but no proactive scan runs. |
| `neorunbase.scrubber.interval.minutes` | `360` | Minutes between automatic scrub passes (6h). |
| `neorunbase.scrubber.throttle.bytes.per.sec` | `52428800` | Maximum disk throughput the scrubber may consume (50 MB/s). The scan paces itself to stay under this rate. |

## Internal Protocol Tuning

| Property | Default | Description |
| --- | --- | --- |
| `neorunbase.nio.read.buffer.size` | `65536` | Per-connection read buffer allocated by the internal NIO servers (64 KiB). Larger buffers reduce read syscalls at the cost of memory per channel. |
| `neorunbase.nio.selector.timeout.ms` | `1000` | `Selector.select()` poll timeout in the internal NIO event loops. Lower = snappier wakeups, slightly more CPU spin. |
| `neorunbase.internal.rpc.timeout.ms` | `120000` | Internal RPC request timeout (2 min). Iceberg scan / DDL-forwarding paths floor this at `300000` (5 min) regardless of this value. |
| `neorunbase.datanode.discovery.poll.interval.ms` | `2000` | During coordinator startup, how often to re-poll ZooKeeper for registered data nodes while waiting for at least one. |
| `neorunbase.datanode.discovery.max.retries` | `30` | Maximum data-node discovery poll attempts at startup before proceeding without them. Total wait ≈ retries × poll interval. |
| `neorunbase.leader.init.poll.interval.ms` | `1000` | On a non-leader coordinator, how often to poll for the leader to finish initialization before completing its own startup. |
| `neorunbase.leader.init.max.retries` | `15` | Maximum non-leader initialization poll attempts before giving up the wait for the leader. |

## Data Node Periodic Tasks

| Property | Default | Description |
| --- | --- | --- |
| `neorunbase.datanode.disk.info.refresh.interval.seconds` | `60` | How often the data node refreshes its disk capacity/availability info in ZooKeeper so coordinators see current free-space numbers. |
| `neorunbase.datanode.temp.table.cleanup.interval.seconds` | `60` | How often the data node sweeps for expired temporary tables (intermediate query results). |
| `neorunbase.datanode.temp.table.ttl.ms` | `600000` | Time-to-live for temporary tables before they are eligible for cleanup (10 min). |

## PG Wire Protocol Server Tuning

| Property | Default | Description |
| --- | --- | --- |
| `neorunbase.pgwire.read.buffer.initial.bytes` | `8192` | Initial per-connection read buffer, grown on demand up to the cap below. |
| `neorunbase.pgwire.read.buffer.max.bytes` | `67108864` | Absolute upper bound on a single inbound pg-wire message (64 MiB). A client advertising a larger frame receives ErrorResponse + connection close. |
| `neorunbase.pgwire.worker.pool.size` | `0` | pg-wire worker pool that executes queries off the selector. `0` = auto: `max(2, 2 × CPU cores)`. |
| `neorunbase.pgwire.worker.shutdown.timeout.seconds` | `5` | Grace period for the worker pool to drain on shutdown before forced termination. |

## Vector ANN & FTS Index Maintenance

| Property | Default | Description |
| --- | --- | --- |
| `neorunbase.ann.flush.interval.seconds` | `30` | Interval between background flushes of dirty in-memory ANN indexes to their encrypted sidecar files. `0` disables periodic flush (sidecars written only on shard close). |
| `neorunbase.fts.flush.interval.seconds` | `30` | Same flush trade-off for the BM25 inverted-index sidecars. `0` disables periodic flush (written only on shard close + backup pre-flush hook). |
| `neorunbase.fts.storage.default` | `encrypted_disk` | FTS storage backend for new `CREATE INDEX … USING fts`: `encrypted_disk` (KMS-enveloped Lucene segments on disk, PB-class scale) or `memory` (entire index in heap, flushed to one encrypted byte[] sidecar; bounded by RAM). |
| `neorunbase.fts.plaintext.cache.bytes` | `268435456` | Per-FTS-index plaintext (decrypted segment) LRU cache cap when `storage=encrypted_disk` (256 MiB). Cold segments are re-decrypted on demand. |
| `neorunbase.fts.chunk.size.bytes` | `1048576` | AES-GCM chunk size inside each encrypted FTS segment file (1 MiB). Larger chunks reduce GCM tag overhead but increase the per-decrypt memory window. |

## Graph Traversal

| Property | Default | Description |
| --- | --- | --- |
| `neorunbase.graph.default.max.results` | `1000` | Default cap on the visited set when `GRAPH_NEIGHBORS` / `POST /admin/graph/neighbors` omit `max_results`. Per-call values always win. |

## CSR Adjacency (Graph Acceleration)

CSR (Compressed Sparse Row) materializes a per-shard adjacency-list view of selected edge tables so that BFS expansion becomes a sequential neighbor-block read instead of a B-tree IN-list lookup. Edge rows remain primary; CSR is a derived view rebuilt by background compaction (write-back / LSM style). Disabled by default — turn on per table once the edge schema (`src_id BIGINT`, `dst_id BIGINT`, `rel_type TEXT`) is in place. See [Graph Traversal & Analytics](../features/graph.md) for the full surface.

| Property | Default | Description |
| --- | --- | --- |
| `neorunbase.csr.enabled` | `false` | Master switch. When `false` the original row-scan traversal path is unchanged. |
| `neorunbase.csr.enabled.tables` | (empty) | Comma-separated list of qualified edge tables to materialize (e.g. `public.relations,public.edges`). |
| `neorunbase.csr.compaction.interval.seconds` | `3600` | Per-shard background interval at which the CSR compactor scans for new edge rows and rebuilds affected adjacency blocks. |
| `neorunbase.csr.max.delta.rows.before.compaction` | `100000` | Trigger an out-of-band compaction when delta rows since the last CSR build cross this threshold. Bounds CSR staleness under heavy ingest. |
| `neorunbase.csr.segment.size` | `67108864` | CSR segment file rollover (64 MiB). Mirrors `neorunbase.heap.segment.size`. |
| `neorunbase.csr.neighbor.timeout.ms` | `30000` | Per-node RPC timeout for one BFS-hop neighbor expansion through CSR. |

## Distributed Search Scatter (FTS / Vector / Hybrid)

| Property | Default | Description |
| --- | --- | --- |
| `neorunbase.search.scatter.stage.timeout.ms` | `30000` | Per-stage RPC timeout for FTS, Vector ANN, and Hybrid scatter calls. One query fans out to every owning data node in parallel; this is the wall-clock budget per node. The Hybrid path shares this value with its inner FTS + Vector executors. |

## Distributed PageRank (BSP / Pregel-lite)

| Property | Default | Description |
| --- | --- | --- |
| `neorunbase.pagerank.distributed.stage.timeout.ms` | `60000` | Per-superstep RPC timeout for `PAGERANK_LOCAL_STEP_REQ`. Caps how long any one node can take to return its sparse contribution map. Tighten on small graphs. |
| `neorunbase.pagerank.distributed.delta.broadcast.ratio` | `0.001` | Delta-broadcast threshold ratio. After the first superstep the coordinator only re-sends rank entries that moved by more than `max(convergenceThreshold × ratio, floor)`. Smaller = more entries re-sent; larger = wire savings at the cost of staleness. |
| `neorunbase.pagerank.distributed.delta.broadcast.floor` | `1e-12` | Absolute floor on the delta-broadcast threshold. Guards against a very small `convergenceThreshold` collapsing the derived threshold to ~zero. |

## Coordinator Cluster Sync

| Property | Default | Description |
| --- | --- | --- |
| `neorunbase.coordinator.sync.max.retries` | `3` | Maximum retries when the leader broadcasts metadata or IAM snapshots to peer coordinators over the internal protocol. |
| `neorunbase.coordinator.sync.retry.delay.ms` | `1000` | Sleep between sync retry attempts. With `max.retries` this caps total time spent retrying a single peer per round. |

## Cluster Bootstrap Fetch

On every startup, non-leader coordinators and data nodes pull authoritative KMS / metadata / IAM snapshots from the leader and overwrite their local RocksDB before signaling ready in ZooKeeper. If retries are exhausted the process exits with code 1 so the supervisor restarts it — it never serves stale state.

| Property | Default | Description |
| --- | --- | --- |
| `neorunbase.cluster.bootstrap.fetch.max.retries` | `30` | Maximum bootstrap-fetch attempts before the process exits with code 1. |
| `neorunbase.cluster.bootstrap.fetch.retry.delay.ms` | `1000` | Sleep between bootstrap-fetch retry attempts. |

## Sticky Leader Election

| Property | Default | Description |
| --- | --- | --- |
| `neorunbase.cluster.election.sticky.enabled` | `true` | When enabled, the previously-elected leader's node-id is persisted to ZK; non-incumbent nodes defer joining the election queue so the incumbent reclaims leadership on restart without a leadership toggle. Disabling falls back to first-lock-wins. |
| `neorunbase.cluster.election.deference.window.ms` | `3000` | How long non-incumbent nodes wait before entering the leader-selector queue at startup. Should be ≥ typical peer startup time. Increase for slow-booting nodes. |

## Distributed Query / DDL Timeouts

| Property | Default | Description |
| --- | --- | --- |
| `neorunbase.query.ann.rebuild.timeout.ms` | `300000` | RPC timeout for `ANN_REBUILD_REQ` when `REBUILD INDEX` is issued (5 min). Full vector index rebuilds can take minutes on large shards. |
| `neorunbase.query.table.sync.poll.interval.ms` | `100` | Poll interval used by `waitForTableSync` on non-leader coordinators when a freshly-created table needs to propagate from the leader's catalog. |
| `neorunbase.query.scatter.future.timeout.ms` | `30000` | Fallback per-shard scatter-future timeout used by inline scatter paths (full-table fetch, replica fan-out) that don't thread through `internal.rpc.timeout.ms`. |
| `neorunbase.query.merge.prepare.timeout.ms` | `60000` | Distributed merge PREPARE per-shard timeout (1 min). PREPARE writes pending tombstones + staged rows to the shard WAL, which may need to flush significant state. |
| `neorunbase.reshard.shard.operation.timeout.ms` | `120000` | Internal protocol RPC timeout for shard copy and shard replicate during resharding. |
| `neorunbase.reshard.ddl.operation.timeout.ms` | `60000` | Internal protocol RPC timeout for DDL fan-out (DROP TABLE, CREATE INDEX) during reshard. |

## Disk Repair

| Property | Default | Description |
| --- | --- | --- |
| `neorunbase.disk.repair.enabled` | `false` | Enable the automatic disk-repair service. When enabled it detects failed/full data dirs and re-replicates affected shards onto healthy disks/nodes. |
| `neorunbase.disk.repair.interval.seconds` | `300` | Period between disk-repair scan cycles (5 min). |
| `neorunbase.disk.repair.grace.period.seconds` | `600` | How long a disk must remain unhealthy before it is confirmed dead and its shards are repaired elsewhere (10 min). Avoids reacting to transient blips. |
| `neorunbase.disk.repair.max.concurrent` | `4` | Maximum number of shard repairs running at once. Caps the IO/network impact of recovery. |
| `neorunbase.disk.repair.min.available.bytes` | `104857600` | Minimum free space for a disk to count as healthy (100 MiB); below this it is excluded from new shard placement. |

## JVM Options

JVM flags applied to every NeorunBase process live in `conf/jvm.conf`. The shipped defaults enable G1 GC and string deduplication; heap and direct memory sizing are commented out and should be set per-host. Add additional `-Xms`, `-Xmx`, `-XX:MaxDirectMemorySize`, or `-XX:+...` flags as appropriate for the role and host.

```text
-XX:+UseG1GC
-XX:+UseStringDeduplication
-XX:+OptimizeStringConcat
-XX:+UseCondCardMark
-Dio.netty.noPreferDirect=true
```

The start scripts also append any `-D` and `-X` flags passed on the command line, so per-instance overrides do not require editing the shared file.
