# Distributed Sharding

NeorunBase distributes table data across multiple Data Nodes through hash-based sharding, providing horizontal scalability for both reads and writes.

## How Sharding Works

When a table is created, NeorunBase assigns a configurable number of shards and distributes them across the available Data Nodes. Each row is assigned to a shard based on a hash of the designated shard key column, ensuring even data distribution. When `CREATE TABLE` does not specify a shard count, the cluster default `neorunbase.default.num.shards` (16) is used (`neorunbase.properties`, section "Default Settings").

## Creating a Sharded Table

The shard count, shard key, and replication factor are specified as table options in `CREATE TABLE`. NeorunBase's `DdlParser` (`neorunbase-sql/.../DdlParser.java`) accepts two equivalent syntaxes:

```sql
-- Positional option syntax
CREATE TABLE orders (
    id BIGINT PRIMARY KEY,
    customer TEXT,
    amount INT
) SHARD KEY (id) SHARDS 4 REPLICAS 2;

-- WITH-clause syntax (equivalent)
CREATE TABLE orders (
    id BIGINT PRIMARY KEY,
    customer TEXT,
    amount INT
) WITH (num_shards=4, replication_factor=2, shard_key=id);
```

- `SHARD KEY (col)` / `shard_key=col` — the column hashed to place each row. If omitted, the table's primary key column is used.
- `SHARDS N` / `num_shards=N` — number of shards. Defaults to `neorunbase.default.num.shards` (16).
- `REPLICAS N` / `replication_factor=N` — copies of each shard. Defaults to `neorunbase.default.replication.factor` (2). See [Replication & High Availability](replication-ha.md).

If a table is created without any `PRIMARY KEY`, NeorunBase auto-generates a hidden `_rowid BIGINT` primary key and shards on it (`DdlParser.java` — "No PRIMARY KEY specified … auto-generating _rowid column").

## Shard Key

The shard key is a column specified at table creation time. NeorunBase uses the shard key value to determine which shard a row belongs to (`ShardKeyUtils.computeShardId`). Queries that include the shard key in their `WHERE` clause can be routed directly to the relevant shard(s), significantly improving query performance.

## Query Routing

NeorunBase's `ShardRouter` (`neorunbase-sql/.../ShardRouter.java`) inspects the query's `WHERE` clause and routes accordingly:

- **Point queries** (`shard_key = value`): routed to a single shard.
- **IN queries** (`shard_key IN (v1, v2, …)`): routed to only the shards those values hash to (the router hashes each value and unions the shard set).
- **Everything else** — including range predicates (`<`, `>`, `BETWEEN`) on the shard key, or a `WHERE` that doesn't constrain the shard key at all: scattered to **all** shards in parallel and results are merged. Because sharding is **hash-based**, the hash does not preserve value order, so range predicates on the shard key cannot be pruned to a subset of shards.

Once the target shards are chosen, [read placement](replication-ha.md#read-placement-failover) decides **which replica** of each shard answers, and retries on another replica if that one does not respond.

## Shard Pruning

The coordinator can keep per-column Bloom filters for shards (`ShardBloomCache`, keyed `table#column → shardId → BloomFilter`) and skip shards that definitely do not contain a queried value. It applies to **equality on an indexed column** only — not to range predicates, and not to columns without an index.

It is **disabled by default** (`neorunbase.query.shard.bloom.prune.enabled=false`), because the filter is built only from `INSERT`s that a given coordinator process handled and lives only in that process's heap:

- With two or more coordinators, each one knows only its own inserts, so it can answer "definitely absent" for a value another coordinator wrote, prune the shard that really holds it, and drop rows from the result.
- Rows arriving by any other route — Kafka ingest, restore, resharding, replica repair — are never recorded, with the same effect.
- Nothing survives a coordinator restart.

Enable it only for a single-coordinator deployment whose writes all arrive as SQL `INSERT`s; see [Shard Pruning by Bloom Filter](../configuration/configuration.md#shard-pruning-by-bloom-filter) for the sizing knobs. The sound placement for this filter is the data node, which owns the shard's index and sees every write regardless of origin — that is where it will move.

## Resharding

NeorunBase supports online resharding, allowing you to change the number of shards for a table. During resharding, data is transparently migrated to the new shard layout without downtime, and the table stays read-write.

Trigger a reshard with SQL:

```sql
ALTER TABLE orders SET SHARDS 8;
```

or via the admin REST API:

```bash
# Kick off a reshard
POST /admin/tables/{name}/reshard
# Poll progress (returns {"status":"idle"} when none is in progress)
GET  /admin/tables/{name}/reshard
```

Both are handled by `AdminHttpServer` (`/admin/tables/{name}/reshard`). The per-shard copy/replicate RPCs during a reshard are governed by `neorunbase.reshard.shard.operation.timeout.ms` (default 120000) and `neorunbase.reshard.ddl.operation.timeout.ms` (default 60000) — see `neorunbase.properties`, section 24. Confirm the new count with `GET /admin/tables/{name}` (`"numShards"`).
