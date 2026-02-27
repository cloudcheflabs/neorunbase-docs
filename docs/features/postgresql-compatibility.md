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
- **Other**: `JSON`, `UUID`

## Virtual Catalog

NeorunBase implements `pg_catalog` and `information_schema` virtual catalogs, enabling standard PostgreSQL introspection commands such as `\d`, `\dt`, and `\di` in `psql`.
