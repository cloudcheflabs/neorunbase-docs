# Site Replication (DR)

NeorunBase 1.0.0 introduces **cross-site (disaster-recovery) replication**: a leader-coordinator on the **primary** site streams its WAL to one or more **standby** sites in near-real time, so a failover site can keep serving reads (and, after promotion, writes) when the primary site is lost.

Site replication is distinct from [intra-cluster replication](replication-ha.md). Intra-cluster replication keeps multiple copies of each shard inside the **same** cluster, behind the same coordinators, sharing the same KMS. Site replication crosses cluster boundaries — every standby site has its own ZooKeeper ensemble, its own coordinator quorum, its own data nodes, and its own KMS.

The 1.0.0 release ships the wire path, the leader-only streamer, the admin surface, and **automatic DDL replication** — a successful `CREATE TABLE` / `DROP TABLE` on the primary leader is shipped to peer sites so their catalogs mirror the primary without a manual DDL (`QueryExecutor.shipDdlToPeers`, `neorunbase-coordinator/.../QueryExecutor.java`). It does **not** yet ship an automatic WAL-tail tap on the production **DML** commit path — no `INSERT` / `UPDATE` / `DELETE` is auto-captured into a peer outbox. The only producers of data-plane records in 1.0.0 are the DDL hook above and the `test-enqueue` admin endpoint. v1 is the foundation: it is fully observable, exercised end-to-end in CI, and ready for operators to wire DML into bespoke flows via the test-enqueue hook. The receiver, wire path, and outbox all handle INSERT/UPDATE/DELETE today — only the automatic primary-side capture of committed DML is outstanding.

## The model

```
+----------------------- primary site (role=primary) ---------+
|                                                             |
|   client → coord (leader) ──► WalManager.write(rec)         |
|                                       │                     |
|                                       ▼                     |
|                          SiteReplicationService.enqueue()   |
|                                       │                     |
|                          per-peer outbox (in-memory)        |
|                                       │                     |
|                          site-replication-streamer-<peer>   |
|                                       │                     |
+---------------------------------------┼---------------------+
                                        │  REPLICATE_WAL_BATCH_REQ
                                        ▼  (TCP, opcode 300)
+--------------------- standby site (role=standby) -----------+
|                                                             |
|        data-node (port 7100)                                |
|        DataNodeRequestHandler.handleReplicateWalBatch       |
|                ├─ skip if originSiteId == localSiteId       |
|                ├─ INSERT/UPDATE → shard.put                 |
|                ├─ DELETE        → shard.delete              |
|                └─ PREPARE/COMMIT/ROLLBACK → seq ack only    |
|                                                             |
+-------------------------------------------------------------+
```

The diagram shows the intended full data path. In 1.0.0, `enqueue()` is fed by two producers only — the automatic DDL hook (`CREATE TABLE` / `DROP TABLE`) and the `test-enqueue` admin endpoint; the `WalManager.write` → `enqueue` tap for committed DML is the outstanding 1.1.0 piece.

The streamer is **leader-only** — only the elected leader coordinator runs the per-peer threads. Followers carry the same code but their `leaderSupplier` returns `false`, so the `streamLoop` sleeps in a tight idle. Leader handover is sticky-aware: the new leader picks up streaming from its own in-memory queue (which is empty on a cold leader); records that were buffered on the *old* leader but not yet shipped are *lost on cold restart*. v1 accepts this trade-off — the recovery flow on the standby is "restore from the latest standby-side backup, then resume streaming", which closes any gap the in-memory queue could have introduced.

## Wire format

Each batch is a single `REPLICATE_WAL_BATCH_REQ` (opcode `300`). The response is `REPLICATE_WAL_BATCH_RES` (opcode `301`).

The codec lives in `com.cloudcheflabs.neorunbase.common.replication.SiteReplicationCodec`. The body layout is:

```
[ siteIdLen : int ]
[ siteId    : utf-8 ]      ← header: where this batch came FROM
[ fromSeq   : long ]
[ recordCount : int ]
[ recordLen : int ]
[ record    : bytes ]      ← WalRecord wire format, repeated × recordCount
...
```

The receiver decodes the header, stamps every decoded `WalRecord` with `originSiteId = batch.originSiteId`, then applies them in order. The `originSiteId` is a **transient** field on `WalRecord` — it is **not** persisted into the local WAL's binary format, only carried in memory. This guarantees that:

- A record received from site B and applied locally never re-enters site B's outbox (loop prevention at the enqueue side).
- A standby promoted to primary later does not "leak" the upstream `originSiteId` into its own WAL bytes — its own newly-issued records get a fresh `originSiteId = localSiteId`.

The response body is a JSON object:

```json
{
  "appliedCount": 7,
  "lastAppliedSeq": 1042,
  "skippedCount": 0,
  "failedCount": 0,
  "originSiteId": "site-a"
}
```

`appliedCount` is the number of records the receiver mutated state for. `skippedCount` is the loop-guard count — records dropped because `originSiteId == localSiteId`. `lastAppliedSeq` is the highest sequence among applied records, which the streamer surfaces via `/admin/api/site-replication/status`.

## Why the protocol layer skips encryption for this opcode

NeorunBase's internal protocol normally encrypts every payload with the cluster's `internal-protocol` KMS key. But site replication crosses **distinct** KMS instances — site A's `internal-protocol` key is unknown to site B. If the receiver tried to decrypt the payload with its own key, it would fail with `Tag mismatch!`.

`InternalMessage.encode()` therefore skips KMS encryption for `REPLICATE_WAL_BATCH_REQ` / `REPLICATE_WAL_BATCH_RES`. v1 ships these two opcodes as **plaintext at the protocol layer**.

This is acceptable for v1 because:

1. The wire still carries an `originSiteId` header that lets receivers reject foreign or self-loop batches.
2. The receiver requires the caller to present a configured peer's `accessKey` / `secretKey` (in v1 these are configured at both ends but not yet enforced over the wire; **v1.1** will add HMAC challenge-response over the same opcode).
3. Production deployments are expected to terminate cross-site replication links over **TLS** (stunnel, mTLS at the kernel, or a service mesh). The protocol layer's own encryption is intra-cluster's defence in depth, not the only defence.

If you operate without TLS between sites, treat v1 site replication as untrusted-transport — verify it stays inside a private network you control.

## Configuration

Site replication is configured on the **primary** with a JSON document that lists peers. The configuration is persisted in `RocksDbMetadataStore` (key `replication.site.config`) and survives coordinator restart.

### Identity: `neorunbase.site.id`

Every node in a cluster reports the same `neorunbase.site.id`. It is a free-form string — recommended values are short (`site-a`, `seoul`, `dc1`) because it is sent on every batch header.

If unset, it defaults to `neorunbase.cluster.id` so existing clusters carry on with no change in behaviour (they look like single-site primaries that have no peers configured, which is the v1 default).

```text
-Dneorunbase.site.id=site-a
```

Set this **identically** on every coordinator and data node in a cluster — heterogeneous site IDs inside one cluster will confuse the loop guard.

### Admin REST surface

All endpoints are under `/admin/api/site-replication/`, require `Authorization: Bearer <admin-token>`, and reject the password-change bootstrap token (the receiver is intentionally locked behind the same password-rotation gate as the rest of the admin surface).

#### `GET /admin/api/site-replication/config`

Returns the current configuration. `secretKey` is **always masked** — the response substitutes the boolean field `secretKeySet`.

```json
{
  "enabled": true,
  "role": "primary",
  "batchMaxRecords": 100,
  "batchMaxBytes": 1048576,
  "tickMs": 200,
  "peers": [
    {
      "siteId": "site-b",
      "internalEndpoint": "site-b-data:7100",
      "accessKey": "admin",
      "secretKeySet": true
    }
  ]
}
```

#### `POST /admin/api/site-replication/config`

Apply a new config. The streamer is stopped, the in-memory outboxes are rebuilt for the new peer set, and the streamer is restarted. Idempotent.

| Field | Type | Required | Notes |
|---|---|---|---|
| `enabled` | bool | yes | Master switch. When `false`, `enqueue()` is a no-op and the streamer threads idle. |
| `role` | enum | yes | `primary` or `standby`. Cosmetic in v1 (the streamer runs unconditionally on the leader regardless of role); v1.1 will use it to gate the WAL-tail tap on primary-only. |
| `batchMaxRecords` | int | no, default 100 | Max records the streamer drains per tick before shipping. |
| `batchMaxBytes` | int | no, default 1 MiB | Soft cap on serialized batch size. v1 uses it as a guideline, not a hard split. |
| `tickMs` | int | no, default 200 | Streamer idle interval when the outbox is empty. Lower = lower replication lag, higher = lower CPU when idle. |
| `peers[].siteId` | string | yes | Peer's `neorunbase.site.id`. Used as a key for per-peer status. |
| `peers[].internalEndpoint` | string | yes | `host:port` of one of the peer's data nodes. Any data node works — the receiver runs on every data node. |
| `peers[].accessKey` / `secretKey` | string | yes | v1: stored, not yet enforced over the wire. v1.1: HMAC challenge-response. |

Returns `{"status":"ok"}`.

#### `GET /admin/api/site-replication/status`

Live status snapshot:

```json
{
  "enabled": true,
  "role": "primary",
  "leader": true,
  "totalEnqueued": 1042,
  "totalSent": 1042,
  "totalFailed": 3,
  "peers": {
    "site-b": {
      "queueDepth": 0,
      "lastAppliedSeq": 1042
    }
  }
}
```

- `leader` — `true` only on the elected leader coordinator. Standby coordinators report `false` and `totalEnqueued`/`totalSent` of `0`.
- `totalEnqueued` — records the local `enqueue()` accepted. Counted **once per call** regardless of fan-out across peers.
- `totalSent` — sum over all peers of records the receiver acknowledged via `appliedCount`. Includes records the receiver dropped via the loop guard.
- `totalFailed` — records the streamer retried because the receiver returned an error or the RPC failed. The records are pushed back at the **tail** of the in-memory queue, not the head, so a long-running outage can re-order a tail by up to `tickMs × queueDepth`.
- `peers.<siteId>.queueDepth` — how many records are currently pending shipment.
- `peers.<siteId>.lastAppliedSeq` — the highest sequence the peer has acknowledged. Track this with your monitoring system to alarm on replication lag.

#### `POST /admin/api/site-replication/test-enqueue`

Test hook (intended for operators verifying connectivity, not for production traffic). Accepts a JSON body that mirrors `WalRecord`:

```json
{
  "txId": "tx-smoke-1",
  "sequence": 1001,
  "type": "INSERT",
  "shardId": 0,
  "table": "public.sr_smoke",
  "key": "cGsx",
  "data": "cm93MQ=="
}
```

`key` / `data` are base64 — the receiver treats them as opaque bytes. `type` is one of `INSERT` / `UPDATE` / `DELETE` / `PREPARE` / `COMMIT` / `ROLLBACK`.

Returns `{"status":"enqueued"}`. Watch `totalSent` and `peers.<siteId>.lastAppliedSeq` move on the next status poll.

## Operator workflow

### Bringing up a new standby

1. Cold-restore the standby site from the primary's most recent backup. NeorunBase ships [Backup & Restore](backup-restore.md) — use `/admin/api/backup/restore` on the standby with the primary's archive. This brings the standby's RocksDB shards, IAM, and metadata to a consistent point-in-time snapshot.
2. Bring the standby up with its own distinct `neorunbase.site.id` (e.g. `site-b`).
3. On the **primary**, POST the standby's data-node endpoint to `/admin/api/site-replication/config`:

   ```bash
   curl -X POST https://primary-coord/admin/api/site-replication/config \
        -H "Authorization: Bearer $ADMIN_TOKEN" \
        -H 'Content-Type: application/json' \
        -d '{
          "enabled": true,
          "role": "primary",
          "peers": [{
            "siteId": "site-b",
            "internalEndpoint": "site-b-data-1.dr.example.com:7100",
            "accessKey": "admin",
            "secretKey": "<peer-secret>"
          }]
        }'
   ```

4. Verify with `GET /admin/api/site-replication/status` — `enabled:true`, `leader:true`, and `peers.site-b.queueDepth` initially `0` (DDL you issue on the primary will flow automatically, but no `INSERT`/`UPDATE`/`DELETE` traffic is auto-captured in 1.0.0 — see "What 1.0.0 ships").
5. Use the test-enqueue endpoint to confirm the wire path works end-to-end. The standby's data-node log should show one `REPLICATE_WAL_BATCH` apply entry per shipped batch.

### Monitoring lag

Poll `/admin/api/site-replication/status` from your monitoring system (Prometheus exporter, observability pipeline, etc.):

- **Replication lag alarm** — alarm when `peers.<siteId>.queueDepth > N` for more than M minutes, or when `peers.<siteId>.lastAppliedSeq` stops advancing while `totalEnqueued` advances.
- **Failure alarm** — alarm when `totalFailed` derivative exceeds a threshold (e.g. ≥10 failures per minute sustained).
- **Leader flip** — `leader:true` on the wrong coordinator suggests a leader-election storm. Cross-reference with the [Cluster Operations](../operations/operations.md) leader-election runbook.

### Promoting a standby to primary

v1 promotion is **manual**:

1. Stop the (failed) primary if it is reachable, or fence it at the network layer.
2. On the chosen standby: change its config role to `primary` (it is cosmetic in v1 but reserves the field for v1.1's hot-path gate). Add the *former* primary as a peer if and when it returns, so the standby can stream back to it.
3. Re-point your application traffic to the standby's pg-wire/Arrow Flight endpoints. NeorunBase does not own DNS — that is your orchestration layer's job.
4. Audit any in-flight transactions on the primary that did not reach the standby (compare the standby's last applied sequence to the primary's last issued sequence, if the primary is recoverable). v1 cannot guarantee zero-loss promotion on a sudden primary failure — it is an **asynchronous** replication design.

### Network requirements

The streamer dials peers on their **data-node internal port** (default `7100`). Open this port between sites:

```text
primary-coords/* → standby-datanodes/* on 7100/tcp
```

If you also want the standby to stream back to the primary (active-active is not supported in v1 — there is no conflict-resolution layer — but operators sometimes pre-open the reverse direction so promotion is a config-only operation), open the symmetric path too.

Latency: the streamer's `tickMs` (default 200 ms) is the **idle** wake interval, not the per-RPC latency. A batch leaves immediately as records enqueue. End-to-end replication lag is dominated by the network RTT between sites + the receiver's RocksDB write latency. The wire ships records as a single batch, so a coast-to-coast RTT of ~70ms produces ~70ms lag for the first record and amortises down for the rest of a batch.

## What 1.0.0 ships

The 1.0.0 release ships the full wire path plus **automatic DDL replication**. Automatic DML replication is **not** in 1.0.0 — the data plane is exercised through the test-enqueue hook:

| Capability | Status |
|---|---|
| Wire path (opcode 300/301, codec) | ✅ |
| Leader-only streamer threads | ✅ |
| Per-peer in-memory outbox | ✅ |
| Loop prevention via `originSiteId` | ✅ |
| Admin REST (config, status, test-enqueue) | ✅ |
| Admin UI page (`/site-replication`) | ✅ |
| Plaintext at protocol layer; terminate over TLS in production | ✅ |
| **DDL replication** — successful `CREATE TABLE` / `DROP TABLE` on the primary leader auto-ships and is re-applied on the standby via its local leader (`QueryExecutor.shipDdlToPeers` → receiver `ddlForwarder`) | ✅ |
| **Receiver-side apply** of INSERT/UPDATE/DELETE + PREPARE/COMMIT/ROLLBACK (the last three are seq-ack markers, no RocksDB side-effect) — the standby *can* apply DML it receives | ✅ |
| **Receiver-side index reconcile** — after DML apply, ANN (HNSW) + FTS (Lucene) caches on the standby are invalidated so the next query rebuilds incrementally from the freshly replicated rows; CSR adjacency rebuilds on `CsrCompactor`'s own tick | ✅ |
| **Automatic production DML WAL-tail tap** — every committed `INSERT`/`UPDATE`/`DELETE` auto-enqueues to peers | ❌ not in 1.0.0 — only the DDL hook and `test-enqueue` produce records; automatic DML capture is a follow-up |

Out of scope for 1.0.0 and tracked as follow-ups:
- **Automatic DML tail-tap** — auto-capture every committed INSERT/UPDATE/DELETE into the peer outbox. Today only DDL auto-ships; DML must be driven through the test-enqueue hook (the receiver/wire/outbox already handle DML, so this is a primary-side capture hook, not a protocol change).
- Persistent outbox (resume after coordinator restart without losing in-flight records).
- HMAC challenge-response over the wire (currently relies on TLS at the transport layer).
- Reverse direction (peer → primary) for active-active (requires conflict resolution — research phase).
- Per-peer outbox size cap with backpressure (today: bounded queue with drop-oldest on overload).
- Catch-up from a checkpoint after long outage (today: rebuild standby via [backup-restore](backup-restore.md), then resume).

## Configuration reference

| JVM property | Default | Description |
|---|---|---|
| `neorunbase.site.id` | value of `neorunbase.cluster.id` | This cluster's site identity. Stamped onto every outbound batch. |

There are no other system properties for site replication — all knobs are in the JSON config persisted via `/admin/api/site-replication/config`.

## Troubleshooting

### `totalSent` never advances after `enqueue`

Likely causes:

- `enabled:false` in the persisted config. Re-POST with `enabled:true`.
- The receiver returns `peer rejected batch: ...` — check the standby's coordinator/data-node logs. Common cause: `localSiteId not configured on this data node` (the standby was started without `-Dneorunbase.site.id=...` and falls back to a UUID-based clusterId — the loop guard then treats every record as foreign-origin and applies them, but if the standby's data node lacks the WAL apply column families it falls back further).
- DNS / firewall — confirm the streamer's connect log line: `Connected to data node at <peer-endpoint>`. Absence of that line means the connect attempt itself fails. Check the **primary's coordinator log** for `Connect to ... timed out after 10s` from `SiteReplicationService`.

### `totalFailed` is non-zero but `totalSent` keeps moving

Transient network blips. The streamer re-queues the failed batch at the tail and continues. Investigate only if `totalFailed` derivative is sustained.

### Receiver logs `Tag mismatch!`

This should not happen as of 1.0.0 — the encryption-skip for opcodes 300/301 is the fix. If you see it, you are running a mixed version (one site upgraded, the other not) — both sites must run ≥ 1.0.0 for the encryption-skip to apply on both encode and decode.

### Receiver logs `decode failed: ...`

Codec version mismatch — both sites must be on the same protocol version. The codec layout is stable at 1.0.0; downgrade or upgrade in lockstep across sites.

## See also

- [Replication & High Availability](replication-ha.md) — intra-cluster shard replication, the other layer of fault tolerance.
- [Backup & Restore](backup-restore.md) — the bootstrap mechanism for a new standby and the cold-restore path during promotion.
- [NIC Silent-Fail Safety](nic-fail-safety.md) — the underlying transport hardening that keeps the streamer from wedging on a half-open peer.
- [Cluster Operations](../operations/operations.md) — leader-election runbook + general operational guidance.
