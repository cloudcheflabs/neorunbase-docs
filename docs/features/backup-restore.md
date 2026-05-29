# Backup & Restore

NeorunBase backs up its critical state to S3-compatible object storage on a schedule. Backups are incremental, opt-in, and can be restored cluster-wide from the admin UI.

## What Gets Backed Up

Every backup run captures both halves of the cluster's state:

- **Coordinator state** — KMS keys, IAM users, and cluster metadata.
- **Data-node state** — every shard's data and write-ahead log files.

A single backup run produces a coordinated snapshot — coordinator metadata and data-node shards share the same backup ID, so restore reassembles a consistent point in time.

## Configuration

The `S3 Backup` page in the admin UI exposes the full configuration:

- **Enabled** — master switch (default: off).
- **S3 endpoint, region, bucket, prefix** — the destination bucket.
- **Access key / secret key** — long-lived static credentials.
- **Path-style addressing** — required for ShannonStore / MinIO.
- **Interval (minutes)** — how often the scheduled backup runs.
- **Cron** — optional 5-field UNIX cron expression for wall-clock schedules (e.g. `0 2 * * *` daily at 02:00). Coexists with the interval rule; whichever comes due first fires the backup.
- **Retention (days)** — how long the visible backup history is kept (default 30 days).

The same page exposes a **Test Connection** button (verifies the bucket is reachable) and a **Backup now** button (runs an immediate backup).

### Cron vs. Interval

The two automatic rules are independent and share the same coordinator tick loop:

- **Interval** is a relative "at least N minutes since the last successful backup" timer. Use it as a safety-net cadence.
- **Cron** is wall-clock based. Use it for predictable schedules like nightly or weekly runs.

The cron expression is evaluated in the leader coordinator's local time zone. The first sighting after the cron is set, or after a leader handoff, **arms from "now"** — NeorunBase does not back-fire missed cron times on startup. An invalid cron is rejected with HTTP 400 at config-save time rather than being persisted and silently skipped.

## Incremental by Design

NeorunBase backs up files only when their content has actually changed:

- **First run** uploads everything.
- **Subsequent runs** upload only files that differ from prior backups; unchanged files are reused — a re-run with no cluster changes uploads zero bytes.

Each backup writes a manifest that lists every file alive at backup time, so retention or restore can refer to any single backup ID without re-uploading shared files.

## History Retention

Past backups are listed in the admin UI and aged out by the **Retention (days)** policy (default 30 days). When an entry passes retention it disappears from the visible history; the underlying objects remain in S3 unless an S3 lifecycle rule removes them. This keeps backup deletes deliberate and reversible — operators decide when to actually reclaim S3 storage.

## Restore

Restore is initiated from the admin UI by selecting a backup ID and confirming. The flow is **safe by default**:

1. Each node downloads the files referenced by the chosen backup into a *staging directory* alongside its live data — the running cluster keeps serving traffic and is never touched in place.
2. The admin UI confirms the staging is complete and prompts the operator to restart the containers.
3. On restart, every node automatically swaps the staged tree into place **before** opening its databases. The previous live data is moved aside (kept on disk as `<dir>.pre-restore-<id>`, not deleted) so the restore is reversible.

A failed or partial swap leaves the staging directory intact, so the next restart retries instead of silently considering the restore "applied." Operators can clean up the `.pre-restore-<id>` directories once they're satisfied with the restored state.

## Specialised Indexes (HNSW, FTS) and Backup Safety

Vector (HNSW) and full-text (FTS) indexes live as encrypted sidecars in each shard's directory, so they ride along with the shard tree on every backup — no separate path. Two safeguards keep them consistent with the row store:

- **Periodic flush** — both index types run a background scheduler that flushes the in-memory authoritative state to its sidecar at a configurable interval (default 30 s for ANN and FTS each).
- **Pre-backup flush hook** — every backup run triggers an explicit flush on each Data Node *before* any file is uploaded, so the backup captures the live in-memory state, not the last-flushed snapshot. The window between a write and a backup that doesn't include that write is therefore zero.

If a sidecar is missing or stale on a restored cluster, the shard rebuilds it from authoritative RocksDB rows on the next mutation (HNSW path) — so a restore that loses sidecar files alone still converges. The FTS rebuild-on-mutation path is on the roadmap; until it lands, FTS sidecars depend on the standard backup → restore round-trip preserving the sidecar files (which it does).

## Limitations

- Restore requires a coordinated restart of all coordinator and data-node containers — there is no online (in-place) restore.
- Backups capture cluster state only; native PostgreSQL `pg_dump`-style logical exports are out of scope.
