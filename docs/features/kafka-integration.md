# Kafka Integration

NeorunBase supports streaming data ingestion from Apache Kafka, enabling real-time data to flow directly into NeorunBase tables.

## Streaming Ingestion

NeorunBase can consume messages from Kafka topics and automatically insert them into designated tables. This provides a seamless pipeline from event streams to a queryable relational database.

## Multiple Consumer Groups

You can configure multiple independent Kafka consumer groups, each consuming from different topics and writing to different tables. This allows you to ingest data from various Kafka sources simultaneously.

## Where Consumers Run

Kafka consumer workers run on **Data Nodes**, not on Coordinators. The leader Coordinator computes a round-robin assignment of consumer groups across the live Data Nodes and broadcasts it via the internal protocol; each Data Node then starts only the workers it has been assigned. Each worker pulls from its Kafka partitions and forwards the parsed INSERT to the leader Coordinator over the internal protocol, so the Coordinator's normal shard routing, replication, and transaction handling apply uniformly to ingested rows. Adding or removing Data Nodes triggers a re-assignment automatically.

## Configuration

Kafka ingestion is configured in `neorunbase.properties` (or the matching `-D` /
environment overrides). Two top-level keys turn it on and list the groups to run:

| Property | Default | Description |
| --- | --- | --- |
| `neorunbase.kafka.consumer.enabled` | `false` | Master switch. When `true`, the groups listed below are started. |
| `neorunbase.kafka.consumer.groups` | *(empty)* | Comma-separated list of consumer-group names to start. Empty = no consumers even when `enabled=true`. |
| `neorunbase.kafka.consumer.leader.rpc.timeout.ms` | `30000` | RPC timeout for a Data Node's Kafka worker forwarding a batch INSERT/DDL to the leader Coordinator. |

Each name `<name>` in `...consumer.groups` expands to a block of
`neorunbase.kafka.consumer.group.<name>.*` keys that fully describe that pipeline:

| Per-group key | Default | Description |
| --- | --- | --- |
| `.brokers` | `localhost:9092` | Kafka bootstrap servers. |
| `.topics` | *(empty)* | Comma-separated topics to consume. |
| `.group-id` | `neorunbase-<name>` | Kafka `group.id`. |
| `.target-table` | *(empty)* | NeorunBase table to ingest into. |
| `.id-columns` | *(empty)* | Primary-key / upsert columns (enables upsert semantics on matching rows). |
| `.num-shards` | `2` | Shard count if the target table is auto-created. |
| `.format` | `json` | Record format. |
| `.batch-size` | `100` | Records buffered per INSERT batch. |
| `.poll-timeout-ms` | `1000` | `KafkaConsumer.poll` timeout (ms). |
| `.auto-offset-reset` | `latest` | `earliest` or `latest`. |
| `.extra.properties` | *(empty)* | Raw `KafkaConsumer` props as `k=v,k=v` (e.g. `max.poll.records=500`). |
| `.max-consumers` | `3` | Max parallel consumer threads for the group. |

## JSON Message Format

`format=json` (the default) consumes JSON-formatted messages from Kafka topics. Messages
are parsed and batch-inserted into the target table, matching JSON fields to table columns.
When `id-columns` is set, matching rows are upserted rather than blindly inserted.

## Management

Kafka consumer groups can be managed at runtime through the JWT-authenticated admin REST
API (leader-forwarded from any coordinator):

| Method & Path | Description |
| --- | --- |
| `GET /admin/api/kafka/config` | Fetch the current Kafka consumer configuration. |
| `POST /admin/api/kafka/config` | Update the configuration (enable/disable, groups, per-group keys). |
| `GET /admin/api/kafka/status` | Report per-group running status. |
| `POST /admin/api/kafka/groups/<name>/start` | Start a consumer group. |
| `POST /admin/api/kafka/groups/<name>/stop` | Stop a consumer group. |

Because consumers run on the Data Nodes, these leader endpoints fan out the start/stop and
config changes to the Data Nodes over the internal protocol.
