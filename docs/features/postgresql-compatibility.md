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

- **DML**: `SELECT`, `INSERT`, `UPDATE`, `DELETE`
- **DDL**: `CREATE TABLE`, `DROP TABLE`, `ALTER TABLE`, `CREATE INDEX`, `DROP INDEX`, `CREATE SCHEMA`, `DROP SCHEMA`
- **Transactions**: `BEGIN`, `COMMIT`, `ROLLBACK`
- **Queries**: `JOIN`, `GROUP BY`, `ORDER BY`, `LIMIT`, `HAVING`, subqueries, aggregation functions, `CASE WHEN`, `CAST`, and more

## Data Types

NeorunBase supports a wide range of data types:

- **Numeric**: `INTEGER`, `BIGINT`, `SMALLINT`, `FLOAT`, `DOUBLE`, `DECIMAL`
- **String**: `VARCHAR`, `TEXT`, `CHAR`
- **Temporal**: `DATE`, `TIME`, `TIMESTAMP`
- **Boolean**: `BOOLEAN`
- **Binary**: `BYTEA`
- **Geospatial**: `POINT`, `LINESTRING`, `POLYGON`, `GEOMETRY`
- **Vector**: `VECTOR(dim)` (pgvector-compatible; see [Vector Database](vector-database.md))
- **Other**: `JSON`, `UUID`

## Virtual Catalog

NeorunBase implements `pg_catalog` and `information_schema` virtual catalogs, enabling standard PostgreSQL introspection commands such as `\d`, `\dt`, and `\di` in `psql`. The same catalog patterns also drive JDBC `DatabaseMetaData.getSchemas()` / `getTables()` / column-detail queries, so JDBC-based tools (BI/ETL clients, IDE database explorers) can discover NeorunBase tables without any custom shim.

## Extended Query Protocol

The pg-wire implementation supports the full extended query flow — Parse / Bind / Describe / Execute / Sync — so JDBC clients using `PreparedStatement` work the same as simple-mode `Statement`s. Bound parameters are accepted in either text or binary format; the server decodes the binary wire format per the type OID declared in Parse (covering `INT2`/`INT4`/`INT8`, `FLOAT4`/`FLOAT8`, `BOOL`, `TEXT`/`VARCHAR`/`BPCHAR`, `BYTEA`, `UUID`, `DATE`, `TIMESTAMP`, `TIMESTAMPTZ`).
