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

A high-water mark is recorded inside the Iceberg table, so coordinator restarts resume sync exactly where it left off.

## Built-in Time Column + Partitioning

Every Iceberg table created by NeorunBase carries an extra column **`_neorun_synced_at`** (`timestamptz`). The column is set automatically — operators don't read or write to it.

Tables are also automatically:

- **Partitioned** by `days(_neorun_synced_at)` — time-range queries skip whole files.
- **Sorted** by `[_neorun_synced_at, primary key]` — within-partition primary-key ranges stay clustered.

The result is good time-range query performance out of the box without forcing operators to choose a partition column manually.

## Schema Evolution — `ADD COLUMN`

Adding a column to a NeorunBase table (`ALTER TABLE … ADD COLUMN …`) is propagated to the matching Iceberg table on the next sync. The column is added as nullable on the Iceberg side (existing snapshots have no value for it).

`DROP COLUMN`, `RENAME COLUMN`, and type promotion are not yet propagated — drop and recreate the Iceberg table if the source schema changes that way.

## Reading Iceberg Tables (`iceberg.<ns>.<table>`)

`SELECT * FROM iceberg.<namespace>.<table>` is a federated read against the Iceberg catalog. Queries support:

- WHERE-clause and column-projection pushdown to data nodes
- **Equality deletes**, **position deletes**, and Iceberg **format-version 3 deletion vectors** are all applied at read time, so external tools writing into the same table stay compatible

### Format version 2 and 3 (deletion vectors)

NeorunBase reads tables in both Iceberg format-version 2 and 3. Its own CDC sync writes **equality deletes**, which remain valid in v3, so no write change is needed for v3. Tables written by other engines (Ontul, Trino, Spark) on v3 use **deletion vectors** — a roaring bitmap of deleted row positions stored in a [Puffin](https://iceberg.apache.org/puffin-spec/) `deletion-vector-v1` blob instead of position-delete files — and NeorunBase reads and applies these transparently.

New tables created by NeorunBase are v2 by default; opt into v3 with `-Dneorunbase.iceberg.format.version=3` (or `NEORUNBASE_ICEBERG_FORMAT_VERSION=3`).

### Multi-Format Data Files (Parquet, ORC, Avro)

NeorunBase reads Iceberg data files in all three formats — **PARQUET**, **ORC**, and **AVRO** — including tables written by other engines such as Trino and Spark. The format is detected **per file** from the `DataFile` format recorded in the Iceberg manifest, so a single table may mix formats across snapshots and still read correctly.

Multi-format read applies everywhere an external Iceberg table is read:

- The distributed data-node scan
- The coordinator-local scan
- Loading an external Iceberg table into LakeBase

NeorunBase still **writes Parquet** for its own CDC sync (data and delete files) — that is unchanged. ORC and Avro are supported on the read path only.

`MERGE INTO iceberg.<ns>.<target> USING <source> …` is supported as a copy-on-write merge. Direct `INSERT`, `UPDATE`, `DELETE` against Iceberg tables are not — use the native NeorunBase table plus CDC sync, or `MERGE INTO`.

## Open Lakehouse Analytics

Once a table is synced, any engine that speaks Iceberg can read it: Spark, Trino, Hive, Flink, etc.

## What Is Not (Yet) Supported

- Direct `INSERT` / `UPDATE` / `DELETE` on Iceberg tables (use `MERGE INTO` or the native table)
- `DROP COLUMN`, `RENAME COLUMN`, type promotion in schema evolution
- Snapshot expiration and small-file / manifest compaction
- Time travel (`AS OF SNAPSHOT` / `AS OF TIMESTAMP`)
- Branching and tagging
- Iceberg REST catalogs other than Polaris
