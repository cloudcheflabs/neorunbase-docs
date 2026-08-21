# PostgreSQL Compatibility

NeorunBase implements the PostgreSQL wire protocol (v3), allowing any PostgreSQL-compatible client to connect seamlessly without modification.

## Supported Clients

You can connect to NeorunBase using any standard PostgreSQL client, including:

- `psql` command-line tool
- JDBC drivers (PostgreSQL JDBC)
- pgAdmin
- Any application or library that supports the PostgreSQL protocol

## SQL Support

NeorunBase supports standard SQL operations with PostgreSQL conformance:

- **DML**: `SELECT`, `INSERT` (incl. `INSERT … SELECT` and `ON CONFLICT`), `UPDATE`, `DELETE`, and `MERGE INTO` (upsert; merge-on-read on Iceberg targets)
- **DDL**: `CREATE TABLE` (incl. `PRIMARY KEY`, `UNIQUE`, `CHECK`, `DEFAULT`, and `FOREIGN KEY (...) REFERENCES ...` with `ON DELETE` / `ON UPDATE` `CASCADE` / `SET NULL` / `RESTRICT` / `NO ACTION`), `DROP TABLE [IF EXISTS]`, `ALTER TABLE`, `CREATE INDEX`, `DROP INDEX`, `CREATE SCHEMA`, `DROP SCHEMA`
- **Transactions**: `BEGIN` (or `START TRANSACTION`), `COMMIT`, `ROLLBACK`
- **Queries**: `JOIN` (incl. `NATURAL JOIN` and `JOIN … USING`), `GROUP BY`, `HAVING`, `ORDER BY` (column, alias, expression or ordinal), `LIMIT` / `OFFSET`, `SELECT DISTINCT`, derived tables, `WITH` (incl. `WITH RECURSIVE`), set operators (`UNION`, `UNION ALL`, `INTERSECT`, `EXCEPT`), window functions, `TABLESAMPLE BERNOULLI`, uncorrelated subqueries, aggregation functions (`COUNT`, `SUM`, `AVG`, `MIN`, `MAX`, `STRING_AGG`, `ARRAY_AGG`, and their `DISTINCT` forms), scalar functions (`ABS`, `ROUND`, `LENGTH`, `UPPER`, `LOWER`, `COALESCE`, `NULLIF`, `EXTRACT`, `TO_CHAR`, …), `CASE WHEN`, `CAST`, and more

An aggregate with **no `GROUP BY` is defined over the whole input, so it returns exactly one row even when nothing matched** — `COUNT` yields `0`, the others `NULL`:

```sql
SELECT count(*) FROM orders WHERE id = -1;   -- one row: 0
SELECT sum(amount) FROM orders WHERE id = -1; -- one row: NULL
```

This matters for drivers: a client that reads the value straight off the first row would otherwise find no row at all. With a `GROUP BY`, an empty input genuinely produces no groups and the result is empty, as PostgreSQL does.

### Predicates and operators

- **Negated predicates** keep their negation everywhere they can appear — standalone, under
  `AND`/`OR`, in `SELECT`, `UPDATE` and `DELETE`: `NOT LIKE`, `NOT ILIKE`, `NOT BETWEEN`,
  `NOT IN`, `IS NOT NULL`, `NOT (...)`, `<>`.
- **Pattern matching**: `LIKE` / `ILIKE`, SQL's `SIMILAR TO` (where `%` and `_` are the
  wildcards and the rest is regex), and the POSIX regex operators `~`, `~*`, `!~`, `!~*`.
- **Boolean tests**: `IS TRUE`, `IS FALSE`, `IS UNKNOWN` and their `IS NOT` forms. These are
  not the same as `= TRUE`: a boolean test answers `false` for NULL, where the comparison
  answers NULL.
- **Subquery predicates**: `EXISTS (…)`, `x IN (SELECT …)`, `x NOT IN (SELECT …)` and scalar
  `(SELECT …)` used as a value. The inner query runs on the coordinator and its result is
  substituted into the plan. **Correlated** subqueries — ones referring to a column of the
  outer query — are not supported and report the column they cannot resolve.
- **Array subscripting**: `arr[1]`, 1-based, with an out-of-range subscript yielding NULL.
- **String concatenation** uses `||` (`SELECT first_name || ' ' || last_name`), equivalent to
  `CONCAT(...)`.
- **A filter is never optional.** If a `WHERE` or `HAVING` clause uses something the engine
  cannot express, the statement is **refused**. It is not executed with the clause dropped.
  This matters most for `UPDATE` and `DELETE`, where running without the predicate would
  rewrite or remove every row in the table.
- **An unknown function is an error**, not a value. `SELECT bogus_fn(c)` fails rather than
  returning something that resembles data.
- **An unknown column is an error**, not NULL. `WHERE no_such_col = 1` and
  `SELECT no_such_col` both fail, rather than quietly matching nothing or returning blanks —
  a misspelled column is a mistake, and an empty result reads exactly like an empty table.

### Aggregates and windows

- `COUNT`, `SUM`, `AVG`, `MIN`, `MAX`, `STRING_AGG(expr, separator)` and `ARRAY_AGG(expr)`,
  plus `COUNT(DISTINCT …)`, `SUM(DISTINCT …)` and `AVG(DISTINCT …)`. The `DISTINCT` forms are
  computed by merging the shards' value sets, not by adding up per-shard counts — a value that
  appears on two shards is one value.
- Aggregates answer in the type the input implies: `SUM` over an `INT` column is an integer,
  `MIN`/`MAX` keep the column's type, `AVG` is double precision, `COUNT` is `BIGINT`.
- `HAVING` may reference an aggregate the `SELECT` list does not project
  (`SELECT city … GROUP BY city HAVING count(*) > 1`). The extra aggregate is computed and
  then trimmed from the result.
- Window functions: `SUM`, `AVG`, `MIN`, `MAX` and `COUNT` with
  `OVER (PARTITION BY … ORDER BY …)`, plus `ROW_NUMBER()`, `RANK()` and `DENSE_RANK()`.
  With an `ORDER BY` inside the window the aggregate is running (rows up to the current one);
  without one it spans the partition. Explicit frame clauses (`ROWS BETWEEN …`) are not
  supported yet. Windows are evaluated on the coordinator, since a frame spans rows no single
  shard holds.

### Constraints and defaults

A constraint that `CREATE TABLE` accepts is one the engine upholds:

- `PRIMARY KEY` and `FOREIGN KEY` (with the referential actions listed above).
- `UNIQUE` is checked on `INSERT` and `UPDATE`, both against stored rows and within the
  statement's own batch. It is a check rather than a lock: two concurrent inserts of the same
  new value can each pass their check and both land. An `UPDATE` that computes a `UNIQUE`
  column from the row itself (`SET email = lower(email)`) is refused rather than left
  unverified.
- `CHECK` is evaluated per row — on `INSERT` by the coordinator, on `UPDATE` by the data node,
  which is where the row an update produces first exists. A constraint that evaluates to NULL
  passes, as SQL defines.
- `DEFAULT` is applied to any column the `INSERT` did not name. A column the statement did
  name keeps what it was given, NULL included.

## Data Types

NeorunBase supports a wide range of data types:

- **Numeric**: `INTEGER` (`INT`/`INT4`), `BIGINT` (`INT8`), `SMALLINT` (`INT2`), `FLOAT` (`FLOAT4`), `REAL`, `DOUBLE` (`FLOAT8`/`DOUBLE PRECISION`), `NUMERIC`, `DECIMAL`
- **Auto-increment**: `SERIAL`, `BIGSERIAL`
- **String**: `VARCHAR`, `TEXT`, `CHAR`
- **Temporal**: `DATE`, `TIME`, `TIMESTAMP` (`TIMESTAMPTZ` / `TIMESTAMP WITH TIME ZONE` are accepted and mapped to `TIMESTAMP`), `INTERVAL`
- **Boolean**: `BOOLEAN` (`BOOL`)
- **Binary**: `BYTEA`
- **JSON**: `JSON`, `JSONB`
- **Arrays**: `INT[]`, `BIGINT[]`, `TEXT[]`, `FLOAT[]`
- **Geospatial**: `POINT`, `LINESTRING`, `POLYGON`, `GEOMETRY` (PostGIS-style, with `ST_*` functions such as `ST_Point`, `ST_Distance`, `ST_Contains`, `ST_GeomFromText`)
- **Full-text search**: `TSVECTOR`, `TSQUERY` (native PostgreSQL FTS OIDs `3614` / `3615`)
- **Vector**: `VECTOR(dim)` (pgvector-compatible; see [Vector Database](vector-database.md))
- **Other**: `UUID`

## Table-Valued Functions

NeorunBase ships a small set of table-valued functions (TVFs) that are usable anywhere a table reference is legal — typically inside `FROM` and JOINed against entity / metadata tables. Each one is rewritten on the Coordinator before SQL parsing so the rest of the query (JOINs, `WHERE`, `ORDER BY`, `LIMIT`) plans through the standard pipeline.

| Function                     | Returns           | Purpose                                                                  |
|------------------------------|-------------------|--------------------------------------------------------------------------|
| `GRAPH_NEIGHBORS(...)`       | `(id, depth)`     | Multi-hop BFS over an edge table; cycles broken by visited set            |
| `GRAPH_PATH_EXISTS(...)`     | `(path_exists)`   | Single-row BOOLEAN — does a path exist from `src` to `dst` within `max_depth`? RIG fact-check primitive |
| `PAGERANK(...)`              | `(node_id, rank)` | PageRank with optional edge-weight column and centralised / BSP-distributed mode |
| `PERSONALIZED_PAGERANK(...)` | `(node_id, rank)` | Personalised PageRank seeded on a query-specific entity                  |
| `HYBRID_SEARCH(...)`         | `(id, score)`     | Parallel BM25 + vector ANN scatter, blended into a single top-K (min-max or RRF) |

All five take named arguments (PostgreSQL `name => value` form). See [Graph Traversal & Analytics](graph.md) and [Hybrid Search](hybrid-search.md) for argument reference and example SQL.

## Virtual Catalog

NeorunBase implements `pg_catalog` and `information_schema` virtual catalogs, enabling standard PostgreSQL introspection commands in `psql` — including `\d`, `\dt`, `\dt+`, `\di`, `\dv`, `\ds`, `\dn` (schemas), `\du` (roles), `\df` (functions), `\dp` (privileges), and `\l` (databases). `\di` reports each index's **method** (`index (btree)` / `index (fts)` / `index (hnsw)`), so an index created without a `USING` clause — a BTREE that cannot serve `@@` or a vector search — is visible as such rather than indistinguishable from the index you meant to build. The same catalog patterns also drive JDBC `DatabaseMetaData.getSchemas()` / `getTables()` / column-detail queries, so JDBC-based tools (BI/ETL clients, IDE database explorers) can discover NeorunBase tables without any custom shim.

## Extended Query Protocol

The pg-wire implementation supports the full extended query flow — Parse / Bind / Describe / Execute / Sync — so JDBC clients using `PreparedStatement` work the same as simple-mode `Statement`s. Bound parameters are accepted in either text or binary format; the server decodes the binary wire format per the type OID declared in Parse (covering `INT2`/`INT4`/`INT8`, `FLOAT4`/`FLOAT8`, `BOOL`, `TEXT`/`VARCHAR`/`BPCHAR`, `BYTEA`, `UUID`, `DATE`, `TIMESTAMP`, `TIMESTAMPTZ`).
