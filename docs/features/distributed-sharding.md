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

A scatter query fans out to every target shard. NeorunBase does **not** try to skip shards using a coordinator-side summary of their contents, because no such summary can be made sound:

- A negative answer ("this shard definitely has no such value") is only safe if the coordinator's copy is never missing a value that exists. Any staleness at all turns into a pruned shard and a **silently missing row**.
- Keeping it fresh would require every write — from any coordinator, from Kafka ingest, from restore, from resharding, from replica repair — to synchronously reach every coordinator's copy. That is more coordination than the scatter it was meant to avoid.

A coordinator-local Bloom filter of this shape did exist and was removed: it was populated only from the `INSERT`s that one coordinator process handled, so a second coordinator would answer "definitely absent" for a value the first had written.

What actually makes a non-matching shard cheap is that the shard itself answers fast. An equality on an indexed column becomes a single RocksDB seek into that index's column family; a miss returns an empty result without reading a row or touching the heap. The cost that remains is the RPC, and that is the part no correct coordinator-side filter can remove.

Predicates that *can* be narrowed before fan-out are handled by [Query Routing](#query-routing) above: an equality or `IN` on the shard key resolves to the exact shards that can hold those values.

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
