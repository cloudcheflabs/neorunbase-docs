# IAM Policy Reference

NeorunBase's authorization layer accepts AWS IAM-style JSON policy documents. This page is the reference for writing them: the JSON schema, the resource format, every supported action, and the evaluation rules — including the advanced column- and row-level access controls.

For a high-level overview of the IAM subsystem (users, groups, access keys) see [Identity and Access Management](iam.md).

---

## Policy Document Structure

A policy is a JSON object with a `Version` and a list of `Statement` objects.

```json
{
  "Version": "2024-01-01",
  "Statement": [
    {
      "Sid": "AllowSelectOnOrders",
      "Effect": "Allow",
      "Action": "pg:Select",
      "Resource": "db:table:public.orders"
    }
  ]
}
```

| Field | Required | Description |
|-------|----------|-------------|
| `Version` | Yes | Policy language version. Use `"2024-01-01"`. |
| `Statement` | Yes | One or more statements. |
| `Statement[].Sid` | No | Free-form identifier for the statement. |
| `Statement[].Effect` | Yes | `"Allow"` or `"Deny"`. Case-insensitive. |
| `Statement[].Action` | Yes | A single action string or list. Wildcards (`*`, `?`) supported. |
| `Statement[].Resource` | Yes | A single resource string or list. Wildcards supported. |
| `Statement[].Columns` | No | Column-level whitelist (on Allow) or blacklist (on Deny). See [Column-Level ACL](#column-level-acl). |
| `Statement[].MaskedColumns` | No | Map of column → SQL mask expression applied on `SELECT`. See [Column Masking](#column-masking). |
| `Statement[].Condition` | No | SQL `WHERE` fragment for row-level filtering. See [Row-Level ACL](#row-level-acl). |

---

## Resource Format

Resources use a flat colon-delimited form:

```
db:<resource-type>:<id>
```

| Resource type | Example | Used for |
|---------------|---------|----------|
| `table` | `db:table:public.orders` | All DML and table-level DDL |
| `schema` | `db:schema:analytics` | `CREATE SCHEMA`, `DROP SCHEMA` |
| `catalog` | `db:catalog:iceberg-east` | `CREATE CATALOG`, `ALTER CATALOG`, `DROP CATALOG` |

### How resources are computed at runtime

NeorunBase parses the SQL statement and constructs the resource string before evaluating policies. The mapping is fixed:

| SQL | Resource passed to evaluator |
|-----|------------------------------|
| `SELECT … FROM analytics.orders` | `db:table:analytics.orders` |
| `INSERT INTO users …` (no schema qualifier) | `db:table:public.users` |
| `MERGE INTO target USING source …` | `db:table:<target>` (primary) **and** `db:table:<source>` (dual `pg:Select` check) |
| `CREATE SCHEMA analytics` | `db:schema:analytics` |
| `CREATE CATALOG "iceberg-east" …` | `db:catalog:iceberg-east` |

When a table reference is unqualified (no `schema.` prefix), NeorunBase uses the connection's current database, falling back to `public` when none is set. Always match this in your policies — a policy granting `db:table:analytics.users` will **not** match a query against `users` from a connection whose default schema is `public`.

### Wildcards

- `*` matches any run of characters (including empty).
- `?` matches exactly one character.
- Without `*` or `?`, comparison is **case-insensitive exact match**.

Common patterns:

| Pattern | Matches |
|---------|---------|
| `db:table:*.*` | Every fully qualified table |
| `db:table:public.*` | Every table in the `public` schema |
| `db:table:*.audit_*` | Every table whose name starts with `audit_` in any schema |
| `db:table:iceberg.*.*` | Tables under the Iceberg catalog namespace (3-part names) |
| `db:schema:team_*` | Every schema whose name starts with `team_` |
| `db:catalog:*` | Every catalog (gates who may manage catalogs) |
| `*` | Everything |

---

## Action Catalog

All actions live under the `pg:` namespace. Wildcards work at any depth (`pg:*`, `pg:Create*`).

### DML

| Action | Triggered by | Resource type |
|--------|--------------|---------------|
| `pg:Select` | `SELECT`, plus dual check on `USING` source of `MERGE` | `db:table:` |
| `pg:Insert` | `INSERT` | `db:table:` |
| `pg:Update` | `UPDATE` | `db:table:` |
| `pg:Delete` | `DELETE` | `db:table:` |
| `pg:Merge` | `MERGE INTO` (target) | `db:table:` |

### DDL — Tables

| Action | Triggered by | Resource type |
|--------|--------------|---------------|
| `pg:CreateTable` | `CREATE TABLE` | `db:table:` |
| `pg:DropTable` | `DROP TABLE` | `db:table:` |
| `pg:AlterTable` | `ALTER TABLE` | `db:table:` |
| `pg:CreateIndex` | `CREATE INDEX`, `CREATE UNIQUE INDEX` | `db:table:` (the indexed table) |

### DDL — Schemas

| Action | Triggered by | Resource type |
|--------|--------------|---------------|
| `pg:CreateSchema` | `CREATE SCHEMA` | `db:schema:` |
| `pg:DropSchema` | `DROP SCHEMA` | `db:schema:` |

### DDL — Catalogs

Catalog DDL manages named [catalogs](catalogs.md) (the built-in `lakebase` plus external Iceberg catalogs).

| Action | Triggered by | Resource type |
|--------|--------------|---------------|
| `pg:CreateCatalog` | `CREATE CATALOG` | `db:catalog:` |
| `pg:AlterCatalog` | `ALTER CATALOG` | `db:catalog:` |
| `pg:DropCatalog` | `DROP CATALOG` | `db:catalog:` |

### Statements that bypass authorization

`SET` commands and transaction control (`BEGIN`, `COMMIT`, `ROLLBACK`) are not authorized; they execute without policy evaluation.

---

## Evaluation Rules

For each statement issued by a connection, NeorunBase:

1. Looks up every policy attached to every group the authenticated user belongs to.
2. Classifies the SQL into a `pg:*` action and computes the resource string.
3. Iterates statements:
   - A statement applies only if **both** its `Action` and `Resource` match.
   - If any matching statement has `Effect: "Deny"` (and no `Columns` field — see below), the decision is **immediately DENY**.
   - Otherwise, if any matching statement has `Effect: "Allow"`, the decision is **ALLOW**.
   - If no statement matches, the decision is **ABSTAIN**, treated as **deny** (default-deny).

In short: **explicit Deny > explicit Allow > implicit Deny**.

Wildcard matching is **case-insensitive** and uses a cached regex (up to 10,000 entries).

---

## Column-Level ACL

A statement may carry a `Columns` array. Its meaning depends on `Effect`:

- **`Effect: "Allow"` with `Columns`**: a **whitelist**. Only those columns are returned by `SELECT`; `INSERT` and `UPDATE` may only touch those columns.
- **`Effect: "Deny"` with `Columns`**: a **blacklist**. Those columns are forbidden in any access. Note: a Deny statement that carries `Columns` does **not** trigger a resource-level deny — it only restricts the listed columns, so the surrounding Allow can still apply to the rest of the row.

Column names are case-folded to lowercase. Multiple Allow whitelists are unioned, then any Deny blacklist columns are removed.

### Examples

Reveal only safe columns of `public.users`:

```json
{
  "Version": "2024-01-01",
  "Statement": [{
    "Sid": "LimitedColumns",
    "Effect": "Allow",
    "Action": "pg:Select",
    "Resource": "db:table:public.users",
    "Columns": ["id", "name", "email", "department"]
  }]
}
```

Block reading the `salary` column even if other policies grant the row:

```json
{
  "Version": "2024-01-01",
  "Statement": [{
    "Sid": "HideSalary",
    "Effect": "Deny",
    "Action": "pg:Select",
    "Resource": "db:table:hr.*",
    "Columns": ["salary"]
  }]
}
```

---

## Column Masking

Where `Columns` removes a column entirely, **`MaskedColumns`** keeps the column in the result but replaces its value with the output of a SQL expression. A statement may carry a `MaskedColumns` object mapping each column name to a SQL mask expression:

```json
"MaskedColumns": {
  "salary": "0",
  "ssn": "'REDACTED'"
}
```

On `SELECT`, each masked column returns the **evaluated expression** instead of the raw value. Masking applies both to explicit column lists and to `SELECT *`.

Mask expressions are ordinary SQL — a literal, a function call, or a partial mask. For example:

| Mask expression | Effect |
|-----------------|--------|
| `0` | Returns the constant `0` for every row. |
| `'REDACTED'` | Returns the string `REDACTED`. |
| `'***-**-' \|\| right(ssn, 4)` | Partial mask keeping only the last four characters. |

### Precedence

- **Deny beats Mask.** A column that is explicitly **Denied** (via a `Columns` blacklist) is never masked — it is simply not accessible.
- **Higher `Sid` wins.** When two statements mask the *same* column, the statement with the **larger `Sid`** wins, so the outcome is deterministic.

### Principal substitution

Mask expressions may reference the current principal and have these tokens substituted before evaluation:

| Token | Substituted with |
|-------|------------------|
| `${user.userId}` | The authenticated user's id. |
| `${user.id}` | The authenticated user's id. |
| `${user.groups}` | The principal's groups. |

### Example

Allow reading `hr.employees`, but mask `salary` to `0` and `ssn` to a redacted literal:

```json
{
  "Version": "2024-01-01",
  "Statement": [{
    "Sid": "MaskSensitiveHR",
    "Effect": "Allow",
    "Action": "pg:Select",
    "Resource": "db:table:hr.employees",
    "MaskedColumns": {
      "salary": "0",
      "ssn": "'REDACTED'"
    }
  }]
}
```

---

## Row-Level ACL

A statement with `Effect: "Allow"` may carry a `Condition` field — a SQL `WHERE` fragment that is OR-merged with any other matching conditions and AND-ed onto the query.

The token `${user.userId}` is substituted with the authenticated user's id (single-quoted, with internal quotes escaped). This lets a single policy serve every user while restricting each one to their own rows.

```json
{
  "Version": "2024-01-01",
  "Statement": [{
    "Sid": "OwnRowsOnly",
    "Effect": "Allow",
    "Action": "pg:Select",
    "Resource": "db:table:public.orders",
    "Condition": "owner_id = ${user.userId}"
  }]
}
```

Static condition (no substitution):

```json
{
  "Sid": "SalesOnly",
  "Effect": "Allow",
  "Action": "pg:Select",
  "Resource": "db:table:public.orders",
  "Condition": "department = 'sales'"
}
```

Row filters only apply to `Allow` statements. Multiple matching filters are joined with `OR`.

---

## Defaults

On first startup of a fresh cluster, NeorunBase seeds:

| Object | Value |
|--------|-------|
| Policy | `AdministratorAccess` — `Action: "*"`, `Resource: "*"` |
| Group | `admin-group` — bound to `AdministratorAccess` |
| User | `admin` (password `admin`, must change on first login) — member of `admin-group` |

Rotate the admin password immediately after first login.

---

## Worked Examples

### Read-only analyst

```json
{
  "Version": "2024-01-01",
  "Statement": [{
    "Sid": "ReadAllTables",
    "Effect": "Allow",
    "Action": "pg:Select",
    "Resource": "db:table:*.*"
  }]
}
```

### Application read/write on its own schema

```json
{
  "Version": "2024-01-01",
  "Statement": [{
    "Sid": "AppSchema",
    "Effect": "Allow",
    "Action": ["pg:Select", "pg:Insert", "pg:Update", "pg:Delete"],
    "Resource": "db:table:app.*"
  }]
}
```

### Schema admin (DDL + DML on one schema)

```json
{
  "Version": "2024-01-01",
  "Statement": [{
    "Sid": "SchemaAdmin",
    "Effect": "Allow",
    "Action": [
      "pg:Select", "pg:Insert", "pg:Update", "pg:Delete",
      "pg:CreateTable", "pg:DropTable", "pg:AlterTable", "pg:CreateIndex"
    ],
    "Resource": "db:table:analytics.*"
  }]
}
```

### Audit-safe: full DML, but logs are read-only

Because explicit Deny wins, the broad Allow paired with a narrow Deny is the idiomatic carve-out.

```json
{
  "Version": "2024-01-01",
  "Statement": [
    {
      "Sid": "AllowDML",
      "Effect": "Allow",
      "Action": ["pg:Select", "pg:Insert", "pg:Update", "pg:Delete"],
      "Resource": "db:table:*.*"
    },
    {
      "Sid": "DenyModifyAudit",
      "Effect": "Deny",
      "Action": ["pg:Update", "pg:Delete"],
      "Resource": ["db:table:*.audit_*", "db:table:*.*_logs"]
    }
  ]
}
```

### Namespace isolation (team workspace)

```json
{
  "Version": "2024-01-01",
  "Statement": [{
    "Sid": "TeamWorkspace",
    "Effect": "Allow",
    "Action": "*",
    "Resource": ["db:table:team_a.*", "db:schema:team_a"]
  }]
}
```

### Iceberg + Relational read/write

The Iceberg catalog namespace uses three-segment table names (`iceberg.<namespace>.<table>`).

```json
{
  "Version": "2024-01-01",
  "Statement": [
    {
      "Sid": "RelationalDML",
      "Effect": "Allow",
      "Action": ["pg:Select", "pg:Insert", "pg:Update", "pg:Delete", "pg:Merge"],
      "Resource": "db:table:*.*"
    },
    {
      "Sid": "IcebergRead",
      "Effect": "Allow",
      "Action": "pg:Select",
      "Resource": "db:table:iceberg.*.*"
    }
  ]
}
```

### Catalog-level resources

Catalog DDL is authorized on `db:catalog:<name>` resources, so an Allow/Deny on `db:catalog:*` (or a specific catalog) gates who may manage [catalogs](catalogs.md).

Full access everywhere, but no one may create or manage catalogs (a broad Allow carved out by a Deny — explicit Deny wins):

```json
{
  "Version": "2024-01-01",
  "Statement": [
    {
      "Sid": "FullAccess",
      "Effect": "Allow",
      "Action": "*",
      "Resource": "*"
    },
    {
      "Sid": "NoCatalogManagement",
      "Effect": "Deny",
      "Action": ["pg:CreateCatalog", "pg:AlterCatalog", "pg:DropCatalog"],
      "Resource": "db:catalog:*"
    }
  ]
}
```

Grant catalog management on a single catalog only:

```json
{
  "Version": "2024-01-01",
  "Statement": [{
    "Sid": "ManageEastCatalog",
    "Effect": "Allow",
    "Action": ["pg:CreateCatalog", "pg:AlterCatalog", "pg:DropCatalog"],
    "Resource": "db:catalog:iceberg-east"
  }]
}
```

### Merge with read-only source

`MERGE INTO target USING source` triggers two checks: `pg:Merge` on the target and `pg:Select` on the source. Both must pass.

```json
{
  "Version": "2024-01-01",
  "Statement": [
    {
      "Sid": "MergeTarget",
      "Effect": "Allow",
      "Action": ["pg:Select", "pg:Merge"],
      "Resource": "db:table:warehouse.*"
    },
    {
      "Sid": "ReadFromStaging",
      "Effect": "Allow",
      "Action": "pg:Select",
      "Resource": "db:table:staging.*"
    }
  ]
}
```

---

## Notes & Caveats

- **Default schema is `public`.** Unqualified table references resolve to `public.<table>` for the IAM resource string. Connect with a specific database name or use fully qualified names to control this.
- **Wildcard matching is case-insensitive.** `Resource: "DB:TABLE:Public.Users"` matches `db:table:public.users`.
- **Row filters are OR-ed.** Multiple Allow statements with conditions widen access, they don't narrow it.
- **A Deny with `Columns` is not a resource deny.** It only restricts the listed columns — the row itself can still be returned by another Allow.
- **Masking yields to Deny.** A column denied via a `Columns` blacklist is never masked; it is simply inaccessible. When two statements mask the same column, the larger `Sid` wins.
- **Catalog DDL is authorized.** `CREATE`/`ALTER`/`DROP CATALOG` require `pg:CreateCatalog` / `pg:AlterCatalog` / `pg:DropCatalog` on `db:catalog:<name>`.
- **MERGE has dual authorization.** A user able to merge into a table must also be able to `pg:Select` on the source table.
- **`SET` and transaction control bypass authorization.** Use this in mind when designing policies for session-level features.
