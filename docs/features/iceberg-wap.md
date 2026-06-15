# Iceberg Write-Audit-Publish (WAP)

Write-Audit-Publish (WAP) lets you stage Iceberg writes on a **branch**, **audit** the
staged data in isolation, and only then **publish** it to `main` — the version every
reader sees. Until you publish, `SELECT` over SQL keeps reading `main` as if nothing
changed, so bad or partial data never reaches consumers.

This builds on the [Iceberg Integration](iceberg-integration.md) catalog and CDC sync.

## The flow

```
        write                 audit                    publish
  ┌──────────────┐     ┌──────────────────┐     ┌───────────────────┐
  │ CTAS /       │     │ readers still see │     │ fast-forward main │
  │ MERGE INTO / │ ──▶ │ main (frozen);    │ ──▶ │ to the audit head │
  │ CDC sync     │     │ rows staged on    │     │ → rows go live    │
  │ → audit      │     │ the audit branch  │     │                   │
  └──────────────┘     └──────────────────┘     └───────────────────┘
```

1. **Write** — every Iceberg write lands on the WAP branch (e.g. `audit`) instead of
   `main`.
2. **Audit** — `main` is frozen; `SELECT ... FROM <catalog>.<ns>.<table>` over psql/JDBC
   still reads `main` and returns the *previous* contents. The staged rows live only on
   the branch, where you inspect them via the Iceberg/Polaris REST catalog.
3. **Publish** — a single REST call fast-forwards `main` to the branch head, atomically
   making the staged data visible to all readers.

## Selecting the WAP branch

There is **no SQL `SET`** for the WAP branch. Pick one of two ways:

### 1. Per-catalog — `CREATE CATALOG ... WITH ('wap.branch'=...)`

```sql
CREATE CATALOG iceberg WITH (
    type='rest',
    uri='http://polaris:8181/api/catalog',
    warehouse='quickstart_catalog',
    security='OAUTH2',
    'client-id'='root',
    'client-secret'='s3cr3t',
    'extra-properties'='prefix=quickstart_catalog,header.Polaris-Realm=POLARIS,scope=PRINCIPAL_ROLE:ALL',
    's3.endpoint'='http://minio:9000',
    's3.access-key'='minioadmin',
    's3.secret-key'='minioadmin',
    's3.region'='us-east-1',
    's3.path-style-access'='true',
    'wap.branch'='audit'
);
```

Every Iceberg write through this catalog is routed to the `audit` branch.

### 2. Coordinator system property — `-Dneorunbase.iceberg.wap.branch`

Launch the coordinator with:

```bash
-Dneorunbase.iceberg.wap.branch=audit
```

(equivalently `NEORUNBASE_ICEBERG_WAP_BRANCH=audit`). This is what the end-to-end test
[`tests/test-iceberg-wap-e2e.sh`](#references) injects into coordinator-1.

#### Table-scoped overrides

The system property can be narrowed to specific tables. Most specific wins:

| Property | Scope |
| --- | --- |
| `-Dneorunbase.iceberg.wap.branch.<namespace>.<table>` | one table in one namespace |
| `-Dneorunbase.iceberg.wap.branch.<table>` | a table by name (any namespace) |
| `-Dneorunbase.iceberg.wap.branch` | default for all Iceberg writes |

A catalog's `'wap.branch'` option, when present, takes precedence over the system
property for that catalog. Leave all of these unset and writes go straight to `main`
(WAP disabled).

## Behavior for CTAS / MERGE INTO / CDC sync

When a WAP branch is active, the branch is created if missing (branched off the current
`main`) and **all** of these write to it, never to `main`:

- **CTAS** — `CREATE TABLE iceberg.<ns>.<t> AS SELECT ...` stages the new snapshot on the
  branch.
- **`MERGE INTO`** — `MERGE INTO iceberg.<ns>.<target> USING <source> ...` (copy-on-write)
  commits its add-data + delete files to the branch.
- **CDC sync** — the automatic table mirror (full first sync and incremental
  add-data + equality-delete commits) targets the branch. The sync high-water mark is
  still tracked, so restarts resume correctly.

`main` stays exactly as it was until you publish, so concurrent readers are unaffected.

## Auditing the branch

Because `main` is frozen, `SELECT` over SQL will **not** show the staged rows — that is the
point. To audit the staged data, read the table's metadata refs from the Iceberg/Polaris
REST catalog: the branch appears under `.metadata.refs[<branch>]` with its head
`snapshot-id`.

```bash
# Polaris OAuth token
POLARIS_TOKEN=$(curl -s http://localhost:8181/api/catalog/v1/oauth/tokens \
  -H "Polaris-Realm: POLARIS" \
  -d grant_type=client_credentials -d client_id=root -d client_secret=s3cr3t \
  -d scope=PRINCIPAL_ROLE:ALL | python3 -c "import sys,json;print(json.load(sys.stdin)['access_token'])")

# Table metadata — look at .metadata.refs.audit.snapshot-id
curl -s "http://localhost:8181/api/catalog/v1/quickstart_catalog/namespaces/public/tables/wap_demo" \
  -H "Authorization: Bearer $POLARIS_TOKEN" -H "Polaris-Realm: POLARIS" \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['metadata']['refs'])"
```

From that snapshot you can read the staged parquet files directly (see the Python
example and `tests/query_iceberg.py`), run row counts, checksums, or any data-quality
check before publishing.

## Publishing (REST)

Publish is an authenticated admin REST call. First obtain a JWT:

```bash
TOKEN=$(curl -s -X POST http://localhost:8084/admin/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"admin"}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['token'])")
```

### Fast-forward publish

`POST /admin/api/iceberg/wap/publish` advances `to` (default `main`) to the head of
`branch`:

```bash
curl -s -X POST http://localhost:8084/admin/api/iceberg/wap/publish \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"namespace":"public","table":"wap_demo","branch":"audit","to":"main"}'
# -> {"success":true,"publishedSnapshotId":...,"from":"audit","to":"main"}
```

After this, `SELECT ... FROM iceberg.public.wap_demo` returns the published rows.

### Cherry-pick publish

When you want to publish a single specific snapshot (rather than fast-forward the whole
branch), use `POST /admin/api/iceberg/wap/cherrypick`:

```bash
curl -s -X POST http://localhost:8084/admin/api/iceberg/wap/cherrypick \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"namespace":"public","table":"wap_demo","snapshotId":1234567890123456789}'
# -> {"success":true,"publishedSnapshotId":...,"cherrypickedSnapshotId":1234567890123456789}
```

!!! note "Auth and leader"
    Both endpoints require the `Authorization: Bearer <token>` header. They are handled
    on the cluster leader; requests to a follower coordinator are forwarded automatically.

## Examples and end-to-end test

A complete, runnable demonstration ships in the NeorunBase repo:

- **Java (JDBC)** — `neorunbase-server/src/test/java/com/cloudcheflabs/neorunbase/server/IcebergWapExample.java`.
  Connects via JDBC, creates the catalog with `'wap.branch'='audit'`, CTAS into the
  Iceberg table, shows `main` is empty while the `audit` branch holds the rows, calls the
  publish endpoint with `java.net.http.HttpClient`, then re-reads `main`.

    ```bash
    CP=$(./gradlew -q :neorunbase-server:printTestClasspath)
    CP="neorunbase-server/build/classes/java/test:$CP"
    java -cp "$CP" com.cloudcheflabs.neorunbase.server.IcebergWapExample \
        localhost 5434 localhost 8084 admin admin
    ```

- **Python** — `examples/python/iceberg_wap_example.py`. Runs CTAS via psycopg2 (or the
  `psql` CLI), audits the `audit` branch through the Polaris REST catalog
  (`.metadata.refs`), posts to the publish endpoint, then re-verifies `main`. Connection
  details come from environment variables.

    ```bash
    pip install psycopg2-binary requests
    python3 examples/python/iceberg_wap_example.py
    ```

- **End-to-end script** — `tests/test-iceberg-wap-e2e.sh` brings up Polaris + MinIO + a
  two-coordinator NeorunBase cluster (Docker), runs the full write → audit → publish flow,
  and asserts that `main` is empty before publish and has the rows after.

All three use the same names: catalog `iceberg`, namespace `public`, table `wap_demo`,
WAP branch `audit`.
