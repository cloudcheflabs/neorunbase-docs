# Catalogs

NeorunBase organizes all tables under named **catalogs**. The built-in catalog is the cluster's native distributed store; additional catalogs connect to external Apache Iceberg catalogs, so one SQL surface reaches both NeorunBase-native tables and lakehouse tables managed elsewhere.

## The Built-in `lakebase` Catalog

`lakebase` is NeorunBase's own distributed store. It is **always present** and **cannot be created or dropped**. Native tables are addressed with a two-part name:

```sql
SELECT * FROM analytics.orders;          -- schema.table
SELECT * FROM lakebase.analytics.orders; -- catalog.schema.table (equivalent)
```

When a table reference omits the catalog, NeorunBase resolves it against `lakebase`. The two forms above are equivalent.

## External Iceberg Catalogs

In addition to `lakebase` you can register any number of external Iceberg catalogs by name (for example `iceberg-east`, `iceberg-west`). A catalog is a connection to a REST/Polaris Iceberg catalog plus its own S3 backend. Tables in a registered catalog use a three-part name:

```sql
SELECT * FROM "iceberg-east".sales.events;  -- <catalog>.<namespace>.<table>
```

!!! note "Connectors today"
    Only the **Iceberg** connector exists today (`'type'='rest'`, talking to a REST/Polaris catalog). The catalog layer is built so more connector types can be added later without changing how catalogs are addressed in SQL.

## Managing Catalogs with SQL

Catalog DDL is **leader-forwarded**: it runs on the leader coordinator, is persisted in the cluster metadata store, and is replicated to every coordinator (leader + followers), so every node resolves the same set of catalogs.

### `CREATE CATALOG`

```sql
CREATE CATALOG IF NOT EXISTS "iceberg-east" WITH (
  'type'              = 'rest',
  'uri'               = 'http://polaris:8181/api/catalog',
  'warehouse'         = 'my_catalog',
  'security'          = 'OAUTH2',
  'client-id'         = '...',
  'client-secret'     = '...',
  'extra-properties'  = 'prefix=my_catalog,header.Polaris-Realm=POLARIS,scope=PRINCIPAL_ROLE:ALL',
  's3.endpoint'       = 'http://minio:9000',
  's3.access-key'     = '...',
  's3.secret-key'     = '...',
  's3.region'         = 'us-east-1',
  's3.path-style-access' = 'true',
  'default-namespace' = 'lakebase'
);
```

Option keys may be **bare or quoted** (Trino-style), and values may use single or double quotes. The parser is **comma-safe**, so values such as `extra-properties` can themselves contain commas.

| Option | Description |
|--------|-------------|
| `type` | Connector type. Currently `rest` (REST/Polaris Iceberg catalog). |
| `uri` | Iceberg REST catalog base URI (the `/api/catalog` path). |
| `warehouse` | Iceberg warehouse / catalog name. |
| `security` | `OAUTH2` for the client-credentials flow. |
| `client-id` / `client-secret` | OAuth2 client credentials. |
| `extra-properties` | Free-form extra REST-catalog properties (comma-separated `key=value`), e.g. `prefix`, `header.Polaris-Realm`, `scope`. |
| `s3.endpoint` | S3 (or S3-compatible) endpoint for the catalog's data files. |
| `s3.access-key` / `s3.secret-key` | Static S3 credentials. |
| `s3.region` | S3 region string. |
| `s3.path-style-access` | `true` for MinIO/ShannonStore and most S3-compatible gateways. |
| `default-namespace` | Namespace used when one is not given. |
| `index.patterns` | Comma-separated `namespace.tableGlob` patterns (`*`/`?` wildcards) selecting which tables get an automatic primary-key serving index. **Blank = opt-in OFF** (nothing auto-indexed) — safe for catalogs with thousands of tables. See [Iceberg Serving](iceberg-serving.md). |
| `wap.branch` | Default Write-Audit-Publish branch for this catalog's Iceberg writes. See [Iceberg WAP](iceberg-wap.md). |

### `ALTER CATALOG`

```sql
ALTER CATALOG "iceberg-east" SET (
  's3.endpoint' = 'http://minio-new:9000'
);
```

`ALTER CATALOG … SET` **patches only the named options** and leaves the rest of the definition intact. You can change one field without re-supplying secrets such as `client-secret` or `s3.secret-key`.

### `DROP CATALOG`

```sql
DROP CATALOG IF EXISTS "iceberg-east";
```

A `DROP CATALOG` is **refused if the catalog still contains any namespace with tables** — empty the catalog first. The built-in `lakebase` catalog **cannot be dropped**.

### Inspecting Catalogs

```sql
SHOW CATALOGS;                              -- lakebase + every registered catalog
SHOW SCHEMAS FROM "iceberg-east";
SHOW TABLES  FROM "iceberg-east".sales;
SHOW CREATE TABLE "iceberg-east".sales.events;
```

## Startup Default Catalog

The startup properties `neorunbase.iceberg.*` (`catalog.type=polaris`/`rest` plus the `polaris.*` / `s3.*` fields) **seed a DEFAULT catalog at boot**. Additional catalogs are then created at runtime via `CREATE CATALOG` or the Admin UI. See [Configuration](../configuration/configuration.md#iceberg-catalog-integration) for the property reference.

## Admin REST API

Catalog management is also available through the admin REST API. All endpoints are JWT-authenticated and **leader-forwarded** (non-leaders proxy the call to the leader). Secrets are **masked** in responses.

| Method & Path | Description |
|---------------|-------------|
| `GET /admin/api/catalogs` | List catalogs (`lakebase` + registered; secrets masked). |
| `GET /admin/api/catalogs/<name>` | Fetch one catalog (secrets masked). |
| `POST /admin/api/catalogs` | Create a catalog. |
| `PUT /admin/api/catalogs/<name>` | Alter (patch) a catalog's options. |
| `DELETE /admin/api/catalogs/<name>` | Drop a catalog. Returns **HTTP 409** if it is non-empty. |

Create a catalog:

```json
POST /admin/api/catalogs
{
  "name": "iceberg-east",
  "options": {
    "type": "rest",
    "uri": "http://polaris:8181/api/catalog",
    "warehouse": "my_catalog",
    "security": "OAUTH2",
    "client-id": "...",
    "client-secret": "...",
    "extra-properties": "prefix=my_catalog,header.Polaris-Realm=POLARIS,scope=PRINCIPAL_ROLE:ALL",
    "s3.endpoint": "http://minio:9000",
    "s3.access-key": "...",
    "s3.secret-key": "...",
    "s3.region": "us-east-1",
    "s3.path-style-access": "true",
    "default-namespace": "lakebase"
  }
}
```

Alter a catalog (patch — unspecified options are preserved):

```json
PUT /admin/api/catalogs/iceberg-east
{
  "options": {
    "s3.endpoint": "http://minio-new:9000"
  }
}
```

## Admin UI

The Admin UI includes a **Catalogs** page in the sidebar. It lists every catalog and lets you create, edit, and drop them through a form. As with the SQL and REST paths, a drop is **refused for a non-empty catalog**, and `lakebase` cannot be removed.

## Authorization

Catalog DDL is gated by IAM. `CREATE CATALOG`, `DROP CATALOG`, and `ALTER CATALOG` are authorized as the actions `pg:CreateCatalog`, `pg:DropCatalog`, and `pg:AlterCatalog` on the resource `db:catalog:<name>`. See the [IAM Policy Reference](iam-policy.md#catalog-level-resources) for examples.
