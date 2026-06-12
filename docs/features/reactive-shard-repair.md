# Reactive Shard Repair

NeorunBase 1.0.0 adds a **continuous, divergence-driven** repair worker that detects when a shard's replicas have drifted apart — without waiting for a full data node to drop from ZooKeeper.

This sits alongside the existing repair paths and fills a gap they don't cover:

| Path | Triggered by | What it catches |
|---|---|---|
| **Disk Repair** (`DiskRepairService`) | A disk on a known data node fails (`getAvailableBytes() == 0` after grace period) | Per-disk loss, single-disk-failure scenarios. |
| **Shard Repair** (`ShardRepairService` — manual trigger) | An operator clicks Trigger Repair or `POST /admin/maintenance/repair/trigger` | Full data-node loss after grace period. Operator-driven. |
| **Reactive Shard Repair** ← new in 1.0.0 | A scan finds a replica whose row count diverges from the high-water-mark replica beyond the configured threshold | Partial write failure, replica that silently fell behind, split-brain recovery without waiting for full node loss. |

## The detection algorithm

The worker runs on the elected **leader coordinator**. Followers carry the same code but their `leaderSupplier` returns `false`, so the scan loop and the drain loop both idle.

```
every scan.interval.seconds (default 60s):
  for each table:
    for each shard:
      if shard has only 1 replica, skip (nothing to compare against)
      reference_count := max(probe(replica.SHARD_INFO_REQ).cfRowCounts[table]
                             for replica in shard.replicas)
      reference_node  := the replica that holds reference_count
      for each replica != reference_node:
        if probe.cfRowCount < reference_count * (1 - divergence.fraction):
          enqueue (table, shardId, src=reference_node, dst=this replica)

drain loop:
  pull tasks off the queue;
  invoke ShardRepairService.copyShardToReplica(task)
  on success → totalRepaired++
  on failure → re-queue at the tail after backoff → totalFailed++
```

The probe re-uses the existing `SHARD_INFO_REQ` opcode — no new wire format. The receiver already returns per-CF row counts (`shards.<shardId>.cfRowCounts.<table>`), so the divergence check is one number subtraction per replica per scan.

The drain re-uses `ShardRepairService.copyShardToReplica`, which is the same primitive the manual trigger uses. So a reactively-detected divergence is repaired the same way an operator-triggered repair would be — by copying the SST + WAL state via `SHARD_COPY_REQ` / `SHARD_REPLICATE_REQ`.

## What you observe

### Status endpoint

```bash
curl -H "Authorization: Bearer $TOKEN" http://coord:8080/admin/maintenance/reactive-repair/status
{
  "enabled": true,
  "running": true,
  "leader": true,
  "lastScanTimestamp": 1781272157653,
  "queueDepth": 0,
  "totalDetected": 0,
  "totalRepaired": 0,
  "totalFailed": 0
}
```

- `lastScanTimestamp` advances every scan interval.
- `queueDepth` is positive when there are tasks pending repair.
- `totalDetected` / `totalRepaired` / `totalFailed` are cumulative counters since coordinator startup.

### Admin UI

The Shard Repair page (`/repair`) prints a "Reactive Shard Repair (continuous)" panel above the manual repair UI with the same counters in tiles.

### Logs

```
INFO  c.c.n.c.r.ReactiveShardRepairService - Reactive shard repair started (scan every 60s, divergence threshold 5%)
INFO  c.c.n.c.r.ReactiveShardRepairService - Reactive repair: public.t shard 3 replica dn-7102 diverged (47/53), enqueuing repair from dn-7100
INFO  c.c.n.c.r.ReactiveShardRepairService - Reactive repair: public.t shard 3 dn-7102 ← dn-7100 OK
```

## Configuration

| JVM property | Default | Description |
|---|---|---|
| `neorunbase.reactive.shard.repair.enabled` | `true` | Master switch. When `false`, neither the scan nor drain thread runs. |
| `neorunbase.reactive.shard.repair.scan.interval.seconds` | `60` | How often the scan walks every (table, shard, replica) tuple. Lower = faster reaction, higher network probe cost. Minimum 5 seconds. |
| `neorunbase.reactive.shard.repair.divergence.fraction` | `0.05` | A replica is enqueued for repair when its row count is below `referenceCount × (1 − divergence.fraction)`. The default 5% absorbs normal replication lag without false positives; lower it to 0.01 if your workload's lag is very tight. |

These are loaded by `ConfigLoader` at coordinator startup. Change in `conf/neorunbase.properties` and restart the coordinator.

## When does the reactive worker fire?

In a healthy cluster running normal writes, all replicas should stay within a few records of each other — well inside the 5% default tolerance. The worker fires when something goes wrong **without** the cluster also detecting the broken replica via ZK:

- A replica's data node hung mid-write (TCP stack still alive, application thread stuck).
- A replica's data node restarted and silently missed WAL apply for some batches.
- A network partition let writes succeed on majority replicas while one replica was unreachable; the partition has since healed but the replica is now stale.
- A bug introduced a silent write skip on one replica (this is what the algorithm protects against — a class of errors you'd otherwise only catch via `SELECT count(*)` discrepancies across replicas).

When the worker fires, the metric to alarm on is **`totalFailed` rate** — sustained failures usually mean the source replica is unreachable, the target's disk is full, or the SHARD_REPLICATE_REQ landed mid-IO. Investigate the most recent log lines around the failed task.

## Operator workflow

### Tuning for noisy networks

If your cluster runs across a noisy WAN where momentary lag triggers false positives, raise `divergence.fraction` (e.g. to 0.10 for 10%) or raise the scan interval (e.g. to 300s for 5 min). The worker is designed to be *eventually correct*, not immediately reactive — a 5-minute reaction time is still cheap operator vigilance compared to manual repair triggers.

### Disabling the worker

Set `neorunbase.reactive.shard.repair.enabled=false` in `conf/neorunbase.properties` and restart the coordinator. The status endpoint will then return `enabled:false, running:false`. The manual repair endpoints continue to work.

### Reading the per-task log line

```
Reactive repair: public.t shard 3 replica dn-7102 diverged (47/53), enqueuing repair from dn-7100
```

- `public.t` — the table whose replica diverged.
- `shard 3` — the shard ID.
- `dn-7102` — the replica that fell behind.
- `47/53` — `dn-7102` reports 47 rows in this shard for this table; the high-water-mark replica reports 53.
- `dn-7100` — the high-water-mark replica that will be used as the copy source.

## See also

- [Replication & High Availability](replication-ha.md) — intra-cluster shard replication and shard repair fundamentals.
- [Topology-Aware Shard Placement](topology-aware-placement.md) — where the divergent replica was placed in the first place.
- [Cluster Operations](../operations/operations.md) — disk repair, manual shard repair, and the broader maintenance runbook.
