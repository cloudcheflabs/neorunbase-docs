# Distributed Transactions

NeorunBase supports ACID transactions across multiple shards, ensuring data consistency in a distributed environment.

## ACID Compliance

NeorunBase provides full ACID guarantees for transactions:

- **Atomicity**: All operations within a transaction either succeed together or are rolled back entirely.
- **Consistency**: The database transitions from one valid state to another after each transaction.
- **Isolation**: Concurrent transactions do not interfere with each other.
- **Durability**: Once a transaction is committed, the changes are permanently stored.

## Cross-Shard Transactions

When a transaction involves data on multiple shards, NeorunBase coordinates the commit across all participating shards using a two-phase commit protocol:

1. **Prepare phase**: All participating shards prepare to commit and confirm readiness.
2. **Commit phase**: Once all shards confirm, the transaction is committed on all shards simultaneously.

If any shard fails to prepare, the entire transaction is rolled back.

## Transaction Support

NeorunBase supports standard SQL transaction control statements:

- `BEGIN` — Start a new transaction
- `COMMIT` — Commit the current transaction
- `ROLLBACK` — Roll back the current transaction

## Write-Ahead Log

All write operations are recorded in a Write-Ahead Log (WAL) before being applied to the storage engine. This ensures durability even in the event of a crash — uncommitted transactions can be recovered from the WAL upon restart.
