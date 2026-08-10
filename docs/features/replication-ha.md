# Replication & High Availability

NeorunBase provides fault tolerance and high availability through shard replication and automatic failure recovery.

This page covers **intra-cluster** replication — copies of each shard inside a single cluster, sharing coordinators, KMS, and ZooKeeper. For cross-cluster, cross-site, disaster-recovery streaming, see [Site Replication (DR)](site-replication.md).

## Shard Replication

Each shard can be configured with a replication factor. NeorunBase maintains multiple copies of each shard across different Data Nodes. This ensures that data remains available even if one or more Data Nodes go down. When a table does not override it, the cluster default `neorunbase.default.replication.factor` (2 — one primary plus one replica) applies (`neorunbase.properties`, section "Default Settings"). The cluster needs at least `replication_factor` Data Nodes to place all copies of a shard.

Set the replication factor at table creation with `REPLICAS N` or `replication_factor=N` (see [Distributed Sharding](distributed-sharding.md#creating-a-sharded-table)):

```sql
CREATE TABLE orders (id BIGINT PRIMARY KEY, amount INT) SHARD KEY (id) SHARDS 4 REPLICAS 2;
```

### Changing the replication factor online

The replication factor of an existing table can be raised or lowered at runtime through the admin REST API (`AdminHttpServer`):

```bash
POST /admin/tables/{name}/replicas
Content-Type: application/json
{"replicationFactor": 2}
```

Raising RF triggers copying the shard to additional Data Nodes; lowering it drops surplus replicas. Confirm with `GET /admin/tables/{name}` (`"replicationFactor"`). The per-shard copy/replicate RPC timeout during this and other repair/balancing operations is `neorunbase.shard.balancing.rpc.timeout.ms` / `neorunbase.shard.repair.rpc.timeout.ms` (default 60000; `neorunbase.properties`, section 29).

## Read Placement & Failover

Writes are replicated **synchronously**: a DML fans out to every replica of the shard and waits for all of them before returning, and fails if any replica does not apply it. A live replica therefore holds the same rows — and the same secondary-index column families, HNSW vector sidecars and Lucene full-text indexes — as the primary.

`neorunbase.query.read.replica.selection` decides which of those copies answers a read:

| Value | Behaviour |
| --- | --- |
| `primary` (default) | Reads go to the shard's primary. The other replicas are used only as failover. Preserves the strict read-your-writes path. |
| `round_robin` | Shards are spread over their replicas, so the index copies replication already keeps resident on every replica also serve queries instead of sitting idle. Raises read throughput and search parallelism. |

`round_robin` is opt-in because a replica that restarted and has not finished repair can lag behind the primary; under `primary` a read never lands on one.

**Failover applies under both settings.** A read that fails on one replica is retried on the next — a shard read no longer fails outright just because one Data Node is unreachable. This covers:

- ordinary `SELECT` shard reads (retried down the replica list in order),
- full-text (BM25) and vector (ANN) scatter search — the failed node's shards are regrouped onto their surviving replicas and retried once,
- CSR graph neighbour expansion and distributed PageRank, which take whichever replica the router puts first.

A `SELECT` is read-only, so retrying it on another replica is always safe. Only one re-route round is attempted: the replacement nodes are different replicas, so a second failure means the shard has no healthy copy and the query fails with the original error.

## Automatic Failure Detection

NeorunBase continuously monitors the health of all Data Nodes. When a Data Node becomes unavailable, the system automatically detects the failure and initiates recovery actions.

## Automatic Shard Repair

When a Data Node failure is detected, NeorunBase automatically replicates the affected shards from surviving replicas to healthy Data Nodes, restoring the desired replication factor without manual intervention. A failed node must stay unreachable for `neorunbase.shard.repair.grace.period.ms` (default 600000 = 10 min; `neorunbase.properties`, section 29) before its shards are confirmed lost and repair is triggered — this avoids reacting to transient network blips.

NeorunBase has three complementary repair paths:

- **Whole-node shard repair** (`ShardRepairService`) — recovers from full Data Node loss after the grace period; also operator-triggerable via `POST /admin/maintenance/repair/trigger`.
- **Disk repair** (`DiskRepairService`) — re-replicates shards off a failed/full disk on an otherwise healthy node. Off by default (`neorunbase.disk.repair.enabled=false`); see `neorunbase.properties`, section 19.
- **Reactive shard repair** — a continuous, divergence-driven worker that detects replicas that have silently drifted apart, without waiting for a node to drop. See [Reactive Shard Repair](reactive-shard-repair.md).

## Shard Rebalancing

When Data Nodes are added to or removed from the cluster, NeorunBase automatically rebalances shards across the cluster to ensure even data distribution and optimal resource utilization.

## Coordinator High Availability

Multiple Coordinators can run simultaneously. NeorunBase uses ZooKeeper-based leader election (Curator `LeaderSelector`) to designate a primary Coordinator for metadata management, while all Coordinators can serve client queries. If the leader Coordinator fails, a new leader is automatically elected.

Leader election is **sticky** by default (`neorunbase.cluster.election.sticky.enabled=true`; `neorunbase.properties`, section 22c): the previously-elected leader's node id is persisted to ZooKeeper, and on startup non-incumbent nodes wait `neorunbase.cluster.election.deference.window.ms` (default 3000) before joining the election queue, so a restarting incumbent reclaims leadership without a leadership toggle — avoiding a double-init of leader-only services (metrics collection, the repair workers, the site-replication streamer). Setting `sticky.enabled=false` falls back to first-lock-wins.

On every startup, non-leader coordinators and data nodes perform a mandatory **bootstrap fetch** — they pull authoritative KMS / metadata / IAM snapshots from the leader and overwrite their local RocksDB before signaling ready in ZooKeeper, so a node never serves stale state. If retries are exhausted the process exits with code 1 so the supervisor restarts it (`neorunbase.properties`, section 22b).
