# Configuration

NeorunBase is configured through `conf/neorunbase.properties` in the release archive. Every property has a sensible default; you typically only override cluster identity, ZooKeeper connection, advertised hostnames, ports, and whichever feature toggles (Iceberg, Kafka, encryption, scrubber) your deployment uses.

## Configuration Sources & Precedence

NeorunBase resolves each configuration key from three sources, **in order**:

1. **JVM system property** — e.g. `-Dneorunbase.coordinator.pg.port=5433` on the command line.
2. **Environment variable** — e.g. `NEORUNBASE_COORDINATOR_PG_PORT=5433`. The mapping is uppercase with dots replaced by underscores.
3. **Properties file** — `conf/neorunbase.properties`, the on-disk default.

A higher source always wins. This lets the start scripts in `bin/` override individual settings on the command line (the example launcher uses this to start two Data Nodes on different ports from a single config file), and lets containerized or systemd-managed deployments inject ports, hostnames, and key material via environment variables without rewriting the file.

Property values can reference other properties with `${...}` substitution. For example:

```properties
neorunbase.base.data.dir=./data
neorunbase.datanode.data.dir=${neorunbase.base.data.dir}/shards
neorunbase.wal.directory=${neorunbase.base.data.dir}/wal
```

## Master Key

Every NeorunBase server process reads a master key from the environment variable named by `neorunbase.kms.master.key.env` (default `NEORUNBASE_MASTER_KEY`). The master key:

- Must be **at least 32 characters** long.
- Must be **identical across all Coordinators and Data Nodes** in the cluster.
- Is the root of the envelope-encryption hierarchy. PBKDF2 with `neorunbase.kms.pbkdf2.iterations` iterations (default `200000`) derives the master encryption key, which wraps the per-key DEKs used for KMS, internal protocol, metadata, WAL, heap segments, and at-rest shard data.
- Is also the secret used to derive the JWT signing key for the admin HTTP server.

Losing the master key makes all encrypted on-disk state and KMS-wrapped DEKs unrecoverable. Keep it in a secret manager and rotate per your security policy.

## Cluster

| Property | Default | Description |
| --- | --- | --- |
| `neorunbase.cluster.id` | `neorunbase-cluster` | Unique identifier for this NeorunBase cluster. Must match across all nodes. |
| `neorunbase.zookeeper.server.list` | `localhost:2181` | ZooKeeper connection string (`host1:port1,host2:port2`). |
| `neorunbase.zookeeper.root.path` | `/neorunbase` | Root znode path under which all NeorunBase znodes are created. |
| `neorunbase.zookeeper.session.timeout.ms` | `30000` | ZooKeeper session timeout. |
| `neorunbase.zookeeper.connection.timeout.ms` | `10000` | ZooKeeper connection timeout. |
| `neorunbase.base.data.dir` | `./data` | Base directory for all NeorunBase data. Most other directory properties derive from this. |

## Coordinator

| Property | Default | Description |
| --- | --- | --- |
| `neorunbase.coordinator.host` | `0.0.0.0` | Bind address for the PG wire protocol listener. |
| `neorunbase.coordinator.pg.port` | `5432` | TCP port for PostgreSQL wire protocol connections. |
| `neorunbase.coordinator.advertised.host` | `localhost` | Hostname advertised to clients and other nodes. Set to a reachable hostname in production. |
| `neorunbase.coordinator.worker.threads` | `235` | Worker threads handling PG wire protocol connections. |
| `neorunbase.coordinator.internal.port` | `7100` | TCP port for coordinator-to-coordinator NIO communication (metadata sync, KMS sync). |

## Data Node

| Property | Default | Description |
| --- | --- | --- |
| `neorunbase.datanode.host` | `0.0.0.0` | Bind address for the internal protocol listener. |
| `neorunbase.datanode.internal.port` | `7000` | TCP port for the internal protocol from Coordinators. |
| `neorunbase.datanode.advertised.host` | `localhost` | Hostname advertised to Coordinators. Set to a reachable hostname in production. |
| `neorunbase.datanode.data.dir` | `${neorunbase.base.data.dir}/shards` | Local directory for shard data. Multiple comma-separated directories enable multi-disk placement. |
| `neorunbase.datanode.worker.threads` | `256` | Worker threads handling internal protocol requests. |
| `neorunbase.datanode.disk.info.refresh.interval.seconds` | `60` | How often the Data Node refreshes disk capacity / availability info in ZooKeeper. |
| `neorunbase.datanode.temp.table.cleanup.interval.seconds` | `60` | How often the Data Node sweeps for expired intermediate-result temp tables. |
| `neorunbase.datanode.temp.table.ttl.ms` | `600000` | Temp-table TTL before cleanup is eligible (10 minutes). |

## Default Table Settings

| Property | Default | Description |
| --- | --- | --- |
| `neorunbase.default.num.shards` | `16` | Default number of shards when `CREATE TABLE` does not specify `SHARDS`. |
| `neorunbase.default.replication.factor` | `2` | Default replication factor for shards. |

## Authentication

| Property | Default | Description |
| --- | --- | --- |
| `neorunbase.admin.user` | `admin` | Initial admin user for PG wire protocol and Admin UI. |
| `neorunbase.admin.password` | `admin` | Initial admin password. **Change on first login.** |

## Logging & Monitoring

| Property | Default | Description |
| --- | --- | --- |
| `neorunbase.log.mode` | `ASYNC_FILE` | `ASYNC_FILE` (rolling files), `RING_BUFFER` (in-memory ring buffer for container environments), or `NOP` (console only). |
| `neorunbase.log.path` | `${neorunbase.base.data.dir}/logs` | Base directory for log files. |
| `neorunbase.log.level` | `INFO` | Log level for NeorunBase application loggers. |
| `neorunbase.log.ring.buffer.capacity` | `1000` | Number of log lines retained in the in-memory ring buffer (when `RING_BUFFER` is selected). |

## KMS (Key Management Service)

| Property | Default | Description |
| --- | --- | --- |
| `neorunbase.kms.enabled` | `true` | Master toggle for the built-in KMS. When enabled the internal protocol is encrypted. |
| `neorunbase.kms.rocksdb.path` | `${neorunbase.base.data.dir}/kms` | Local KMS key persistence directory. |
| `neorunbase.kms.master.key.env` | `NEORUNBASE_MASTER_KEY` | Name of the env var holding the master key passphrase. |
| `neorunbase.kms.pbkdf2.iterations` | `200000` | PBKDF2 iterations for master-key derivation. |
| `neorunbase.kms.encrypt.internal.protocol` | `true` | Encrypt internal protocol messages between Coordinator and Data Nodes. |
| `neorunbase.kms.internal.protocol.key.id` | `internal-protocol` | Key ID used for internal protocol encryption. |

## Admin HTTP Server

| Property | Default | Description |
| --- | --- | --- |
| `neorunbase.admin.http.port` | `8080` | HTTP port for the Admin UI and REST API. |
| `neorunbase.admin.ui.static.path` | `./admin-ui` | Path to the React Admin UI build. |
| `neorunbase.admin.worker.threads` | `256` | Worker threads for admin HTTP request handling. |
| `neorunbase.admin.jwt.expiry.ms` | `86400000` | JWT token expiry (24 hours). JWT signing uses SHA-256 of the master key. |
| `neorunbase.admin.http.max.content.length` | `10485760` | Maximum HTTP request body size (10 MB). |
| `neorunbase.admin.shutdown.timeout.seconds` | `5` | Admin HTTP server shutdown grace period. |
| `neorunbase.admin.http.proxy.connect.timeout.ms` | `5000` | Connect timeout for coordinator-to-coordinator HTTP proxy calls. |
| `neorunbase.admin.http.proxy.read.timeout.ms` | `30000` | Read timeout for the same proxy calls. Increase for large log-tail responses. |

## Metrics Collection

| Property | Default | Description |
| --- | --- | --- |
| `neorunbase.metrics.collector.interval.ms` | `15000` | Interval between metric collection rounds across cluster nodes. |
| `neorunbase.metrics.collector.initial.delay.ms` | `10000` | Initial delay before metrics collection starts. |
| `neorunbase.metrics.retention.days` | `7` | Number of days to retain historical metrics. |
| `neorunbase.metrics.rocksdb.path` | `${neorunbase.base.data.dir}/metrics` | Metrics time-series storage directory. |
| `neorunbase.metrics.collector.connect.timeout.ms` | `3000` | HTTP connect timeout for metric scraping. |
| `neorunbase.metrics.collector.read.timeout.ms` | `5000` | HTTP read timeout for metric scraping. |
| `neorunbase.metrics.retention.cleanup.initial.delay.minutes` | `60` | Initial delay before retention cleanup. |
| `neorunbase.metrics.retention.cleanup.interval.minutes` | `360` | Retention cleanup interval. |

## Metadata Store (Coordinator)

| Property | Default | Description |
| --- | --- | --- |
| `neorunbase.metadata.rocksdb.path` | `${neorunbase.base.data.dir}/metadata` | Encrypted metadata store on the leader Coordinator. Holds table schemas and shard maps. |
| `neorunbase.metadata.kms.key.id` | `metadata-encryption` | KMS key ID for metadata at-rest encryption. |

## Data At-Rest Encryption

| Property | Default | Description |
| --- | --- | --- |
| `neorunbase.data.encryption.enabled` | `true` | Enable at-rest encryption. Each shard uses a per-shard DEK wrapped via the KMS. |
| `neorunbase.data.encryption.kms.key.id` | `data-encryption` | KMS key ID used to wrap per-shard DEKs. |

## Internal Protocol Tuning

| Property | Default | Description |
| --- | --- | --- |
| `neorunbase.protocol.compression.enabled` | `true` | Snappy compression for internal protocol messages. |
| `neorunbase.protocol.compression.min.bytes` | `256` | Minimum payload size to trigger compression. |
| `neorunbase.nio.read.buffer.size` | `65536` | NIO read buffer size for internal protocol connections. |
| `neorunbase.nio.selector.timeout.ms` | `1000` | NIO selector poll timeout. |
| `neorunbase.internal.rpc.timeout.ms` | `5000` | Internal RPC request timeout. |
| `neorunbase.datanode.discovery.poll.interval.ms` | `2000` | Coordinator startup poll interval for Data Node discovery. |
| `neorunbase.datanode.discovery.max.retries` | `30` | Maximum Data Node discovery retries during startup. |
| `neorunbase.leader.init.poll.interval.ms` | `1000` | Non-leader Coordinator initialization poll interval. |
| `neorunbase.leader.init.max.retries` | `15` | Non-leader initialization max retries. |

## Iceberg Catalog Integration

NeorunBase can synchronize tables to an Apache Polaris Iceberg REST catalog. See [Iceberg Integration](../features/iceberg-integration.md) for the full feature description.

| Property | Default | Description |
| --- | --- | --- |
| `neorunbase.iceberg.catalog.type` | `none` | `none` (disabled) or `polaris`. |
| `neorunbase.iceberg.polaris.uri` |  | Polaris REST endpoint, e.g. `http://shannon-polaris:8181/api/catalog`. |
| `neorunbase.iceberg.polaris.oauth.token.endpoint` |  | Polaris OAuth2 token endpoint, e.g. `http://shannon-polaris:8181/api/catalog/v1/oauth/tokens`. |
| `neorunbase.iceberg.polaris.catalog.name` |  | Polaris catalog name (used as Iceberg `prefix` and `?warehouse=` parameter). |
| `neorunbase.iceberg.polaris.client.id` |  | OAuth2 client_credentials principal id. |
| `neorunbase.iceberg.polaris.client.secret` |  | OAuth2 client_credentials secret. |
| `neorunbase.iceberg.polaris.realm` | `POLARIS` | Sent as the `Polaris-Realm` request header. |
| `neorunbase.iceberg.polaris.scope` | `PRINCIPAL_ROLE:ALL` | OAuth2 scope on token requests. |
| `neorunbase.iceberg.s3.endpoint` | `http://localhost:9000` | S3 endpoint for Iceberg storage. |
| `neorunbase.iceberg.s3.access.key` | `admin` | S3 access key. |
| `neorunbase.iceberg.s3.secret.key` | `admin123` | S3 secret key. |
| `neorunbase.iceberg.s3.region` | `us-east-1` | S3 region. |
| `neorunbase.iceberg.s3.path.style.access` | `true` | Path-style access (required for MinIO and similar). |
| `neorunbase.iceberg.s3.sse.type` | `none` | `none`, `s3` (SSE-S3), or `kms` (SSE-KMS) for Parquet objects written by the Iceberg sink. |
| `neorunbase.iceberg.s3.sse.key` |  | KMS key id/ARN when `sse.type=kms`. |
| `neorunbase.iceberg.sync.interval.ms` | `60000` | Background sync interval. `0` disables. |
| `neorunbase.iceberg.sync.thread.pool.size` | `4` | Thread pool size for parallel table syncs. |
| `neorunbase.iceberg.sync.auto.start` | `false` | Auto-start sync job on server startup. |
| `neorunbase.iceberg.sync.table.filters` |  | Comma-separated table filters (`*` wildcard supported). Empty = all dirty tables. |
| `neorunbase.iceberg.sync.changelog.batch.size` | `500` | Batch size for changelog reads during incremental sync. |
| `neorunbase.iceberg.sync.task.timeout.ms` | `300000` | Per-table sync task timeout. |
| `neorunbase.iceberg.default.namespace` | `lakebase` | Default Iceberg namespace, auto-created on catalog init. |

## S3 Backup

Scheduled cluster backup to S3-compatible storage. See [Backup & Restore](../features/backup-restore.md) for the full feature description. Most operators configure these from the admin UI rather than from properties; the persisted values are stored encrypted in the metadata store.

| Property | Default | Description |
| --- | --- | --- |
| `neorunbase.backup.s3.enabled` | `false` | Master switch for scheduled backups. |
| `neorunbase.backup.s3.endpoint` |  | S3 endpoint URL. |
| `neorunbase.backup.s3.region` | `us-east-1` | S3 region. |
| `neorunbase.backup.s3.bucket` |  | Destination bucket. |
| `neorunbase.backup.s3.prefix` | `neorunbase-backup` | Object key prefix under the bucket. |
| `neorunbase.backup.s3.access.key` |  | Static access key. |
| `neorunbase.backup.s3.secret.key` |  | Static secret key (encrypted at rest). |
| `neorunbase.backup.s3.path.style.access` | `true` | Path-style addressing (required for ShannonStore / MinIO). |
| `neorunbase.backup.interval.minutes` | `60` | Scheduled backup interval. |
| `neorunbase.backup.retention.days` | `30` | History retention window for the visible backup list. |

## Kafka Consumer

NeorunBase can ingest JSON messages directly from Kafka into target tables. See [Kafka Integration](../features/kafka-integration.md).

| Property | Default | Description |
| --- | --- | --- |
| `neorunbase.kafka.consumer.enabled` | `false` | Master toggle for Kafka consumer integration. |
| `neorunbase.kafka.consumer.groups` |  | Comma-separated list of consumer group names. |

For each group `<name>` listed in `neorunbase.kafka.consumer.groups`, configure a per-group block:

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

The leader Coordinator distributes consumer groups round-robin across the Data Nodes; each Data Node hosts the consumer worker for the groups assigned to it.

## IAM (Identity & Access Management)

| Property | Default | Description |
| --- | --- | --- |
| `neorunbase.iam.rocksdb.path` | `${neorunbase.base.data.dir}/iam` | RocksDB path for IAM data persistence. |
| `neorunbase.iam.sts.cleanup.interval.hours` | `1` | Sweep interval for expired STS temporary credentials. |

## Write-Ahead Log (WAL) & Encryption

| Property | Default | Description |
| --- | --- | --- |
| `neorunbase.wal.encryption.enabled` | `true` | Enable WAL envelope encryption. |
| `neorunbase.wal.encryption.kms.key.id` | `wal-encryption` | KMS key ID for WAL segment DEKs. |
| `neorunbase.wal.directory` | `${neorunbase.base.data.dir}/wal` | Base WAL directory. Per-shard subdirectories are created automatically. |
| `neorunbase.wal.segment.size` | `67108864` | WAL segment rotation threshold (64 MB). |
| `neorunbase.wal.sync.interval.ms` | `1000` | WAL fsync interval. |

## Row-Data Storage Mode

| Property | Default | Description |
| --- | --- | --- |
| `neorunbase.storage.row.data.mode` | `heap` | `heap` (default — append-only encrypted heap segments + 16-byte RocksDB pointer per row) or `rocksdb` (legacy escape hatch; values stored inside RocksDB). Indexes, changelog, and system column families always remain in RocksDB. |
| `neorunbase.heap.segment.size` | `67108864` | Heap segment rotation threshold (64 MB). |
| `neorunbase.heap.encryption.kms.key.id` | `heap-encryption` | KMS key ID for heap segment DEKs. |

## Bitrot Scrubber

| Property | Default | Description |
| --- | --- | --- |
| `neorunbase.scrubber.enabled` | `false` | Enable the periodic background scrubber that walks closed heap segments and verifies every record's AES-GCM auth tag. When disabled, bitrot is still detected on read but no proactive scan runs. |
| `neorunbase.scrubber.interval.minutes` | `360` | Minutes between scrub passes (6 hours). |
| `neorunbase.scrubber.throttle.bytes.per.sec` | `52428800` | Maximum disk throughput the scrubber consumes (50 MB/s). |

## PG Wire Protocol Server Tuning

| Property | Default | Description |
| --- | --- | --- |
| `neorunbase.pgwire.read.buffer.initial.bytes` | `8192` | Initial per-connection read buffer size. Grown on demand up to the cap below. |
| `neorunbase.pgwire.read.buffer.max.bytes` | `67108864` | Absolute upper bound on a single inbound pg-wire message (64 MiB). Larger frames receive ErrorResponse and connection close. |
| `neorunbase.pgwire.worker.pool.size` | `0` | pg-wire worker pool size. `0` = auto: `max(2, 2 × CPU cores)`. |
| `neorunbase.pgwire.worker.shutdown.timeout.seconds` | `5` | Worker pool drain grace period at shutdown. |

## Vector ANN Index Maintenance

| Property | Default | Description |
| --- | --- | --- |
| `neorunbase.ann.flush.interval.seconds` | `30` | Background flush interval for dirty in-memory ANN indexes to encrypted sidecar files. `0` disables periodic flush — sidecars are only written on shard close. Lower values bound the data-loss window on ungraceful shutdown; higher values amortize the full-graph re-serialize cost. |

## Coordinator Cluster Sync

| Property | Default | Description |
| --- | --- | --- |
| `neorunbase.coordinator.sync.max.retries` | `3` | Maximum retries when the leader broadcasts metadata or IAM snapshots to peer Coordinators over the internal protocol. |
| `neorunbase.coordinator.sync.retry.delay.ms` | `1000` | Sleep between sync retry attempts. |

## Distributed Query / DDL Timeouts

| Property | Default | Description |
| --- | --- | --- |
| `neorunbase.query.ann.rebuild.timeout.ms` | `300000` | RPC timeout for `ANN_REBUILD_REQ` (5 minutes). Vector index rebuilds can take minutes on large shards. |
| `neorunbase.query.table.sync.poll.interval.ms` | `100` | Non-leader Coordinator wait poll for newly-created tables to propagate from the leader. |
| `neorunbase.reshard.shard.operation.timeout.ms` | `120000` | Internal protocol RPC timeout for shard copy / replicate during resharding. |
| `neorunbase.reshard.ddl.operation.timeout.ms` | `60000` | Internal protocol RPC timeout for DDL fan-out (DROP TABLE, CREATE INDEX) during reshard. |

## Disk Repair

| Property | Default | Description |
| --- | --- | --- |
| `neorunbase.disk.repair.enabled` | `false` | Enable automatic disk repair. |
| `neorunbase.disk.repair.interval.seconds` | `300` | Repair cycle interval. |
| `neorunbase.disk.repair.grace.period.seconds` | `600` | Grace period before a disk is confirmed dead. |
| `neorunbase.disk.repair.max.concurrent` | `4` | Maximum concurrent shard repairs. |
| `neorunbase.disk.repair.min.available.bytes` | `104857600` | Minimum free bytes for a disk to be considered healthy (100 MB). |

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
