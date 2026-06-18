# IAM Federation with ontul

NeorunBase IAM runs in one of two **authority modes**:

| Mode | Authority | Local IAM | Use when |
| --- | --- | --- | --- |
| **`standalone`** (default) | NeorunBase itself | read-write — create/edit users, groups, policies here | NeorunBase is deployed on its own |
| **`federated`** | **ontul** | **read-only** — synced from ontul; local writes are rejected | NeorunBase is used alongside ontul, which governs access for all data sources |

In a federated deployment, [ontul](https://www.cloudchef-labs.com) is the unified data engine and the single
place where access policies are authored. But an external application or dashboard often connects **directly to
NeorunBase** (over the PostgreSQL wire / JDBC) for low-latency [serving](iceberg-serving.md) — bypassing ontul.
Federation makes the policies authored in ontul apply to that direct path: NeorunBase **pulls** the IAM graph
from ontul and enforces it locally.

!!! info "Pull, not push — ontul is unmodified"
    NeorunBase *pulls* from ontul's existing admin REST API. ontul needs no plugin, no callback, and no
    awareness of NeorunBase. Enforcement happens locally in NeorunBase (no per-query round trip to ontul), so
    direct-serving stays fast.

## How it works

```
            author policy (db:table:…)              pull users+groups+policies
   operator ───────────────────────────▶ ontul ◀───────────────────────────── NeorunBase
                                                  (every sync interval, leader)
                                                         │ import (read-only)
   external app ──── pg-wire + ontul JWT ─────────────▶ NeorunBase ── enforce mask/deny/row-filter
```

1. **Author in ontul, in NeorunBase's resource format.** The operator writes the policy in ontul using
   NeorunBase's native resource syntax — `db:table:<catalog>.<ns>.<table>` for an Iceberg catalog table,
   `db:table:<schema>.<table>` for a native table — and `pg:Select` / `SELECT` actions. The ontul Admin UI's
   policy editor offers a **"NeorunBase serving policy" template** for exactly this.
2. **NeorunBase pulls** `GET /admin/iam/{users,groups,policies}` from ontul on each sync interval (leader only),
   and imports them into its replicated IAM store **verbatim** — no translation. The local bootstrap admin is
   preserved so the NeorunBase console keeps working.
3. **Enforcement** is identical to standalone mode: a query's user → effective policies → column masking, column
   deny, and row-level filters are applied on the [serving path](iceberg-serving.md), including Iceberg catalog
   tables.

### Why no translation is needed — the `db:table:` prefix

NeorunBase resources are prefixed `db:table:`; ontul's own resources are prefixed `data:table:`. The two
prefixes are distinct, so a policy authored for NeorunBase (`db:table:…`) is unambiguously distinguishable from
ontul's own policies (`data:table:…`). NeorunBase imports ontul's policies as-is; any `data:table:` policy that
is pulled simply never matches a NeorunBase query resource (which is always `db:table:…`) and is inert. This is
why federation requires **no resource rewriting** — you author NeorunBase policies in NeorunBase's format.

## Single sign-on (SSO)

The external app authenticates to NeorunBase's pg-wire endpoint with an **ontul-issued JWT** placed in the
password field. NeorunBase validates the token **locally** using a shared HMAC key (ontul's master key) — no
call back to ontul — resolves the subject to the mirrored user, and applies that user's synced policies.

Set the shared key with `neorunbase.iam.federation.ontul.jwt.key` (= ontul's `ONTUL_MASTER_KEY`). When blank,
ontul-token SSO is disabled and mirrored users have no local credential.

## Read-only guard

In federated mode every mutating IAM/STS admin request (`POST`/`PUT`/`DELETE` under `/admin/iam` or
`/admin/sts`) is rejected with **HTTP 403 — "IAM is managed by ontul"**. Reads are allowed, and the mode toggle
(`/admin/iam/mode`) stays writable so you can switch back to standalone. The federation sync is the only writer.

## Resilience

- **Leader-only** writer; followers receive the synced IAM via the normal replication path.
- **Fail-closed**: if a pull fails (ontul unreachable, token expired) the last good snapshot is kept and
  enforcement continues; new/changed policies simply aren't seen until the next successful pull.
- **Runtime toggle**: switch `standalone ↔ federated` from the Admin UI (IAM page) without a restart.

## Configuration

| Property | Default | Description |
| --- | --- | --- |
| `neorunbase.iam.mode` | `standalone` | `standalone` or `federated`. |
| `neorunbase.iam.federation.ontul.url` | *(blank)* | ontul admin base URL, e.g. `http://ontul-master:8080`. |
| `neorunbase.iam.federation.ontul.token` | *(blank)* | ontul credential for the pull — a long-lived ontul token (OTOK, sent as `Token`) or a JWT (`Bearer`, auto-detected). A read-only "IAM reader" principal. |
| `neorunbase.iam.federation.ontul.jwt.key` | *(blank)* | Shared HMAC key (= ontul's master key) to validate ontul JWTs locally for SSO. |
| `neorunbase.iam.federation.sync.interval.ms` | `60000` | Pull interval. |
| `neorunbase.iam.federation.catalog.alias` | *(blank)* | `ontulCat=neorunbaseCat` CSV — only if the same physical catalog has different logical names in each system. |

All of these are also settable at runtime from the **Admin UI → IAM** page (the mode toggle, ontul URL, token,
and sync interval), which persists them across the cluster.

## Mode switching

- **standalone → federated**: the next pull replaces the policy graph with ontul's; the local admin is kept.
- **federated → standalone**: the last-synced policies become the local, now-writable baseline.

## See also

- [Identity and Access Management](iam.md) — the IAM model (users, groups, policies).
- [IAM Policy Reference](iam-policy.md) — policy statement syntax (masking, row filters, column deny).
- [Iceberg Serving (LakeBase)](iceberg-serving.md) — the low-latency serving path RBAC is enforced on.
