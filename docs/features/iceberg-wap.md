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
  │ DML /        │     │ rows staged on    │     │ → rows go live    │
  │ CDC sync     │     │ the audit branch  │     │                   │
  │ → audit      │     │                   │     │                   │
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
`tests/test-iceberg-wap-e2e.sh` injects into coordinator-1.

#### Full precedence (most specific wins)

The branch for a given write is resolved in this exact order — the first source that
yields a non-empty, non-`main` branch wins:

| # | Source | Scope |
| --- | --- | --- |
| 1 | `-Dneorunbase.iceberg.wap.branch.<namespace>.<table>` | one fully-qualified table |
| 2 | `-Dneorunbase.iceberg.wap.branch.<table>` | a table by name (any namespace) |
| 3 | `CREATE CATALOG ... WITH ('wap.branch'='<branch>')` | that catalog |
| 4 | `-Dneorunbase.iceberg.wap.branch` | cluster-wide default for all Iceberg writes |

Note the ordering: the two **table-scoped system properties (1, 2) override a catalog's
`'wap.branch'` option (3)**, and the catalog option in turn overrides the cluster-wide
default (4). Leave all four unset and writes go straight to `main` (WAP disabled).

## Behavior for CTAS / MERGE INTO / CDC sync / row-level DML

When a WAP branch is active, the branch is created if missing (branched off the current
`main`) and **all** of these write to it, never to `main`:

- **CTAS** — `CREATE TABLE iceberg.<ns>.<t> AS SELECT ...` stages the new snapshot on the
  branch.
- **`MERGE INTO`** — `MERGE INTO iceberg.<ns>.<target> USING <source> ...` (merge-on-read)
  commits its add-data + position-delete files to the branch.
- **CDC sync** — the automatic table mirror (full first sync and incremental
  add-data + equality-delete commits) targets the branch. The sync high-water mark is
  still tracked, so restarts resume correctly.
- **Row-level `INSERT` / `UPDATE` / `DELETE`** — the merge-on-read `RowDelta` (position
  deletes on v2, deletion vectors on v3) is committed to the branch, so a delete can be
  staged and audited before it becomes visible. See
  [Row-Level DML on Iceberg Tables](iceberg-dml.md).

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

## Examples

Everything is programmable — NeorunBase speaks the PostgreSQL wire protocol, so writes and the
audit run over plain JDBC/`psycopg2`, and publish is a REST call. With the catalog created
`WITH ('wap.branch'='audit')`, the `CREATE TABLE ... AS SELECT` lands on the `audit` branch and
`main` stays empty until you publish.

### Java (JDBC)

```java
import java.net.URI;
import java.net.http.*;
import java.sql.*;

// JDBC to the coordinator; admin REST token obtained from POST /admin/auth/login.
try (Connection conn = DriverManager.getConnection(
        "jdbc:postgresql://localhost:5434/neorunbase?preferQueryMode=simple", "admin", password);
     Statement stmt = conn.createStatement()) {

    // 1. Catalog with WAP active — every Iceberg write goes to the 'audit' branch.
    stmt.execute("CREATE CATALOG iceberg WITH ("
            + "type='rest', uri='http://localhost:8181/api/catalog', warehouse='quickstart_catalog', "
            + "security='OAUTH2', 'client-id'='root', 'client-secret'='s3cr3t', "
            + "'s3.endpoint'='http://localhost:9000', 's3.access-key'='minioadmin', "
            + "'s3.secret-key'='minioadmin', 's3.path-style-access'='true', "
            + "'wap.branch'='audit')");

    // 2. WRITE — CTAS into the Iceberg table; rows land on 'audit', main stays empty.
    stmt.execute("CREATE TABLE iceberg.public.wap_demo AS SELECT * FROM wap_src");

    // 3. AUDIT — SELECT over main returns 0 (frozen) until publish.
    try (ResultSet rs = stmt.executeQuery("SELECT COUNT(*) FROM iceberg.public.wap_demo")) {
        rs.next();
        System.out.println("main rows before publish = " + rs.getLong(1));   // 0
    }

    // 4. PUBLISH — fast-forward main to the audit branch via the admin REST API.
    HttpRequest req = HttpRequest.newBuilder()
            .uri(URI.create("http://localhost:8084/admin/api/iceberg/wap/publish"))
            .header("Authorization", "Bearer " + token)
            .header("Content-Type", "application/json")
            .POST(HttpRequest.BodyPublishers.ofString(
                "{\"namespace\":\"public\",\"table\":\"wap_demo\",\"branch\":\"audit\",\"to\":\"main\"}"))
            .build();
    HttpClient.newHttpClient().send(req, HttpResponse.BodyHandlers.ofString());

    // 5. VERIFY — main now reflects the published rows.
    try (ResultSet rs = stmt.executeQuery("SELECT COUNT(*) FROM iceberg.public.wap_demo")) {
        rs.next();
        System.out.println("main rows after publish = " + rs.getLong(1));    // 3
    }
}
```

### Python (`psycopg2` + `requests`)

```python
import psycopg2, requests

conn = psycopg2.connect(host="localhost", port=5434, dbname="neorunbase",
                        user="admin", password=PASSWORD)
conn.autocommit = True
cur = conn.cursor()

# 1. Catalog with WAP active.
cur.execute("CREATE CATALOG iceberg WITH ("
            "type='rest', uri='http://localhost:8181/api/catalog', warehouse='quickstart_catalog', "
            "security='OAUTH2', 'client-id'='root', 'client-secret'='s3cr3t', "
            "'s3.endpoint'='http://localhost:9000', 's3.access-key'='minioadmin', "
            "'s3.secret-key'='minioadmin', 's3.path-style-access'='true', "
            "'wap.branch'='audit')")

# 2. WRITE — CTAS; rows land on the 'audit' branch.
cur.execute("CREATE TABLE iceberg.public.wap_demo AS SELECT * FROM wap_src")

# 3. AUDIT — main is empty (frozen) before publish.
cur.execute("SELECT COUNT(*) FROM iceberg.public.wap_demo")
print("main before publish =", cur.fetchone()[0])     # 0

# 4. PUBLISH — fast-forward main to audit.
requests.post("http://localhost:8084/admin/api/iceberg/wap/publish",
              headers={"Authorization": f"Bearer {token}"},
              json={"namespace": "public", "table": "wap_demo", "branch": "audit", "to": "main"})

# 5. VERIFY — main now has the rows.
cur.execute("SELECT COUNT(*) FROM iceberg.public.wap_demo")
print("main after publish =", cur.fetchone()[0])      # 3
```

The same `audit` branch can also be selected without `CREATE CATALOG WITH` by launching the
coordinator with `-Dneorunbase.iceberg.wap.branch=audit` (see [Selecting the WAP branch](#selecting-the-wap-branch)).
