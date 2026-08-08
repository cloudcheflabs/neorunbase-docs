# Operations

This page covers running a NeorunBase cluster in steady state: starting and stopping individual roles, controlling JVM and runtime behavior, and observing the cluster through the Admin UI and metrics. For first-time installation see [Installation](../installation/installation.md); for the full property reference see [Configuration](../configuration/configuration.md).

## Process Layout

A NeorunBase deployment is composed of three process types:

- **ZooKeeper ensemble** — used for cluster coordination, service discovery, and leader election. The release archive ships an example single-node ZooKeeper for evaluation; production deployments use a dedicated ensemble (typically three or five nodes).
- **Coordinator** — fronts the PostgreSQL wire protocol and the Admin UI. Two or more Coordinators may run for high availability; one is elected leader for metadata and KMS ownership.
- **Data Node** — owns shards on local disks. Hosts the heap segment files, RocksDB indexes, WAL, ANN sidecars, and the per-Data-Node Kafka consumer worker (when enabled).

All three are managed by shell scripts under `bin/` in the release archive.

## Starting and Stopping

### Coordinator

```bash
bin/start-coordinator.sh
```

The script launches `com.cloudcheflabs.neorunbase.server.CoordinatorServer` with the classpath set to `conf:lib/*`, applies the JVM options from `conf/jvm.conf`, and writes its PID to `bin/coordinator-<pg-port>.pid`. Standard out and err are redirected to `logs/coordinator-<pg-port>.out`. The Coordinator binds the PG wire port (`neorunbase.coordinator.pg.port`, default `5432`) and the Admin HTTP port (`neorunbase.admin.http.port`, default `8080`).

To stop the Coordinator gracefully:

```bash
bin/stop-coordinator.sh
```

To stop a specific Coordinator instance running with a non-default port (e.g. when multiple Coordinators are colocated on the same host for testing):

```bash
bin/stop-coordinator.sh <pg-port>
```

The stop script sends `SIGTERM` and waits up to 30 seconds for the process to exit before falling back to `SIGKILL`.

### Data Node

```bash
bin/start-datanode.sh
```

This launches `com.cloudcheflabs.neorunbase.server.DataNodeServer` and writes its PID to `bin/datanode-<internal-port>.pid` and console output to `logs/datanode-<internal-port>.out`. The Data Node binds `neorunbase.datanode.internal.port` (default `7000`) for the internal protocol from Coordinators.

Stop with:

```bash
bin/stop-datanode.sh
```

or, for a specific instance by port:

```bash
bin/stop-datanode.sh <internal-port>
```

### ZooKeeper

The release ships a single-node example ZooKeeper for evaluation:

```bash
bin/start-zk.sh
bin/stop-zk.sh
```

Production clusters should point `neorunbase.zookeeper.server.list` at a dedicated, separately managed ZooKeeper ensemble rather than the bundled example.

### Bringing Up the Whole Example Cluster

The example launcher boots ZooKeeper, two Data Nodes, and one Coordinator on a single host:

```bash
export NEORUNBASE_MASTER_KEY=test-master-key-for-integration-tests-12345
bin/start-example-servers.sh
```

The corresponding teardown:

```bash
bin/stop-example-servers.sh
```

## Process Control & Tuning

### Foreground vs. Background

By default the start scripts daemonize the Java process. Set `NEORUNBASE_FOREGROUND=true` to run in the foreground instead — required when NeorunBase is supervised by `systemd`, `runit`, or another process manager that expects the child to stay attached.

```bash
NEORUNBASE_FOREGROUND=true bin/start-coordinator.sh
```

### Per-instance Overrides

The start scripts forward every `-D` and `-X` argument on the command line to the JVM, and forward any other arguments to the server's main method. This is how `start-example-servers.sh` runs two Data Nodes from a single config file:

```bash
bin/start-datanode.sh \
    -Dneorunbase.datanode.internal.port=7002 \
    -Dneorunbase.base.data.dir=data/datanode-1 \
    -Dneorunbase.log.path=logs/datanode-1
```

Per-instance system properties take precedence over `neorunbase.properties` and over `NEORUNBASE_*` environment variables.

### Log Path

The effective log directory is selected with this precedence (highest first):

1. `-Dneorunbase.log.path=...` on the command line.
2. `NEORUNBASE_LOG_PATH` environment variable.
3. `logs/` under the install directory (default).

Each instance also writes a per-port `.out` file derived from the role and detected port (e.g. `coordinator-5432.out`, `datanode-7002.out`) so that colocated instances do not collide.

### JVM Options

Edit `conf/jvm.conf` to set heap size, direct memory cap, GC flags, and any other JVM tuning. The file is read line-by-line, ignoring comments and blank lines, and prepended to the JVM command line. Per-instance overrides can still be passed on the command line.

```text
-Xms8g
-Xmx8g
-XX:MaxDirectMemorySize=4g
-XX:+UseG1GC
-XX:+UseStringDeduplication
```

## Master Key Handling

`NEORUNBASE_MASTER_KEY` must be exported on every NeorunBase host with the same value. In production:

- Inject the master key from a secrets manager into the process environment (for example via `systemd` `EnvironmentFile=`, an entrypoint wrapper, or a sidecar that materializes secrets to a tmpfs file).
- Keep the master key out of `conf/neorunbase.properties` and out of process listings — pass it through the env var only.
- Treat the master key as cluster-critical: losing it makes encrypted on-disk state unrecoverable.

## Admin CLI

The release ships a small administrative CLI, `bin/neorunbase-cli.sh` (`neorunbase-cli`), for local operations against a running Coordinator. Unlike the Admin UI / REST API, the CLI does **not** authenticate over the network: it connects to the Coordinator's local **Unix domain socket**, and authentication is purely OS-level (the socket is created with file mode `600`). You must run it as the **same OS user** as the Coordinator process (or `root`) on the **same host or container**.

The socket path is resolved in this order:

1. `--socket PATH` on the command line — always wins.
2. `$NEORUNBASE_ADMIN_SOCKET`, if exported in the caller's shell.
3. The path the running Coordinator published to `bin/coordinator.socket`. The
   marker file name comes from `neorunbase.admin.socket.marker.file`, so both the
   Coordinator and the CLI pick up a rename from the same properties file.
4. `neorunbase.admin.socket.path` in `conf/neorunbase.properties`, with
   `${neorunbase.base.data.dir}` expanded (a value left holding a `${...}`
   placeholder is rejected rather than used literally).
5. The default `data/admin.sock` under the install directory, then `/data/admin.sock`.

Step 3 matters whenever the data dir was moved: launching with
`-Dneorunbase.base.data.dir=data/coordinator-1` puts the socket at
`data/coordinator-1/admin.sock` while `conf/neorunbase.properties` still reads
`./data`, and only the published marker records where the live process bound.

`NEORUNBASE_MASTER_KEY` is **not** needed to run the CLI — the Coordinator holds the
unsealed key and performs the work. The variable is required by the Coordinator
process itself.

The CLI supports exactly two commands:

| Command | Purpose |
| --- | --- |
| `ping` | Check that the local admin socket is reachable (prints `pong`). |
| `iam:reset-password` | Reset a user's password — the recovery path when the admin password is lost and you cannot log into the Admin UI. |

### Password Recovery

`iam:reset-password` resets a user's password without needing an existing login. Options:

| Option | Default | Description |
| --- | --- | --- |
| `--user USER` | `admin` | The user whose password to reset. |
| `--new-password PWD` | (generated) | Set an explicit new password. Use `--new-password -` to read the password from stdin (one line). If omitted, the Coordinator generates a temporary password and prints it. |
| `--interactive` | — | Prompt for the new password on the TTY (hidden, with confirmation). |
| `--socket PATH` | (see above) | Override the admin socket path. |

```bash
# Reachability check
bin/neorunbase-cli.sh ping

# Reset the admin password to a server-generated temporary value (printed to the terminal)
bin/neorunbase-cli.sh iam:reset-password --user admin

# Reset to an explicit password read from stdin
printf 'my-new-secret' | bin/neorunbase-cli.sh iam:reset-password --user admin --new-password -
```

The reset account is flagged to require a password change on next login.

## Cluster Lifecycle Notes

- **Bootstrap order.** Start ZooKeeper first, then Data Nodes, then Coordinators. The example launcher follows this sequence with brief sleeps; in production, leave Data Nodes time to register in ZooKeeper before bringing up Coordinators so the leader's startup discovery loop completes promptly.
- **Coordinator HA.** When more than one Coordinator is running, ZooKeeper elects a leader for metadata and KMS ownership. Non-leader Coordinators forward write-side admin operations to the leader and serve PG wire traffic locally.
- **Shard repair & rebalancing.** When a Data Node leaves the cluster, the leader Coordinator detects the absence via ZooKeeper ephemeral nodes and replicates affected shards onto healthy Data Nodes (provided `neorunbase.disk.repair.enabled=true` for the disk-level case). When new Data Nodes join, the cluster rebalances shards toward them. See [Replication & High Availability](../features/replication-ha.md).
- **Disaster recovery (cross-site).** A separate, **leader-only** streamer ships WAL records to one or more standby clusters via `REPLICATE_WAL_BATCH` over the internal protocol. Configure peers under `/admin/api/site-replication/config` and monitor `lastAppliedSeq` and per-peer queue depth under `/admin/api/site-replication/status`. See [Site Replication (DR)](../features/site-replication.md).
- **Iceberg sync.** Set `neorunbase.iceberg.catalog.type=polaris` (the only supported catalog backend besides `none`) and `neorunbase.iceberg.sync.auto.start=true` to have the leader Coordinator run background incremental sync. Per-table filters and the changelog batch size are tunable — see [Configuration](../configuration/configuration.md#iceberg-catalog-integration).
- **Kafka ingestion.** Set `neorunbase.kafka.consumer.enabled=true` and define one or more consumer groups. The leader Coordinator distributes the configured groups round-robin across Data Nodes; each worker forwards parsed inserts back to the leader Coordinator over the internal protocol so the standard shard routing path applies. See [Kafka Integration](../features/kafka-integration.md).
- **CSR adjacency for graph workloads.** Set `neorunbase.csr.enabled=true` and list edge tables in `neorunbase.csr.enabled.tables` to opt into the CSR fast path. The leader Coordinator schedules background compactions every `neorunbase.csr.compaction.interval.seconds`; force a rebuild after bulk inserts via `POST /admin/graph/csr/compact`, and inspect per-shard segment counts / `src_id` totals via `GET /admin/graph/csr/status`. See [Graph Traversal & Analytics](../features/graph.md#csr-adjacency-phase-1-acceleration-layer).

## Observability

### Admin UI

The Admin UI is the primary operator surface. It is served by every Coordinator at `http://<coordinator-host>:<neorunbase.admin.http.port>/admin/` (default `8080`).

It exposes:

- Cluster topology (Coordinators, Data Nodes, leader, ZooKeeper, version).
- Tables, columns, indexes, and shard maps including replica placement.
- Live and historical metrics charts driven by the Coordinator's metrics RocksDB store.
- IAM management — users, roles, policies, and STS sessions.
- KMS status and the encrypted-feature toggles (data, WAL, metadata, internal protocol).
- Live log tailing across all Coordinators and Data Nodes.

See [Admin UI](../features/admin-ui.md) for the full feature description.

### Metrics

The leader Coordinator scrapes every Coordinator and Data Node every `neorunbase.metrics.collector.interval.ms` milliseconds (default `15000`) and persists the rolled-up time series for `neorunbase.metrics.retention.days` days (default `7`). The retention sweep runs on the schedule given by `neorunbase.metrics.retention.cleanup.*`. Both raw metrics and aggregated charts are exposed through the Admin UI and the admin REST API.

### Logs

Each NeorunBase process writes:

- `logs/<role>-<port>.out` — JVM stdout / stderr captured by the start script (when running in background mode).
- The Logback-managed log files configured by `conf/logback.xml`, under `neorunbase.log.path` (which itself defaults to `${neorunbase.base.data.dir}/logs`).

Setting `neorunbase.log.mode=RING_BUFFER` switches file-based logging off in favor of an in-memory ring buffer of `neorunbase.log.ring.buffer.capacity` lines, which the Admin UI tails directly. This is the recommended mode for environments where local disk is ephemeral or unavailable.

### Bitrot Scrubber

Setting `neorunbase.scrubber.enabled=true` turns on a throttled background scanner that re-verifies the AES-GCM auth tag on every record in every closed heap segment file. The scanner paces itself at `neorunbase.scrubber.throttle.bytes.per.sec` (default 50 MB/s) so it does not interfere with OLTP traffic, and surfaces any corruption findings through the Admin UI and the admin REST API. Bitrot is detected on read regardless of this toggle; the scrubber adds proactive scanning.

## Shutdown

For a clean shutdown, stop processes in the reverse of startup order:

1. Stop Coordinators (`bin/stop-coordinator.sh`).
2. Stop Data Nodes (`bin/stop-datanode.sh`).
3. Stop ZooKeeper (only if you started it via `bin/start-zk.sh`).

`bin/stop-example-servers.sh` performs this sequence for the example single-host cluster.

When the Coordinator's PG wire worker pool is draining, `neorunbase.pgwire.worker.shutdown.timeout.seconds` (default `5`) controls how long it waits before forcibly terminating in-flight queries. The Admin HTTP server has its own grace period, `neorunbase.admin.shutdown.timeout.seconds`. Tune both upward if the cluster is asked to shed long-running analytical queries during planned shutdowns.
