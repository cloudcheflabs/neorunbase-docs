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

Range and non-shard-key predicates can still benefit from [Shard Pruning](#shard-pruning) via Bloom filters where a per-column filter is available.

## Shard Pruning

The coordinator maintains per-column Bloom filter caches for shards (`ShardBloomCache`, `neorunbase-coordinator/.../ShardBloomCache.java` — keyed `table#column → shardId → BloomFilter`), allowing it to skip shards that definitely do not contain the queried values. This reduces unnecessary I/O and improves query latency.

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
