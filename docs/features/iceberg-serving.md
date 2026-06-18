# Iceberg Serving (LakeBase)

NeorunBase serves Apache Iceberg tables as a **LakeBase**: a low-latency, high-QPS
*serving* layer over the open lakehouse, **without copying rows into NeorunBase's own
storage**. A `SELECT` with a primary-key predicate on an Iceberg table is answered in
**single-digit milliseconds at thousands of QPS** — comparable to, and in the point-lookup
case faster than, a native RocksDB read — while the data itself stays in Iceberg/S3 and
remains readable by every other engine (Spark, Trino, Flink, …).

Analytics (large scans, joins, aggregations) belong on an analytical engine; NeorunBase's
job here is **serving**: fast point and range reads of governed Iceberg tables.

!!! info "How fast?"
    On a 20,000-row table split across 8 data files, 16 concurrent clients over the
    PostgreSQL wire protocol: **point lookup ≈ 9,000 TPS / ~1.8 ms**, **range scan
    (`BETWEEN`) ≈ 9,000 TPS / ~1.8 ms**. Without the index the same workload is a
    multi-file S3 scan at tens of TPS and hundreds of milliseconds.

## Why Iceberg alone is slow to serve

Iceberg is an excellent *table format*, but it has **no row-level index**. Its only pruning
inputs are:

- **partition values**, and
- **per-file column min/max statistics** (used to skip whole data files).

For a serving workload these are often not enough:

- Primary keys are frequently **hash-distributed across files** (e.g. a NeorunBase native
  table synced to Iceberg shards rows by primary key), so every file's `[min, max]` for the
  key column overlaps and min/max pruning cannot isolate a single file.
- Even when pruning isolates one file, that file still has to be read from S3 and scanned
  row by row to find the one matching row.

NeorunBase closes this gap with a **secondary index that stores locations only** — never the
row data. This is the LakeBase "pointer index": the keys and their `(file, row)` locations
live in NeorunBase; the rows stay in Iceberg.

## The primary-key index

For an Iceberg table that declares a **single identifier (primary-key) field**, NeorunBase
keeps a per-snapshot ordered map:

```
primary key  ->  (data file, row ordinal)
```

A serving query then resolves the matching key(s) and reads **only the matching rows** from
the (cached) data files — no multi-file scan, no data-node round trip.

### Supported predicates

The index fast path activates when the `WHERE` clause filters the indexed column with any of:

| Form | Example |
| --- | --- |
| Equality | `WHERE id = 42` |
| `IN` list | `WHERE id IN (1, 2, 3)` |
| `BETWEEN` | `WHERE id BETWEEN 100 AND 199` |
| Range comparison | `WHERE id >= 1000`, `WHERE id < 50` |
| `AND` of the above on the same column | `WHERE id >= 100 AND id <= 199` |

The index is an **ordered** structure, so range predicates are served by an in-memory range
scan over the keys. Constant-folded bounds (e.g. `id BETWEEN :lo AND :lo + 99`) are evaluated
before lookup. Any other predicate shape (or a column without an index) falls back to the
normal scan path, and the full `WHERE` is always re-applied to the returned rows so results
are correct regardless of which path ran.

Range lookups require an ordered key type (integer, bigint, or string/uuid). Other key types
(decimal, float) support equality/`IN` only.

### Fully automatic

There is **no `CREATE INDEX` required** for the primary-key index:

- It is **built lazily** on the first primary-key query against a table.
- It is **invalidated by snapshot id** — when a write produces a new snapshot the index is
  refreshed automatically (see *Maintenance* below). No cron job, no manual rebuild.
- It is **scoped by catalog index patterns** (see *Governance* below), so a catalog with
  thousands of tables never indexes anything unintentionally.

### Persistence across restart

The index is persisted to a local RocksDB store under
`<neorunbase.base.data.dir>/iceberg-pk-index`. On a coordinator restart the index is
**restored from local disk** (when it still matches the current snapshot) instead of
re-reading every data file from S3. Only `(file, row)` locations and a small file dictionary
are stored — never row data.

Toggle with `neorunbase.iceberg.pk.index.persist` (default `true`).

## Maintenance: eager + incremental

The index always reflects the table's current snapshot. Two mechanisms keep that cheap:

**Eager invalidation.** After a write commits a new snapshot — a native→Iceberg sync, a
`MERGE INTO`, an append, or a CTAS — NeorunBase eagerly evicts the cached snapshot for that
table across **every catalog** that exposes it, so the *next* read sees fresh data without
waiting out the metadata cache TTL.

**Incremental update.** When the snapshot advances and every snapshot in between is a plain
**append** (no deletes/overwrites/replaces), NeorunBase indexes **only the newly added data
files** and merges them into the existing index — it does not re-read the whole table. If a
delete or overwrite appears in the range, it safely falls back to a full rebuild.

Between explicit writes the snapshot is immutable, so the index needs no work at all.

!!! note "Leader-master"
    All DDL and declarations (`CREATE INDEX`, catalog changes) are performed by the cluster
    **leader** (followers transparently forward them), and declarations are written to the
    replicated metadata store, so they are honored cluster-wide.

## Manual secondary indexes

For columns that are **not** the primary key — or for external Iceberg tables that do not
declare an identifier field at all — you can declare a secondary index explicitly:

```sql
CREATE INDEX idx_grp ON ice.public.events (grp);
```

```sql
DROP INDEX grp ON ice.public.events;   -- the index name is the column name
```

A secondary index is **non-unique** (a value maps to many rows) and supports the same point
and range predicates as the primary-key index. The declaration is stored in the replicated
metadata store, so a `CREATE INDEX` issued on one coordinator is honored by all of them; the
index itself builds lazily on the next query. Manual secondary indexes are **always honored**,
independent of the catalog's auto-index patterns.

## Governance: which tables get indexed

A single Iceberg catalog can expose **thousands of tables**. Automatically indexing all of
them would be wasteful, so the automatic primary-key index is **scoped by a per-catalog table
pattern**:

```sql
CREATE CATALOG IF NOT EXISTS "ice" WITH (
  'type'             = 'rest',
  'uri'              = 'http://polaris:8181/api/catalog',
  ...
  'index.patterns'   = 'sales.*, users.tuser*, ops.events'
);
```

`index.patterns` is a comma-separated list of `namespace.tableGlob` patterns where `*` matches
any run of characters and `?` matches one character:

| Pattern | Matches |
| --- | --- |
| `sales.*` | every table in the `sales` namespace |
| `users.tuser*` | tables in `users` whose name starts with `tuser` |
| `ops.events` | exactly `ops.events` |

- **Blank (the default) = opt-in OFF**: no table in the catalog is auto-indexed. This makes a
  large catalog safe by default.
- Changing the patterns (via `ALTER CATALOG` or the Admin UI) takes effect on the next query:
  newly matching tables get indexed, no-longer-matching tables stop using the index.
- Manually declared secondary indexes ignore this setting (they are an explicit opt-in for
  their table).

The pattern can also be set in the **Admin UI → Catalogs** form ("Index table patterns"), or
for the startup default catalog via `neorunbase.iceberg.index.patterns` in
`neorunbase.properties`.

## Configuration

All serving knobs live in `neorunbase.properties` (and may be overridden with a matching `-D`
JVM flag, which always wins):

| Property | Default | Description |
| --- | --- | --- |
| `neorunbase.iceberg.pk.index.enabled` | `true` | Enable the PK point/range index. |
| `neorunbase.iceberg.pk.index.persist` | `true` | Persist the index to local RocksDB for restart restore. |
| `neorunbase.iceberg.index.patterns` | *(blank)* | Auto-index table patterns for the **default** catalog (named catalogs set their own). |
| `neorunbase.iceberg.read.cache.ttl.ms` | `30000` | Metadata cache TTL; bounds freshness for changes made by *other* engines (own writes evict eagerly). |
| `neorunbase.iceberg.data.cache.rows` | `2000000` | Max rows held in the in-memory data-file cache (LRU). `0` disables. |
| `neorunbase.iceberg.eqdelete.cache.files` | `1024` | Max equality-delete files whose keys are cached (LRU). |
| `neorunbase.iceberg.plan.cache.enabled` | `true` | Cache the per-snapshot file plan + do in-memory min/max pruning. |
| `neorunbase.iceberg.scan.threads` | `16` | Parallel scan pool for delete-bearing table reads. |
| `neorunbase.plan.cache.size` | `2000` | Coordinator SQL plan cache size (parsed non-aggregation SELECTs). |

## What stays in Iceberg

The index and caches are **acceleration structures, not a copy of the data**:

- Row data is never moved into NeorunBase storage; the index holds only keys and
  `(file, row)` locations.
- Other engines continue to read and write the table normally.
- A dropped index, a disabled index, or a cold cache only affects latency — never
  correctness. The serving query transparently falls back to scanning Iceberg.

## See also

- [Iceberg Integration](iceberg-integration.md) — sync, partitioning, multi-format reads.
- [Catalogs](catalogs.md) — registering and governing Iceberg catalogs.
