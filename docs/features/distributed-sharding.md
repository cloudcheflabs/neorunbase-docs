# Distributed Sharding

NeorunBase distributes table data across multiple Data Nodes through hash-based sharding, providing horizontal scalability for both reads and writes.

## How Sharding Works

When a table is created, NeorunBase assigns a configurable number of shards and distributes them across the available Data Nodes. Each row is assigned to a shard based on a hash of the designated shard key column, ensuring even data distribution.

## Shard Key

The shard key is a column specified at table creation time. NeorunBase uses the shard key value to determine which shard a row belongs to. Queries that include the shard key in their `WHERE` clause can be routed directly to the relevant shard(s), significantly improving query performance.

## Query Routing

NeorunBase automatically routes queries to the appropriate shards:

- **Point queries**: When the shard key value is specified, the query is routed to a single shard.
- **Range queries**: Queries are routed only to the shards that may contain matching data.
- **Full scan queries**: When the shard key is not specified, the query is scattered to all shards in parallel and results are merged.

## Shard Pruning

NeorunBase maintains Bloom filter caches for shards, allowing it to skip shards that definitely do not contain the queried values. This reduces unnecessary I/O and improves query latency.

## Resharding

NeorunBase supports online resharding, allowing you to change the number of shards for a table. During resharding, data is transparently migrated to the new shard layout without downtime.
