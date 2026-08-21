# Iceberg Integration

NeorunBase syncs transactional tables to Apache Iceberg in the background and reads existing Iceberg tables directly from SQL, so downstream analytics engines (Spark, Trino, Hive, Flink) can work on the same data without an external ETL.

!!! note "Iceberg catalogs are first-class"
    An Iceberg connection is registered as a named [catalog](catalogs.md). You can register more than one (e.g. `iceberg-east`, `iceberg-west`); tables in each are addressed as `<catalog>.<namespace>.<table>`. This page uses `iceberg` as the example catalog name.

## Catalog — Apache Polaris

NeorunBase currently supports **Apache Polaris** as the Iceberg REST catalog. A catalog connection is configured from the admin UI, via `CREATE CATALOG` SQL, or seeded at startup from `neorunbase.iceberg.*`. Each catalog supplies:

- Polaris URI and OAuth token endpoint
- Catalog name (used as the Iceberg `prefix` and `warehouse` parameter)
- OAuth2 client ID / client secret, realm, scope
- S3 endpoint, access key, secret key, region

You can register more than one Iceberg catalog. See [Catalogs](catalogs.md) for the full create/alter/drop surface.

S3 access uses the **static access key / secret key** set above. Polaris's STS / vended-credentials path is intentionally not used — every read and write goes through the same long-lived credentials configured in the admin UI.

## Automatic Sync (CDC)

After Iceberg is enabled, NeorunBase automatically keeps every selected table mirrored in Iceberg:

- **First sync** writes the full table as a single Iceberg snapshot.
- **Subsequent syncs** are incremental — only changed rows since the last sync are written, as an atomic add-data + equality-delete commit.

Each table is synced by exactly one coordinator (the leader assigns it, often to a follower), so two coordinators never write the same Iceberg table concurrently.

**Where a synced table lands.** The Iceberg namespace is taken from the **schema of the source table**, not from the catalog's default namespace: `public.orders` is mirrored as `iceberg.public.orders`, and `sales.orders` as `iceberg.sales.orders`. The namespace is created if it does not exist. `default-namespace` (`neorunbase.iceberg.default.namespace`) applies only to a source table addressed with no schema at all.

A per-shard **high-water mark** (the source changelog position) is stamped into the incremental commit's **snapshot summary**, atomically with the synced rows. On a coordinator restart the sync reads the high-water mark back from that snapshot and resumes exactly where it left off — it does not re-read the changelog, so there is no double-sync. Because the watermark lives in the same snapshot as the data (rather than in a separate write), the data and the sync position always advance together.

## Built-in Time Column + Partitioning

Every Iceberg table created by NeorunBase carries an extra column **`_neorun_synced_at`** (`timestamptz`). The column is set automatically — operators don't read or write to it.

Tables are also automatically:

- **Partitioned** by `bucket(<primary key>, N)` — N is [`neorunbase.iceberg.cdc.partition.buckets`](../configuration/configuration.md) (default 8).
- **Sorted** by `[primary key, _neorun_synced_at]` — key ranges stay clustered inside a bucket.

The partition is derived from the **primary key**, and that is a correctness requirement rather than a tuning choice. The sync upserts by key and expresses "this key is superseded" as an equality delete, and Iceberg applies a delete file only to data files **in its own partition**. Bucketing by the key puts the delete in the same partition as the row it supersedes, however much later it is written. A partition derived from the write time cannot do that: the updated row moves to a later partition and the delete never reaches the older copy, leaving the key in the table twice.

Time-range queries still prune well without a time partition. An incremental sync commits one instant, so every data file covers a single `_neorun_synced_at` value and file-level min/max statistics skip whole files on a time predicate.

!!! warning "Tables created before NeorunBase 1.0.1"

    Earlier versions partitioned synced tables by `days(_neorun_synced_at)`. On such a table an upsert cannot remove a copy of the key that an earlier partition still holds, so a key updated after its partition rolled over appears **twice**. The partition values live in committed data files and cannot be repaired in place — drop the Iceberg table and let the next sync re-create it in the current layout. The sync logs a warning naming any table still in the old shape.

## Schema Evolution — `ADD COLUMN`

Adding a column to a NeorunBase table (`ALTER TABLE … ADD COLUMN …`) is propagated to the matching Iceberg table on the next sync. The column is added as nullable on the Iceberg side (existing snapshots have no value for it).

`DROP COLUMN`, `RENAME COLUMN`, and type promotion are not yet propagated — drop and recreate the Iceberg table if the source schema changes that way.

## Reading Iceberg Tables (`iceberg.<ns>.<table>`)

`SELECT * FROM iceberg.<namespace>.<table>` is a federated read against the Iceberg catalog. Queries support:

- WHERE-clause and column-projection pushdown to data nodes
- **Equality deletes**, **position deletes**, and Iceberg **format-version 3 deletion vectors** are all applied at read time, so external tools writing into the same table stay compatible

!!! tip "Low-latency serving (LakeBase)"
    A `WHERE` on the table's primary key (`= / IN / BETWEEN / <,<=,>,>=`) is answered by an
    automatic per-snapshot **point/range index** in single-digit milliseconds at thousands of
    QPS, instead of a multi-file scan — turning NeorunBase into a serving layer over the open
    lakehouse without copying data. See **[Iceberg Serving (LakeBase)](iceberg-serving.md)**
    for the index, persistence, maintenance, manual `CREATE INDEX`, and governance patterns.

### Format version 2 and 3 (deletion vectors)

NeorunBase reads **and writes** tables in both Iceberg format-version 2 and 3. On v3, deleted rows are recorded as **deletion vectors** — a roaring bitmap of deleted row positions stored in a [Puffin](https://iceberg.apache.org/puffin-spec/) `deletion-vector-v1` blob, in place of position-delete files. NeorunBase applies vectors written by other engines (Ontul, Trino, Spark) transparently, and writes them itself when a row-level `DELETE` or `UPDATE` runs against a v3 table.

New tables created by NeorunBase are v2 by default; opt into v3 with `-Dneorunbase.iceberg.format.version=3` (or `NEORUNBASE_ICEBERG_FORMAT_VERSION=3`).

Which delete form NeorunBase *writes* depends on the path. The CDC sync writes **equality deletes** on both v2 and v3, because it upserts by primary key and equality deletes remain valid in v3. A row-level `DELETE` or `UPDATE` writes **positional** deletes instead — a parquet position-delete file on v2, a Puffin deletion vector on v3 — see [Row-Level DML on Iceberg Tables](iceberg-dml.md).

Both directions are verified against foreign engines rather than assumed:

| Direction | What is proven | Engine |
| --- | --- | --- |
| NeorunBase **reads** another engine's v3 deletion vectors | Ontul `DELETE`s on a shared v3 table; NeorunBase must return only the survivors, on both the scan path and the per-snapshot [PK index](iceberg-serving.md#the-primary-key-index) | Ontul |
| Another engine **reads** NeorunBase's delete files | NeorunBase `DELETE`/`UPDATE`/`INSERT`s; Trino must report the identical rows | Trino |

### Multi-Format Data Files (Parquet, ORC, Avro)

NeorunBase reads Iceberg data files in all three formats — **PARQUET**, **ORC**, and **AVRO** — including tables written by other engines such as Trino and Spark. The format is detected **per file** from the `DataFile` format recorded in the Iceberg manifest, so a single table may mix formats across snapshots and still read correctly.

Multi-format read applies everywhere an external Iceberg table is read:

- The distributed data-node scan
- The coordinator-local scan
- Loading an external Iceberg table into LakeBase

NeorunBase still **writes Parquet** for its own CDC sync (data and delete files) — that is unchanged. ORC and Avro are supported on the read path only.

`INSERT`, `UPDATE` and `DELETE` run directly against an Iceberg table, as does `MERGE INTO`. `UPDATE`, `DELETE` and `MERGE INTO` are all **merge-on-read**: the matched rows are recorded as position-delete files and the data files are never rewritten. See [Row-Level DML on Iceberg Tables](iceberg-dml.md) for what gets written on each format version and how other engines read it.

## Open Lakehouse Analytics

Once a table is synced, any engine that speaks Iceberg can read it: Spark, Trino, Hive, Flink, etc.

## What Is Not (Yet) Supported

- `DROP COLUMN`, `RENAME COLUMN`, type promotion in schema evolution
- Snapshot expiration and small-file / manifest compaction
- Time travel (`AS OF SNAPSHOT` / `AS OF TIMESTAMP`)
- Branch and tag **DDL** (creating, dropping or listing them from SQL). A configured
  [WAP](iceberg-wap.md) branch is created and fast-forwarded for you; there is no general
  branching surface beyond that.
- Iceberg REST catalogs other than Polaris
