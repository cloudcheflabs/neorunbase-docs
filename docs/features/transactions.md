# Distributed Transactions

NeorunBase supports ACID transactions across multiple shards, ensuring data consistency in a distributed environment.

## ACID Compliance

NeorunBase provides full ACID guarantees for transactions:

- **Atomicity**: All operations within a transaction either succeed together or are rolled back entirely.
- **Consistency**: The database transitions from one valid state to another after each transaction.
- **Isolation**: Concurrent transactions do not interfere with each other.
- **Durability**: Once a transaction is committed, the changes are permanently stored.

## How a transaction executes

NeorunBase uses a **buffer-then-commit** model implemented in the pg-wire connection layer (`PgConnection.java`) and the coordinator (`QueryExecutor.executeTransaction`):

1. `BEGIN` (or `START TRANSACTION`) puts the connection into `IN_TRANSACTION` state and opens an empty statement buffer. The connection reports transaction status `T` to the client.
2. Each subsequent DML statement is **buffered on the connection, not executed immediately** — nothing is written to any shard yet.
3. `COMMIT` hands the whole buffer to `executeTransaction`, which parses and groups the buffered statements by target shard and runs a **two-phase commit** across all participating shards (see below). On success the connection returns to idle status `I`.
4. `ROLLBACK` simply discards the buffer without executing anything, so buffered rows never reach the shards.

Because statements are buffered until `COMMIT`, an aborted or rolled-back transaction has no side effects on the shards.

## Cross-Shard Transactions (two-phase commit)

At `COMMIT`, `executeTransaction` groups the buffered DML payloads by shard and coordinates an atomic commit across every participating shard **and all of its replicas** using two-phase commit with a per-transaction UUID:

1. **Prepare phase** — `PREPARE_BATCH_DML` is sent to every replica of every touched shard. Each shard stages the batch (pending tombstones + staged rows) in its WAL and confirms readiness.
2. **Commit phase** — once all shards have prepared, `COMMIT_PREPARED_TXN` is sent to every replica to make the staged writes durable and visible.

If any shard fails to prepare, the coordinator sends `ABORT_PREPARED_TXN` to every already-prepared shard and the whole transaction is aborted; the client receives an error (SQLSTATE `40001`). Because the prepare/commit fan-out includes **all replicas** of each shard, replication of a transaction's writes is synchronous with the commit.

The same 2PC machinery backs multi-shard bulk operations outside interactive transactions — `CREATE TABLE AS SELECT` (CTAS) and distributed `MERGE` each PREPARE/COMMIT across their target shards for atomicity.

## Transaction Support

NeorunBase supports standard SQL transaction control statements over a single connection/session:

- `BEGIN` / `START TRANSACTION` — Start a new transaction (opens the statement buffer)
- `COMMIT` — Execute the buffered statements via 2PC
- `ROLLBACK` — Discard the buffered statements

Notes and current limitations (from `QueryExecutor.executeTransaction` and `tests/test-transaction.sh`):

- Only **DML** — `INSERT`, `UPDATE`, `DELETE` — may appear inside a transaction. Any other buffered operation raises `Unsupported operation in transaction`.
- **DDL is rejected inside a transaction** (e.g. `CREATE TABLE` between `BEGIN` and `COMMIT` returns a "not allowed" error). Run DDL in autocommit mode.
- Transactions are **per-connection**; there is no cross-connection distributed transaction coordinator exposed to clients.

## Write-Ahead Log

All write operations are recorded in a Write-Ahead Log (WAL) before being applied to the storage engine. This ensures durability even in the event of a crash — uncommitted transactions can be recovered from the WAL upon restart.
