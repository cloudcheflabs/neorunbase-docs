# Admin UI

NeorunBase includes a built-in web-based Admin UI for monitoring and managing the cluster.

## Dashboard

The Dashboard (`/admin/`) counts the live Coordinators and Data Nodes and charts the last hour of
CPU usage (%), memory usage (MB), query TPS and DML TPS (1-minute rates), per node or aggregated.
The series come from the leader Coordinator's metrics store (`GET /admin/monitoring/metrics`); see
[Operations → Metrics](../operations/operations.md#metrics) for collection interval and retention.

## Topology

The Topology page (`/admin/nodes`) is the live map of the cluster. It refreshes every 10 seconds and
on **Refresh**, and has four sections:

### ZooKeeper

Every server in `neorunbase.zookeeper.server.list`, as the Coordinator serving the page sees it:

| Column | Meaning |
|---|---|
| Server | `host:port` from the connect string. The server this Coordinator's own session is connected to is tagged *this coordinator's session*. |
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

The Coordinator gathers this by sending each server ZooKeeper's `srvr` four-letter command, in
parallel, waiting at most `neorunbase.admin.zookeeper.probe.timeout.ms` (default `2000`) per server.
`srvr` is in ZooKeeper's default `4lw.commands.whitelist` (3.5 and later) and the ZooKeeper bundled
with NeorunBase allows all four-letter words (`conf/zk/zoo.cfg`). An ensemble configured with a
whitelist that leaves out `srvr` shows its servers as **Unknown**; add `srvr` to the whitelist to see
them. The Coordinator must be able to reach the client port (default `2181`) of every listed server,
which it already needs to coordinate.

Any Coordinator answers — the view does not go through the leader — so during a failover the page
still shows which ZooKeeper servers are reachable from the Coordinator you are looking at.

### Coordinators

Node id, host, pg-wire port, admin port, and the **Leader** / **Follower** role. **Logs** streams that
Coordinator's log in place.

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
| `GET /admin/zookeeper` | `connectString`, `rootPath`, `servers[]` and `session` — see below |
| `GET /admin/nodes/coordinators`, `GET /admin/nodes/datanodes` | the registered nodes, without leader / disk / shard details |

`GET /admin/zookeeper` example (three-server ensemble, one server stopped):

```json
{
  "connectString": "zk-1:2181,zk-2:2181,zk-3:2181",
  "rootPath": "/neorunbase",
  "servers": [
    {"host": "zk-1", "port": 2181, "status": "UP", "mode": "leader", "version": "3.9.1",
     "zxid": "0x100000045", "connections": 4, "outstanding": 0,
     "latencyMin": 0.0, "latencyAvg": 0.43, "latencyMax": 7.0, "nodeCount": 57, "error": null},
    {"host": "zk-2", "port": 2181, "status": "DOWN", "mode": null, "version": null, "zxid": null,
     "connections": null, "outstanding": null, "latencyMin": null, "latencyAvg": null,
     "latencyMax": null, "nodeCount": null, "error": "connect timed out"},
    {"host": "zk-3", "port": 2181, "status": "UP", "mode": "follower", "version": "3.9.1", "...": "..."}
  ],
  "session": {"connected": true, "state": "CONNECTED", "sessionId": "0x1000a1b2c3d0004",
              "sessionTimeoutMs": 30000, "connectedServer": "zk-1:2181"}
}
```

`status` is `UP`, `DOWN`, `NOT_SERVING` or `UNKNOWN`; fields a server did not report are `null`.

## Shards

The Shards page shows replica distribution by node and by table, and per-table shard detail. The
Shard Repair page drives rebalancing (status, plan, trigger) and replica repair (status, scan,
trigger).

## Management

The Admin UI allows administrators to perform operational tasks:

- **IAM Management**: Create and manage users, groups, policies, access keys, and STS sessions. In federated mode this page is read-only and hosts the standalone ↔ federated mode toggle. See [Identity and Access Management](iam.md).
- **Single Sign-On**: Configure OIDC, SAML 2.0 and LDAP / Active Directory, import the identity provider's metadata, generate the service-provider keypair, and download the SP metadata to hand back to the provider. Settings are stored on the cluster and replicated to every coordinator, so no file edit or restart is involved. See [Single Sign-On](sso.md).
- **Security & KMS**: List the KMS key hierarchy, inspect a key's versions, create new keys, and rotate a key to a new version. See [Encryption at Rest](encryption.md).
- **pg-wire TLS**: Upload, rotate, and remove the cluster-wide PostgreSQL wire protocol certificate. Activation propagates to every Coordinator with no restart and existing connections are unaffected. See [pg-wire TLS](pg-wire-tls.md).
- **Catalogs**: List, create, edit, and drop catalogs (the built-in `lakebase` plus external Iceberg catalogs). Drop is refused for a non-empty catalog and `lakebase` cannot be removed. See [Catalogs](catalogs.md).
- **Iceberg**: Configure the Apache Polaris connection (URI, OAuth client credentials, S3 access/secret key) and monitor sync status. See [Iceberg Integration](iceberg-integration.md).
- **S3 Backup**: Configure scheduled cluster backups, view backup history, and trigger restore. See [Backup & Restore](backup-restore.md).
- **Site Replication**: Configure disaster-recovery peer sites and monitor the leader-only WAL streamer that ships committed writes to them.
- **Kafka Ingestion**: Manage Kafka consumer groups and monitor ingestion pipelines
- **Query Runner**: Execute SQL against the cluster directly from the browser, with a local query history.
- **Shard Operations**: Monitor shard distribution and health, and drive shard/disk repair progress (see [Shards](#shards) and [Disk Repair](#disk-repair))

## REST API

All operations available in the Admin UI are also accessible via a REST API, enabling automation and integration with external monitoring and management tools.
