# Topology-Aware Shard Placement

NeorunBase 1.0.0 places shard replicas across data nodes with awareness of the **fault domains** the data nodes live in — zone, rack, host — and the **capacity** each data node reports. A shard's RF replicas land on as-distinct-as-possible domains, so a rack power loss or a host hardware failure removes at most one copy of any shard.

For unlabeled clusters (the default) the algorithm degenerates to capacity-weighted round-robin, which is backwards-compatible with the legacy `(shard + replicaIndex) mod nodeCount` placement.

## Labels

Each data node reports three optional labels at registration time:

| Label | JVM property | Environment variable | Meaning |
|---|---|---|---|
| `hostId` | `-Dneorunbase.host.id` | `NEORUNBASE_HOST_ID` | Physical server identity. Distinct on dual-socket nodes; identical across processes sharing the same chassis. |
| `rackId` | `-Dneorunbase.rack.id` | `NEORUNBASE_RACK_ID` | The rack the host sits in. Usually one PDU + one ToR switch. |
| `zoneId` | `-Dneorunbase.zone.id` | `NEORUNBASE_ZONE_ID` | Datacenter availability zone — for AWS, an AZ; for self-hosted, a power-domain. |

System property takes precedence over the environment variable. Setting only `hostId` is valid — the strategy falls back: missing `zoneId` falls through to `rackId`, missing `rackId` falls through to `hostId`, missing `hostId` falls through to `nodeId`. This lets you populate labels gradually as your topology becomes legible.

The labels are persisted in ZooKeeper as part of the data node's ephemeral registration and survive heartbeats (the disk-info refresh path preserves them). They are surfaced on:

- `GET /admin/nodes/datanodes` — fields `hostId`, `rackId`, `zoneId`.
- The admin UI Topology page — printed under each data node's host:port.

## The 6-pass cascade

`com.cloudcheflabs.neorunbase.cluster.ShardPlacementStrategy` runs once per shard and assigns `RF` replicas:

```
fault-domain hierarchy (strongest distinctness first):

  zone  →  rack  →  host  →  node  →  desperation (cycle reuse)

Pass 1 — zone-distinct:  rank candidate nodes, accept any whose zone hasn't been picked.
Pass 2 — rack-distinct:  among remaining nodes, accept any whose rack hasn't been picked.
Pass 3 — host-distinct:  among remaining nodes, accept any whose host hasn't been picked.
Pass 4 — node-distinct:  accept any remaining node (this is the legacy floor).
Pass 5 — cycle reuse:    only if RF > distinct-node count, log a warning + repeat from pass 1.
```

Within each pass, candidates are scored by capacity — sum of `availableBytes` across the data node's disks. Nodes that haven't reported disk info get the median weight so they aren't starved during bootstrap. The chosen RF set is then sorted by descending capacity so the largest-capacity replica is designated **primary** — this is weighted-HRW-lite, evaluated at assignment time instead of per-request.

A label being null in one node doesn't break the cascade — that node's "zone" effectively equals its rack/host/nodeId via the fallback. Heterogeneously-labeled clusters (partial topology rollout) get the *best available* distinctness without explicit handling.

## What you observe

### Data-node startup log

```
Registered data node: dn-7100 at dn-host:7100 with 4 disks ready=false
    topology=(host=host-a1 rack=rack-1 zone=zone-east)
Data node dn-7100 registered with topology labels: host=host-a1 rack=rack-1 zone=zone-east
```

### Admin REST surface

```bash
curl -H "Authorization: Bearer $TOKEN" http://coord:8080/admin/nodes/datanodes
[
  {
    "nodeId": "dn-7100",
    "host": "dn-host",
    "internalPort": 7100,
    "hostId": "host-a1",
    "rackId": "rack-1",
    "zoneId": "zone-east",
    "disks": [...]
  },
  ...
]
```

### Admin UI

The Topology page (`/nodes`) prints `zone=… rack=… host=…` under each data node's host:port line.

## Operator workflow

### Initial setup

Put the labels in your systemd EnvironmentFile / Kubernetes pod env / docker compose env at every data-node host:

```bash
NEORUNBASE_HOST_ID=node-prod-a-1
NEORUNBASE_RACK_ID=rack-a-3
NEORUNBASE_ZONE_ID=ap-northeast-2a
```

The labels are read once at data-node startup, so changing them later requires a data-node restart. The labels are not part of the shard catalog — they only affect *future* placement decisions. Restarting after a label change doesn't move existing shards.

### Onboarding a new rack

1. Bring the new data nodes up with their new `rackId` / `zoneId`.
2. Confirm `GET /admin/nodes/datanodes` lists them with labels.
3. **New tables** automatically use the topology-aware placement. **Existing tables** are unaffected until you trigger a per-table reshard via `POST /admin/tables/{name}/reshard`, which re-runs the strategy with the current data-node set.

### Heterogeneous capacity

If your cluster has a mix of small and large data nodes, the strategy already accounts for it via the capacity-weighted primary selection. You don't need to do anything — larger nodes will own a proportionally larger share of primaries. The replication factor remains symmetric (each replica is a copy), but read traffic spreads weighted-HRW-style toward the larger nodes.

## Configuration reference

| JVM property | Environment variable | Default | Description |
|---|---|---|---|
| `neorunbase.host.id` | `NEORUNBASE_HOST_ID` | unset | Distinct hostname for host-distinct placement. |
| `neorunbase.rack.id` | `NEORUNBASE_RACK_ID` | unset | Rack identity for rack-distinct placement. |
| `neorunbase.zone.id` | `NEORUNBASE_ZONE_ID` | unset | Availability zone for zone-distinct placement. |

There are no other knobs — the algorithm and pass priority are deliberately fixed so behaviour is predictable.

## See also

- [Replication & High Availability](replication-ha.md) — intra-cluster shard replication.
- [Site Replication (DR)](site-replication.md) — cross-site disaster recovery streaming.
- [Cluster Operations](../operations/operations.md) — bootstrap order and operator runbook.
