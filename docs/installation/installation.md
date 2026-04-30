# Download & Install

NeorunBase is a distributed, sharded OLTP Lakebase that speaks the PostgreSQL wire protocol. This page covers how to download a NeorunBase release and install it on a single machine for evaluation, and how to lay out a multi-node cluster for production.

## Prerequisites

NeorunBase is written in Java and requires:

- **Java 17** or later on every Coordinator and Data Node host.
- **Apache ZooKeeper 3.6+** for cluster coordination, leader election, and service discovery. The release archive ships with an example ZooKeeper that can be started locally for evaluation.
- A POSIX-compatible filesystem on each Data Node for shard data, WAL, and heap segment files.
- **Master key** — every NeorunBase server process requires the `NEORUNBASE_MASTER_KEY` environment variable to be exported with a value of at least 32 characters. The master key is the root of the envelope-encryption hierarchy used for KMS, WAL, metadata, and at-rest data encryption.

## Download

Download and extract the latest NeorunBase release archive:

```agsl
curl -L -O https://github.com/cloudcheflabs/neorunbase-pack/releases/download/neorunbase-archive/neorunbase-1.0.0.tar.gz
tar zxvf neorunbase-1.0.0.tar.gz
cd neorunbase-1.0.0
```

The extracted directory contains:

| Path | Contents |
| --- | --- |
| `bin/` | Start/stop scripts for Coordinator, Data Node, and the bundled example ZooKeeper. |
| `lib/` | Server JARs and runtime dependencies. |
| `conf/neorunbase.properties` | Main configuration file. See the [Configuration](../configuration/configuration.md) reference. |
| `conf/jvm.conf` | JVM options applied to every NeorunBase process. |
| `conf/logback.xml` | Logback configuration. |
| `conf/zk/` | Configuration for the bundled example ZooKeeper. |
| `admin-ui/` | Built React SPA served by the Coordinator's admin HTTP server. |
| `examples/example.sql` | Sample SQL workload used by the quickstart. |
| `data/`, `logs/` | Default data and log directories (created at first start). |

## Single-node Quickstart

The quickest way to evaluate NeorunBase is to run one Coordinator and two Data Nodes on a single host alongside the bundled example ZooKeeper. The release ships a script that does this in one step:

```agsl
export NEORUNBASE_MASTER_KEY=test-master-key-for-integration-tests-12345
bin/start-example-servers.sh
```

This brings up:

- ZooKeeper on `localhost:2181`
- Two Data Nodes on internal ports `7002` and `7003`
- A Coordinator with PostgreSQL wire protocol on `localhost:5432` and the Admin UI on `http://localhost:8080/admin/`

To exercise the cluster end-to-end, follow the [Getting Started](getting-started.md) page, which walks through logging in to the Admin UI and running an example SQL workload through `psql`.

To stop everything:

```agsl
bin/stop-example-servers.sh
```

## Multi-node Deployment

For production NeorunBase is deployed across multiple hosts. A typical layout is:

- **ZooKeeper ensemble** — three or five dedicated nodes running ZooKeeper, separate from NeorunBase processes.
- **Coordinators** — two or more Coordinator nodes for high availability. One is elected leader for metadata and KMS; the others serve PG wire traffic and proxy admin operations.
- **Data Nodes** — one or more Data Node hosts. Shards are placed across Data Nodes with replication factor `neorunbase.default.replication.factor` (default `2`).

On every NeorunBase host:

1. Install Java 17 and unpack the same NeorunBase release archive.
2. Edit `conf/neorunbase.properties` so that `neorunbase.zookeeper.server.list` points to the production ZooKeeper ensemble, `neorunbase.coordinator.advertised.host` / `neorunbase.datanode.advertised.host` are set to the host's reachable hostname, and the cluster id (`neorunbase.cluster.id`) matches across all nodes.
3. Export `NEORUNBASE_MASTER_KEY` consistently on all nodes — it is the cluster's root encryption key.
4. Start each role:
    - Coordinator: `bin/start-coordinator.sh`
    - Data Node: `bin/start-datanode.sh`

Process management notes:

- `bin/start-coordinator.sh` and `bin/start-datanode.sh` accept `-D` system properties on the command line, which take precedence over `conf/neorunbase.properties` and `NEORUNBASE_*` environment variables. This is how the example launcher runs multiple Data Nodes on the same host with different ports.
- Each process writes its PID to `bin/<name>-<port>.pid` and its console output to `logs/<name>-<port>.out`. `bin/stop-coordinator.sh` and `bin/stop-datanode.sh` use these PID files for graceful shutdown.
- Set `NEORUNBASE_FOREGROUND=true` to run a server in the foreground (useful under systemd, supervisord, or container runtimes).

For the full list of tunables — sharding, replication, encryption, Iceberg, Kafka consumer groups, WAL, scrubber, IAM, internal protocol — see the [Configuration](../configuration/configuration.md) reference. For starting, stopping, and monitoring a running cluster, see [Operations](../operations/operations.md).
