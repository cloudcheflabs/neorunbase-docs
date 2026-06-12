# Per-Table Statistics

NeorunBase 1.0.0 exposes per-table row counts and approximate storage size via the admin REST API, suitable for fast polling from observability pipelines (Prometheus exporters, Grafana, custom dashboards).

```bash
GET /admin/tables/{tableName}/stats
```

```json
{
  "tableName": "public.counter_test",
  "fullName": "public.counter_test",
  "numShards": 16,
  "replicationFactor": 2,
  "totalRowCount": 5,
  "approximateSizeBytes": 1043174
}
```

## Where the numbers come from

The handler scatters one `SHARD_INFO_REQ` per data node that hosts any replica of the table, aggregates the response from the **primary** replica of each shard (so each row is counted once, not once per replica), and sums:

- `totalRowCount` — sum of `shards.<shardId>.cfRowCounts.<fullTableName>` across all shards. Each value is RocksDB's `rocksdb.estimate-num-keys` for the column family, which is exact for fresh tables and within a few percent for tables with significant compaction history.
- `approximateSizeBytes` — sum of `shards.<shardId>.sizeBytes` across all shards. This is the on-disk size of the RocksDB column families that back the shard's data, including any sidecar segments for vector / FTS / CSR indexes that live in the same shard directory. It is an upper bound on the *encrypted* on-disk footprint, not the logical row size.

## Why two endpoints

There are two related endpoints:

| Endpoint | Returns | When to use |
|---|---|---|
| `GET /admin/tables/{name}/stats` | Compact totals only | Fast polling, monitoring exporter. One JSON object per poll regardless of shard count. |
| `GET /admin/tables/{name}/shards` | Per-shard breakdown including replica node IDs and per-shard rowCount | Operator forensics — "which shard is bigger than expected?", "which data nodes does this table live on?", "is my reshard complete?" |

The per-shard endpoint always existed; the totals-only `/stats` is the new 1.0.0 surface and is a thin wrapper that avoids returning a list of N×RF entries when all you wanted was a single number to graph.

## Caveats and accuracy

- **The primary replica is authoritative for the row count.** Replicas can lag the primary by replication latency. If a replica has diverged beyond the [reactive-repair threshold](reactive-shard-repair.md), the `/stats` endpoint still reports the primary's view (because that's the authoritative one) — the divergent replica is what gets repaired in the background.
- **Tombstones aren't subtracted.** RocksDB's `rocksdb.estimate-num-keys` counts entries including tombstones until compaction reclaims them. For DELETE-heavy workloads, the reported count can briefly overshoot the user-visible row count by a few percent until the next major compaction.
- **The size is approximate.** It's the sum of `RocksDBShard.getApproximateSize()`, which itself comes from RocksDB's level-0/level-N file size summaries. Useful for trending, not for billing.
- **Empty / freshly-created tables.** A CREATE TABLE that hasn't received any DML returns `totalRowCount=0, approximateSizeBytes>0` — the non-zero size is the column family metadata overhead, ~50 KiB per shard.

## Integration patterns

### Prometheus

A simple exporter pattern, run on the coordinator host:

```bash
#!/usr/bin/env bash
# Run every 30 seconds; output Prometheus exposition format.
TOKEN=$(cat /etc/neorunbase/admin-token)
for table in $(curl -sS -H "Authorization: Bearer $TOKEN" \
  "http://localhost:8080/admin/tables" | jq -r '.[]'); do
  stats=$(curl -sS -H "Authorization: Bearer $TOKEN" \
    "http://localhost:8080/admin/tables/${table}/stats")
  rows=$(echo "$stats" | jq '.totalRowCount')
  bytes=$(echo "$stats" | jq '.approximateSizeBytes')
  echo "neorunbase_table_rows{table=\"${table}\"} ${rows}"
  echo "neorunbase_table_bytes{table=\"${table}\"} ${bytes}"
done
```

Drop the output behind nginx and point Prometheus at it. The endpoint scales linearly with `(numShards × replicationFactor)` because it has to probe every data node holding a replica — for a 16-shard table at RF=2 on a 6-node cluster, that's at most 6 probes (cached per-node within one handler call), each <5ms. A polling cadence of 15–30 seconds is realistic.

### Self-service "is my table the biggest?"

```bash
TOKEN=...
for t in tableA tableB tableC; do
  bytes=$(curl -sS -H "Authorization: Bearer $TOKEN" \
    "http://coord:8080/admin/tables/${t}/stats" | jq '.approximateSizeBytes')
  printf "%-20s %s\n" "$t" "$bytes"
done | sort -k2 -n -r
```

### Billing / chargeback

Treat `approximateSizeBytes` as a leading indicator with a ~10% safety margin. For actual billing-grade numbers, snapshot the underlying RocksDB SST files and use the storage-encrypted size directly.

## See also

- [Topology-Aware Shard Placement](topology-aware-placement.md) — where these shards live in the first place.
- [Cluster Operations](../operations/operations.md) — disk repair and the broader observability surface.
- [Admin UI](admin-ui.md) — the visual representation of the same numbers.
