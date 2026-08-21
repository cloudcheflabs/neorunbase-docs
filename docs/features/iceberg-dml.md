# Row-Level DML on Iceberg Tables

`INSERT`, `UPDATE`, `DELETE` and `MERGE INTO` run directly against a registered Iceberg table, in the same SQL session as everything else:

```sql
INSERT INTO ice.public.events (id, name, val) VALUES (201, 'a', 1.5), (202, 'b', 2.5);

DELETE FROM ice.public.events WHERE id BETWEEN 21 AND 40;

UPDATE ice.public.events SET name = 'updated' WHERE id BETWEEN 61 AND 70;

MERGE INTO ice.public.events AS t USING staging AS s ON t.id = s.id
  WHEN MATCHED THEN UPDATE SET t.name = s.name
  WHEN NOT MATCHED THEN INSERT (id, name, val) VALUES (s.id, s.name, s.val);
```

No copy into a native table first, no external engine.

## Merge-on-read, never a file rewrite

`UPDATE`, `DELETE` and `MERGE INTO` do **not** rewrite data files. The rows that matched are recorded by position — `(data file, row ordinal)` — and written as **delete files**; `UPDATE` and `MERGE` additionally append the updated or inserted copies as new data files. Everything lands in a single Iceberg `RowDelta` commit, so a reader sees the statement as one atomic snapshot change.

Which delete file gets written is decided by the table's format version:

| Format version | Delete file written | Layout |
| --- | --- | --- |
| **2** | Parquet **position-delete** file, one per partition bundle | rows of `(file_path, pos)`, sorted by file then position |
| **3** | Puffin **deletion vector**, one per data file | roaring bitmap of deleted positions in a `deletion-vector-v1` blob |

Both are the standard merge-on-read forms, so any engine that reads Iceberg reads the result — see [Cross-engine verification](#cross-engine-verification) for what is actually tested, including a Trino version boundary that matters in practice.

!!! note "Deletes from the CDC sync are a different shape"
    The native→Iceberg [CDC sync](iceberg-integration.md) writes **equality deletes**, because it upserts by primary key and an equality delete is the natural expression of "this key is superseded". Equality deletes stay valid in v3. Row-level DML is positional, so it uses position deletes / deletion vectors instead. A table can carry both.

## One deletion vector per data file

The v3 spec allows a data file **at most one** deletion vector. A second `DELETE` touching the same data file therefore cannot simply add another vector — NeorunBase:

1. reads the existing vector,
2. merges the new positions into it,
3. writes the merged vector, and
4. removes the superseded delete file in the same commit.

So two deletes of 20 and 10 rows against one data file leave exactly one vector of cardinality 30, not two vectors.

The `RowDelta` is validated **from the snapshot the statement planned against** (`validateFromSnapshot`). This matters: without that scope, the validation window is the table's entire history and the very vector this statement is deliberately superseding is reported as *"Found concurrently added DV"*, refusing the commit. Scoped correctly, a vector another writer adds for the same data file **after** our snapshot still conflicts — which is what should happen.

## `MERGE INTO`

`MERGE INTO` reads its target, evaluates the `ON` condition against each source row, and commits one
`RowDelta`: **position deletes for the target rows that matched**, plus a data file holding the merged
(`WHEN MATCHED THEN UPDATE`) and inserted (`WHEN NOT MATCHED THEN INSERT`) rows.

Deleting by position rather than by key is what makes the statement mean what it says:

- **It removes exactly the rows that matched.** A key-based delete would also remove any other row that
  happens to share that key, which `MERGE` never asked for.
- **It works whatever the table is partitioned by.** A delete file applies only to data files in its own
  partition, and the position deletes are grouped by the partition of the file each match came from — so a
  target row written long before the statement runs is still reached.
- **It needs no identifier fields.** A table that declares none can still be a `MERGE` target.

Two further behaviours are worth knowing:

- The target is read from **freshly loaded table metadata**, never from the serving metadata cache, so a
  `MERGE` issued immediately after another statement cannot plan against a snapshot that statement already
  superseded.
- If the same target row is matched by two source rows, the row is deleted once and both merged rows are
  written. `MERGE` does not silently collapse a cardinality violation into one row.
- If a data file the statement planned against is gone by commit time (another writer rewrote it), the
  statement fails with *"the table changed concurrently — retry the statement"* rather than deleting
  positions that now address different rows.

## What matching costs

The matched positions come from reading the candidate data files, which means a `DELETE`/`UPDATE` reads the rows it touches:

- The `WHERE` is pushed down first, so files whose min/max bounds cannot match are never opened (same pruning as the [scan path](iceberg-serving.md#scan-path-pruning-for-everything-else)).
- Rows already removed by an earlier delete file — equality, position, or a deletion vector — are skipped, so a repeated `DELETE` reports 0 rather than "matching" rows that are not there.
- Data-file contents are served from the shared per-file cache, so a `DELETE` immediately after a scan usually re-reads nothing.

## Write-Audit-Publish

Row-level DML honours the [WAP branch](iceberg-wap.md) exactly like the sync and CTAS do: when a WAP branch is configured for the catalog or table, the `RowDelta` is committed to that branch instead of `main`, so deletes and updates can be staged and audited before publication.

## `count(*)` after a delete

`SELECT count(*)` with no `WHERE` is answered from metadata whenever metadata can answer it *exactly*:

| Table state | How `count(*)` is answered |
| --- | --- |
| No delete files | Sum of each data file's `record_count`. No data read. |
| **v3 deletion vectors only** | `sum(data.record_count) − sum(dv.record_count)`. **Still exact**, because a vector is 1:1 with its data file, so no position can be subtracted twice. No data read. |
| v2 position deletes | Full scan. Two position-delete files may legally list the same `(file, pos)`, so subtracting their record counts could over-delete. |
| Equality deletes | Full scan. They match by key; how many live rows a delete file removes is not in the metadata at all. |

The snapshot summary's `total-records` is deliberately not used — it counts rows ever *added* and subtracts nothing that deletes hide.

The practical effect: on v3, a serving table keeps answering `count(*)` in constant time after deletes, which is exactly where a full scan is most wasteful. On v2 the same query scans, and that is the correct trade.

## Cross-engine verification

The delete files NeorunBase writes are checked against a foreign reader rather than assumed to be conformant. `tests/test-iceberg-write-crossengine-e2e.sh` runs, for each format version: 100 rows synced, then `DELETE` 20 / `UPDATE` 10 / `INSERT` 2, then asserts that the delete-file **kind** on disk is right (v2 must produce a position-delete parquet and no `.puffin`; v3 the opposite) and that NeorunBase and Trino report the identical 82 rows, the same empty band, the same 10 updated rows and the same inserted rows.

!!! warning "Trino must be new enough for v3"
    OSS **Trino 479** — the version [Chango](https://www.cloudchef-labs.com) currently ships — supports Iceberg **v1 and v2 only**. It rejects `format_version = 3` with *"format_version must be between 1 and 2"* and fails any read of a deletion vector with *"Unexpected PUFFIN file format"*. Trino picked v3 up around release 480/481; **483 reads these vectors correctly**. (Starburst Enterprise/Galaxy builds advertise v3 earlier — a different distribution.)

    So the e2e verifies v2 against Trino 479 and v3 against Trino 483 (`TRINO_VERSION_V2` / `TRINO_VERSION_V3`). If you need Trino interoperability today, keep tables on **format-version 2** — which is NeorunBase's default (`neorunbase.iceberg.format.version`).

The opposite direction — NeorunBase reading deletion vectors written by *another* engine — is covered by `tests/test-iceberg-v3-dv-crossengine-e2e.sh` using Ontul; see [Iceberg Integration](iceberg-integration.md#format-version-2-and-3-deletion-vectors).

## Requirements and limits

- The table must be registered as a [catalog](catalogs.md) and addressed as `<catalog>.<namespace>.<table>`.
- `INSERT INTO` requires an explicit column list.
- `UPDATE` assigns literal values. The updated rows are re-appended, and because a synced table is
  partitioned by a [bucket of its primary key](iceberg-integration.md#built-in-time-column-partitioning)
  they land in the same partition as the rows they replace. (On a table created before that layout —
  partitioned by `days(_neorun_synced_at)` — they move into the current day's partition instead, which is
  why such tables have to be dropped and re-synced.)
- Compaction of accumulated delete files is not automatic — see [What Is Not (Yet) Supported](iceberg-integration.md#what-is-not-yet-supported).
