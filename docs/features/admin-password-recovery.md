# Admin Password Recovery

NeorunBase ships with a built-in recovery channel that lets an operator
reset the `admin` user's password **without stopping the coordinator**,
even when no one remembers the current password.

Recovery uses a local Unix domain socket — there is no HTTP back-door, no
network endpoint, no recovery URL. Authentication is performed by the
operating system: only a process that already shares the coordinator's
filesystem identity can open the socket.

## When to use

- The admin password was forgotten or rotated out of the password manager.
- An automation script needs to provision a known admin password during
  first-time setup, without going through the web UI.
- A new operator is onboarded and you need to hand them an admin credential.

For day-to-day password changes (the user remembers the old password and
wants a new one), use the Admin UI's **Change Password** screen instead —
that path requires the old password and does not flag the user for forced
rotation.

## How it works

```
┌────────────────────┐   JSON over UDS    ┌──────────────────────┐
│  neorunbase-cli    │ ─────────────────▶ │ Coordinator (running)│
│  iam:reset-password│                    │   AuthManager        │
└────────────────────┘                    │   .adminResetPwd()   │
         ▲                                │                      │
         │ stdout: new password           │  saveToDb()          │
         │ (one-time)                     │  cluster sync push   │
                                          │  audit log append    │
                                          └──────────────────────┘
```

| Property | Value |
|---|---|
| **Socket path** | `data/admin.sock` (mode `600`) |
| **Authentication** | OS file permission — same user as the coordinator process |
| **Network surface** | none — Unix domain socket only |
| **Downtime** | none — applied in-process on the live coordinator |
| **Cluster sync** | automatic — leader pushes the new state to followers and data nodes |
| **Audit log** | `data/iam-audit/reset.log` (mode `600`, append-only) |
| **Post-reset state** | `requirePasswordChange = true` (forced rotation on next login) |

## Quick start

The simplest invocation lets the coordinator generate a strong
20-character password and print it to stdout. The new password must be
changed on the admin's next login (the `requirePasswordChange` flag is
set automatically).

```bash
# Inside the coordinator host or container:
bin/neorunbase-cli.sh iam:reset-password

# Or with the packaged docker image:
docker exec neorunbase-coordinator /app/bin/neorunbase-cli.sh iam:reset-password
```

When stdout is piped or redirected, the password alone is printed — suitable
for capturing in automation:

```bash
NEW_PW=$(docker exec neorunbase-coordinator /app/bin/neorunbase-cli.sh iam:reset-password)
```

## Input modes

| Mode | Command | When to use |
|---|---|---|
| **Coordinator-generated** | `neorunbase-cli.sh iam:reset-password` | Default. Strong random password printed once on stdout. |
| **Explicit** | `neorunbase-cli.sh iam:reset-password --new-password 'My!Pass'` | Automation that knows the desired value. Beware: argv may show up in `ps`. |
| **Stdin** | `echo 'My!Pass' \| neorunbase-cli.sh iam:reset-password --new-password -` | Automation that wants to avoid argv exposure. |
| **Interactive** | `neorunbase-cli.sh iam:reset-password --interactive` | Operator at a TTY. Prompts for password twice with no echo. |

Resetting a different user is also supported:

```bash
bin/neorunbase-cli.sh iam:reset-password --user some-user --new-password 'NewPass123'
```

## Configuration

The recovery socket is enabled by default. The relevant `NeorunConfig`
fields:

```yaml
adminSocketEnabled: true
adminSocketPath:    ./data/admin.sock
iamAuditDir:        ./data/iam-audit
```

The CLI resolves the socket path in this order:

1. `--socket /path/to/admin.sock` command-line flag
2. `NEORUNBASE_ADMIN_SOCKET` environment variable
3. `./data/admin.sock` fallback

## Security model

**1. The socket is OS-gated.**
At startup the coordinator creates `data/admin.sock` with mode `600`
(owner read/write only). Even other unprivileged users on the same host
cannot connect. There is no token, no shared secret, no network listener.

**2. The audit log records every reset.**
Every successful reset appends a JSON line to `data/iam-audit/reset.log`
(mode `600`). The plaintext password is **never** logged — only the first
8 characters of its hash, the user, whether it was coordinator-generated,
and the OS user that invoked the CLI.

```json
{"ts":"2026-05-23T16:00:46.922Z","event":"iam.reset-password","user":"admin","generated":true,"hashFp":"OYV/Ojf/","invokedAs":"root"}
```

**3. The new password is exposed exactly once.**
For coordinator-generated passwords, the plaintext is returned only on
the single CLI invocation that triggered the reset. It is not retransmitted.
Treat scrollback and shell history accordingly — or use stdin input mode
to avoid argv exposure entirely.

**4. Forced rotation on next login.**
After reset, the user is flagged `requirePasswordChange = true`. The next
successful login forces the user through the change-password flow, so a
temporary password used by the operator is immediately replaced by a
password only the user knows.

**5. The coordinator must be running.**
Because the recovery channel is in-process, the coordinator must be alive
for the CLI to connect. This is intentional: RocksDB requires an exclusive
lock, so an offline edit would either conflict with a running coordinator
or need a complex stale-lock recovery. With this design, the only way to
reset is to be on the host *and* have the coordinator running *and* share
its filesystem identity.

## Limitations

- **Coordinator must be running.** If the coordinator is down, this CLI
  cannot help. Bring the coordinator back up first, then run the CLI.
- **No knowledge factor.** Any process that shares the coordinator's
  filesystem identity can invoke the CLI. In multi-tenant or shared-shell
  environments, restrict shell access to the coordinator accordingly. A
  future enhancement may add an opt-in "recovery key" requirement for an
  additional knowledge factor.

## Related

- [Identity and Access Management](iam.md)
- [IAM Policy Reference](iam-policy.md)
- [Encryption](encryption.md)
