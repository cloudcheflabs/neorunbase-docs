# Admin UI

NeorunBase includes a built-in web console for monitoring and operating the cluster. Every coordinator
serves it at `http://<coordinator>:<neorunbase.admin.http.port>/admin/` (default port `8080`), and every
coordinator answers every page: requests that need the leader (IAM changes, repair, backup, catalog
changes, …) are forwarded to it, and pages that show leader-held state (repair / rebalance / reshard
progress, backup history, site-replication status) are answered by the leader. The console therefore
behaves the same behind a load balancer whichever coordinator a request lands on.

Every page and action is covered by the browser end-to-end suite in the product repository
(`tests/ui-e2e`), which checks each action both on the page and against the server.

## Signing In

The login form takes a user ID and password. Depending on the [Single Sign-On](sso.md) settings it also
offers:

- **Single Sign-On** / **Sign In with SAML** buttons when OIDC or SAML is enabled — they redirect to the
  identity provider and come back signed in;
- **Use your directory account instead** when LDAP / Active Directory is enabled — the same form then
  checks the credentials against the directory (**Sign In with Directory**).

**First sign-in.** The built-in `admin` account starts with the password `admin`. Signing in with it
opens a mandatory *Password Change Required* dialog; until the password is changed, the session's token
can call nothing but the password-change endpoint. The new password must be at least 6 characters (the
server additionally refuses weak passwords) and must differ from the current one.

The header shows the signed-in user and the role (**Administrator** or **User**); the sidebar footer
shows the product version. **Sign out** (the icon at the top right) clears the session in the browser.
Pages can be opened directly by URL or reloaded (`/admin/nodes`, `/admin/iam`, `/admin/query`, …); an
expired or missing session shows the login form instead of the page.

## Who Can Use What

The console and the admin REST API are for **administrators** — users whose groups carry a policy with
`"Action": "*"` on `"Resource": "*"` (for example the built-in `admin-group`).

A signed-in user who is not an administrator sees only **Dashboard** and **Topology** (read-only). Every
other page shows *Administrators only*, and the server enforces the same rule: apart from
`GET /admin/nodes/*`, `GET /admin/topology`, `GET /admin/zookeeper` and `GET /admin/monitoring/metrics`,
every admin endpoint answers **403** to a non-administrator. That includes the REST SQL endpoints
(`/admin/query/execute`, `/v1/api/*`) and graph traversal (`/admin/graph/*`), which run SQL outside
pg-wire's per-user policy evaluation — non-administrators query data over pg-wire with their own
credentials, where their IAM policies, column masks and row filters apply.

## Dashboard

The Dashboard (`/admin/`) counts the live Coordinators and Data Nodes and charts the last hour of CPU
usage (%), memory usage (MB), query TPS and DML TPS (1-minute rates). **Overall / Coordinators / Data
Nodes** (or a click on a count card) filters the charts to one kind of node. The page refreshes every 30
seconds. The series come from the leader Coordinator's metrics store (`GET /admin/monitoring/metrics`);
see [Operations → Metrics](../operations/operations.md#metrics) for collection interval and retention.

## Topology

The Topology page (`/admin/nodes`) is the live map of the cluster. It refreshes every 10 seconds and
on **Refresh**, and has four sections:

### ZooKeeper

Every server in `neorunbase.zookeeper.server.list`, as the Coordinator serving the page sees it:

| Column | Meaning |
|---|---|
| Server | `host:port` from the connect string. The server this Coordinator's own session is connected to is tagged *this coordinator's session*. |
| Server id | the server's own id (`myid`) as it reports it to `conf`, or `server id unknown (...)` with the reason — see [Server ids](#server-ids-myid) |
| Role | `leader`, `follower`, `observer` or `standalone` (a single-server ZooKeeper) |
| Status | **Online** — the server answered; **Down** — it refused the connection or did not answer in time (the error is shown under the badge); **Not serving** — it is running but has no quorum; **Unknown** — it refuses the status command (see below) |
| Version | ZooKeeper server version |
| Connections | client connections open on that server |
| Latency | request latency min / avg / max, in milliseconds |
| Znodes | number of znodes |
| Zxid | last committed transaction id |

Below the table, the Coordinator's own ZooKeeper session: state (`CONNECTED`, `CONNECTING`, …),
session id, negotiated session timeout and the server it is connected to. The header shows how many
servers are up, so a lost ensemble member is visible before it costs quorum.

The Coordinator gathers this by sending each server ZooKeeper's `srvr` four-letter command (and
`conf` for the server id), in parallel, waiting at most `neorunbase.admin.zookeeper.probe.timeout.ms` (default `2000`) per server.
`srvr` is in ZooKeeper's default `4lw.commands.whitelist` (3.5 and later) and the ZooKeeper bundled
with NeorunBase allows it (`conf/zk/zoo.cfg`: `4lw.commands.whitelist=srvr,conf,ruok,stat,mntr`). An ensemble configured with a
whitelist that leaves out `srvr` shows its servers as **Unknown**; add `srvr` to the whitelist to see
them. The Coordinator must be able to reach the client port (default `2181`) of every listed server,
which it already needs to coordinate.

Any Coordinator answers — the view does not go through the leader — so during a failover the page
still shows which ZooKeeper servers are reachable from the Coordinator you are looking at.

#### Server ids (myid)

Every server of an ensemble needs its own id (`myid`). Two servers with the same id — typically
left behind by a scale-out or a rebuild that reused an instance number — or servers that do not
agree on who is in the ensemble break it **silently**: ZooKeeper raises no error, and clients just
see an unstable or split service. The section therefore also asks each server that answers with the
`conf` four-letter word, which reports the server's own `serverId` and the ensemble membership it
was configured with (its `server.N=` lines). The **Server id** column shows the id, or
`server id unknown (...)` with the reason, and a red line above the table names each problem:

| Warning | Meaning |
|---|---|
| `Duplicate ZooKeeper server id 2 on zk2:2181, zk3:2181 — the ensemble is misconfigured; give every server a unique myid` | two or more configured servers report the same id |
| `ZooKeeper server zk4:2181 reports server id 4, which is not in its own ensemble membership (1, 2, 3) — check its myid` | a server's id is not among the `server.N` entries it was configured with |
| `ZooKeeper servers disagree about the ensemble membership: zk1:2181, zk2:2181 list server ids 1, 2, 3; zk3:2181 lists server ids 2 — every server must be configured with the same server.N list` | the servers were configured with different ensembles |
| `ZooKeeper server id unknown on zk1:2181, zk2:2181, zk3:2181 (conf not in 4lw.commands.whitelist; add conf there and restart that ZooKeeper server) — a duplicate myid cannot be ruled out` | the id could not be read (here `conf` is not whitelisted; also `server is down`, `server is not serving requests`) — the check is never reported as passing for those servers |
| `ZooKeeper servers zk1:2181, zk2:2181 run standalone — a standalone server is a separate single-server ZooKeeper, not part of one ensemble with the other configured servers` | the connect string lists several servers but some of them run standalone |

A healthy ensemble shows each server's id and no warning; a single standalone ZooKeeper reports id
`0` and is not a warning. To fix a duplicate or a mismatch, correct the `myid` file in the server's
data directory (`ZOO_MY_ID` for the `zookeeper` Docker image) and its `server.N=` lines so that
every server has a unique id and all of them list the same ensemble, then restart that server.

`conf` is **not** on ZooKeeper's default whitelist. Add it next to `srvr`, for example
`4lw.commands.whitelist=srvr,conf` (`ZOO_4LW_COMMANDS_WHITELIST=srvr,conf` for the Docker image),
and restart that ZooKeeper server: ZooKeeper reads the whitelist only at startup.
Without it every server's id reads `server id unknown (conf not in 4lw.commands.whitelist; add conf there and restart that ZooKeeper server)`.
The ZooKeeper bundled with NeorunBase (`conf/zk/zoo.cfg`) ships with
`4lw.commands.whitelist=srvr,conf,ruok,stat,mntr`, so `conf` is allowed there.

`tests/e2e-zk-myid.sh` verifies all of this on a real three-server ensemble, on the API and on the
Topology page from a non-secure origin (`tests/ui-e2e/45-zk-myid.mjs`).

### Coordinators

Node id, host, pg-wire port, admin port, and the **Leader** / **Follower** role. **Logs** streams that
Coordinator's log in place: **LIVE / PAUSED** stops and resumes the stream, the refresh button reloads the
last 200 lines, and the view keeps the last 1,000 lines. Logs can be read only from nodes registered in
the cluster — the coordinator relays the request to that node and to no other host.

### Data Nodes

Node id, host, internal port, the number of shard replicas the node holds, and its host / rack / zone
labels (see [Topology-Aware Shard Placement](topology-aware-placement.md)). Expanding a node shows
each data disk's capacity, used and available space. **Logs** streams the node's log.

### Disk Repair

The disk repair service's status (enabled, in progress, last cycle, repaired / failed / scanned
counts, dead disks being tracked) with **Scan** — list healthy and dead disks and how long each dead
disk has been gone — and **Trigger** — run a repair cycle now. See
[Reactive Shard Repair](reactive-shard-repair.md).

### REST endpoints

The page reads these admin endpoints (Bearer token from `POST /admin/auth/login`):

| Endpoint | Returns |
|---|---|
| `GET /admin/topology` | `coordinators` (with `isLeader`) and `dataNodes` (with `disks` and `shardCount`) |
| `GET /admin/zookeeper` | `connectString`, `rootPath`, `servers[]`, `warnings[]` and `session` — see below |
| `GET /admin/nodes/coordinators`, `GET /admin/nodes/datanodes` | the registered nodes, without leader / disk / shard details |

`GET /admin/zookeeper` example (three-server ensemble, one server stopped):

```json
{
  "connectString": "zk-1:2181,zk-2:2181,zk-3:2181",
  "rootPath": "/neorunbase",
  "servers": [
    {"host": "zk-1", "port": 2181, "status": "UP", "mode": "leader", "version": "3.9.1",
     "zxid": "0x100000045", "connections": 4, "outstanding": 0,
     "latencyMin": 0.0, "latencyAvg": 0.43, "latencyMax": 7.0, "nodeCount": 57, "error": null,
     "serverId": 1, "serverIdNote": null,
     "membership": {"1": "zk-1:2888:3888:participant;0.0.0.0:2181",
                    "2": "zk-2:2888:3888:participant;0.0.0.0:2181",
                    "3": "zk-3:2888:3888:participant;0.0.0.0:2181"}},
    {"host": "zk-2", "port": 2181, "status": "DOWN", "mode": null, "version": null, "zxid": null,
     "connections": null, "outstanding": null, "latencyMin": null, "latencyAvg": null,
     "latencyMax": null, "nodeCount": null, "error": "connect timed out",
     "serverId": null, "serverIdNote": "server id unknown (server is down)", "membership": {}},
    {"host": "zk-3", "port": 2181, "status": "UP", "mode": "follower", "version": "3.9.1", "serverId": 3, "...": "..."}
  ],
  "warnings": ["ZooKeeper server id unknown on zk-2:2181 (server is down) — a duplicate myid cannot be ruled out"],
  "session": {"connected": true, "state": "CONNECTED", "sessionId": "0x1000a1b2c3d0004",
              "sessionTimeoutMs": 30000, "connectedServer": "zk-1:2181"}
}
```

`status` is `UP`, `DOWN`, `NOT_SERVING` or `UNKNOWN`; fields a server did not report are `null`.
`warnings` is empty when every server reported a unique id and all of them agree on the membership.

## Shards

The Shards page (`/admin/shards`) has four tabs and a refresh button:

| Tab | Shows |
|---|---|
| **By Node** | total nodes (live), total shards, average per node; each data node with its shard count — expand a node to see, per table, how many shards it holds and their ids. A node with no shards is tagged *NEW*, a dead one *OFFLINE*. |
| **By Table** | each table with its shard count and per-node distribution; expand a table for the replica placement of every shard. |
| **Detail** | pick a table to list every shard with its row count, primary and replicas, plus the table's replication factor and approximate total rows. |
| **Manage** | every table with shard key, shard count, replication factor, status and column count. Click the name to list columns and indexes. |

The **Manage** actions:

- **Lock / Unlock** — make the table read-only (writes are refused over pg-wire with a read-only error)
  or writable again (`POST /admin/tables/<table>/readonly`).
- **RF** — change the replication factor; the table is read-only while replicas are added or removed
  (`POST /admin/tables/<table>/replicas`).
- **Reshard** — redistribute the table over a new shard count. All rows are rehashed and moved, the
  table is read-only meanwhile, and the dialog shows the phase, rows migrated and elapsed time until
  *Resharding complete* (`POST` / `GET /admin/tables/<table>/reshard`).

## Shard Repair

The Shard Repair page (`/admin/repair`):

- **Reactive Shard Repair (continuous)** — the leader's background worker that compares replica row
  counts and re-replicates a diverging replica: enabled, running, queue depth, last scan, detected /
  repaired / failed counts. See [Reactive Shard Repair](reactive-shard-repair.md).
- **Scan Dead Nodes** — lists dead data nodes, how long they have been gone, and every affected shard
  with its surviving replicas and whether it is recoverable.
- **Trigger Repair** (with confirmation) — re-replicates the shards of dead nodes onto live ones; the
  status panel shows the cycle result and repaired / failed / scanned counts.
- **Shard Rebalancing** — the current per-node shard distribution and imbalance; **Preview Plan** lists
  the moves a rebalance would make, **Trigger Rebalance** (with confirmation) runs them.

## IAM

The IAM page (`/admin/iam`) manages local identities. See [Identity and Access Management](iam.md) and
the [IAM Policy Reference](iam-policy.md).

- **Users** — **New User** (user ID + password). Each row shows the user's groups (**Join** adds the
  user to a group) and access keys. **New Keypair** generates an access key (optionally with an expiry
  date) and shows the access key ID and secret in a banner; pg-wire accepts the access key as the user
  name and the secret as the password, with the user's policies. Each key has a download icon (CSV with
  the key pair) and a revoke button. The `admin` user cannot be deleted.
- **Groups** — **New Group**; expand a group to see its members (remove one with its ×) and policies
  (**Attach Custom** attaches a policy, × detaches it). Policies attached to a group apply to every
  member.
- **Policies** — **New Policy** with a policy name and JSON document; **Governance Template** fills the
  document from a template (Read-Only Analyst, App Read-Write, Schema Admin, Column-Restricted Read,
  Row-Level Filter, Iceberg Read-Only, …). Invalid JSON is rejected before anything is sent. Hover a
  policy to edit or delete it (`AdministratorAccess` cannot be deleted).
- **STS Sessions** — generate temporary credentials from a parent access key with a lifetime (hours /
  minutes) and an optional session name. The banner shows the temporary access key, secret and session
  token (the eye buttons reveal them, the copy buttons copy them, **Download CSV** saves them). A pg-wire
  client signs in with the temporary access key as the user name and `<secret>::<session token>` as the
  password. The table lists sessions with their time left; sessions can be downloaded or revoked, and
  **Clear Expired** removes expired ones. Revoked or expired credentials no longer sign in.
- **IAM Authority Mode** — **standalone** (this cluster owns its IAM) or **federated** (ontul is the
  authority: users, groups and policies are pulled from ontul and local IAM becomes read-only). Federated
  mode takes the ontul admin URL, a service token (stored, never shown again — the label says
  *— set*), and the sync interval. See [IAM Federation (ontul)](iam-federation.md). The mode is kept
  across restarts and applies on every coordinator.

## Single Sign-On

The Single Sign-On page (`/admin/sso`) connects the console and pg-wire to an identity provider —
**OpenID Connect**, **SAML 2.0** and **LDAP / Active Directory** tabs (a green dot marks an enabled
provider) plus the group mapping shared by all three. Secrets (OIDC client secret, LDAP bind password,
SAML private key) are write-only: the page shows *· stored* and an empty field keeps the stored value.

- **Save** stores the settings on the cluster and applies them on every coordinator.
- **Test connection** checks the selected provider and reports the result.
- **Discard changes** reloads the stored settings.
- SAML: **Download SP metadata** saves the service-provider metadata to hand to the IdP; **Generate
  keypair** creates the SP signing/encryption keypair (the private key never leaves the cluster); **Read
  metadata** imports the IdP's metadata document and fills in entity ID, sign-on URL and certificate.

See [Single Sign-On](sso.md) for each setting.

## Security & KMS

The Security & KMS page (`/admin/kms`):

- **Create New Master Key** — enter a key ID (letters, digits, `.`, `_`, `-`) and **Generate Key**, then
  confirm. An ID that already exists is refused — rotating is a separate, explicit action.
- The key list shows each key's current version and last rotation; **Manage Key** opens the key's
  version lifecycle (each version's state — ACTIVE, RETIRED — and creation time) and **Rotate Key Now**
  (with confirmation) adds a new active version and retires the previous one, which keeps decrypting.
  Key material is never returned by the API.
- **Change Admin Password** — change the signed-in administrator's password (current + new + confirm).

See [Encryption at Rest](encryption.md).

## pg-wire TLS

The pg-wire TLS page (`/admin/tls`) shows the installed certificate (subject, issuer, validity, SHA-256
fingerprint, signature algorithm, serial, who uploaded it and when; it turns amber within 30 days of
expiry) and installs or replaces one: paste the PEM certificate chain and the unencrypted PKCS#8 private
key and click **Install** / **Replace**. A key that does not match is refused. **Remove** (with
confirmation) turns TLS off cluster-wide. Changes reach every coordinator without a restart; the private
key is never returned. See [pg-wire TLS](pg-wire-tls.md).

## S3 Backup

The S3 Backup page (`/admin/backup`):

- **Configuration** — enable scheduled backups, S3 endpoint, region, bucket, prefix, access key, secret
  key (write-only: *(unchanged — paste to overwrite)*), interval (minutes), retention (days), optional
  cron, and path-style addressing. The form is enabled once the stored configuration has loaded.
  **Save**, **Test connection** (uses the values in the form, including a secret typed but not yet
  saved), and **Backup now** (only while backups are enabled; it answers when the backup finishes).
- **Available backups (from S3)** — the backups recorded in the bucket (they survive a leader restart),
  with file count and uploaded bytes. **Restore** → **Stage restore** downloads the chosen backup into a
  staging directory on every coordinator and data node and shows, per node, the files and bytes staged.
  The swap happens on the next restart of all containers.
- **History** — recent runs (when, trigger, status, files, uploaded bytes, reused files, error), with the
  run in progress.

See [Backup & Restore](backup-restore.md).

## Site Replication

The Site Replication page (`/admin/site-replication`) shows the live status of the leader's WAL streamer
(enabled, role, leader, records enqueued / sent / failed, per-peer queue depth and last applied
sequence) and edits its configuration: enabled, role (`primary` / `standby`), batch limits, tick, and the
peer sites (site ID, internal endpoint, access key, secret key). A peer's secret is write-only — the
label says *(already set — leave blank to keep)* and saving without retyping keeps it. See
[Site Replication (DR)](site-replication.md).

## Query Runner

The Query Runner (`/admin/query`) executes SQL as an administrator. Type a statement and click
**Execute** or press **Ctrl/Cmd + Enter**; **Tab** indents. A `SELECT` shows the column names with their
types, `NULL` values, and 50 rows per page (previous / next); DDL and DML show the command tag and
affected rows; errors are shown in red. The last 20 statements are kept in the browser as a history —
click an entry to load it, **Clear All** to empty it. **Clear** empties the editor.

## Catalogs

The Catalogs page (`/admin/catalogs`) lists the built-in `lakebase` catalog (no edit or delete) and every
registered Iceberg catalog with its connector, type, URI and default namespace.

- **New Catalog** — name, catalog type (`rest`), REST URI, warehouse (the Polaris catalog name), security
  (`OAUTH2` / `NONE`), default namespace, OAuth client id / secret, extra properties (for Polaris:
  `prefix=<catalog>,header.Polaris-Realm=POLARIS,scope=PRINCIPAL_ROLE:ALL`), S3 endpoint / region /
  access key / secret key / path-style access, an optional WAP branch, and index table patterns.
- **Edit** (pencil) — changes only what you change: secret fields are blank with *(unchanged)* and keep
  the stored value; clearing the WAP branch switches the catalog back to `main`.
- **Delete** (bin) — refused while the catalog still contains a namespace with tables.

Secrets are never returned: the secret key and client secret are shown as `********`. A registered
catalog is queried as `<catalog>.<namespace>.<table>` over pg-wire. See [Catalogs](catalogs.md).

## Iceberg

The Iceberg page (`/admin/iceberg`) configures the cluster's Iceberg connection, which the sync job
writes to. The header shows **ENABLED** / **DISABLED**.

- **Connection** — catalog type (`polaris` or `none`), warehouse, default namespace; the Polaris URI,
  OAuth token endpoint, catalog name, realm, client ID / secret and scope; S3 endpoint, region, access
  key, secret key and path-style access. Secrets are write-only (the eye button reveals what you type).
  S3 access always uses these static keys — Polaris STS / vended credentials are not used. **Save**
  applies the connection on every coordinator and registers it as the SQL catalog named after its
  Polaris catalog, so its tables are queryable as `<polaris-catalog>.<namespace>.<table>` right away.
  The connection is kept across restarts.
- **Table Browser** — namespaces, the tables in a namespace, and a table's columns (type, nullable) with
  ready-to-run `SELECT` / `MERGE` statements.
- **Sync Job** — **Start / Stop** the background sync on every coordinator (each syncs the tables
  assigned to it); table filters (`schema.table`, `*` wildcards; empty = all tables) added as chips and
  saved with **Save Filters**; auto-start on restart. **Sync Activity** lists synced tables across all
  coordinators with status, last update and error; **Sync All Dirty** makes every coordinator sync the
  tables written since their last sync and reports how many it synced; **Sync Now** syncs one table.
- **Query Guide** — example SQL for reading Iceberg tables, joining them with LakeBase tables, and
  `MERGE` in both directions.

See [Iceberg Integration](iceberg-integration.md).

## Kafka

The Kafka page (`/admin/kafka`):

- **Configuration** — enable the Kafka consumer, **Add Group**, and per group: brokers, topics, group ID
  (defaults to `neorunbase-<name>`), target table, format (JSON), batch size, poll timeout, auto offset
  reset, max consumers and extra properties (`key=value,…`). Credential-like extra properties
  (`*password*`, `*secret*`, `*jaas*`, `*token*`, …) are shown as `********`; saving with the mask in
  place keeps the stored value. **Remove** deletes a group; **Save** applies the configuration.
- **Consumer Status** — per group: state (RUNNING, STARTING, STOPPED, ERROR), topics, target table, last
  activity, rows ingested, error count and last error, merged over the data nodes that run it; **Start**
  and **Stop** control the group.

See [Kafka Integration](kafka-integration.md).

## REST API

Everything the console does goes through the admin REST API on the same port, authenticated with the
Bearer token from `POST /admin/auth/login` (`{"username": "...", "password": "..."}`), so it can be
automated. All endpoints except the read-only cluster view listed under
[Who Can Use What](#who-can-use-what) require an administrator.
