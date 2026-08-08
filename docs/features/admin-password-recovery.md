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
| **Socket path** | `${neorunbase.base.data.dir}/admin.sock` by default, mode `600` — the live value is published to `bin/coordinator.socket` |
| **Authentication** | OS file permission — same user as the coordinator process |
| **Socket path marker** | `bin/coordinator.socket` — written when the socket binds, removed on shutdown |
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

The recovery socket is enabled by default. Every key below lives in
`conf/neorunbase.properties` and is read at coordinator startup:

```properties
# conf/neorunbase.properties
# Set false to remove the local recovery path entirely.
neorunbase.admin.socket.enabled      = true
neorunbase.admin.socket.path         = ${neorunbase.base.data.dir}/admin.sock
# Name of the file under <neorunbase.home>/bin that receives the socket path the
# coordinator actually bound to (see "How the CLI finds the socket" below).
neorunbase.admin.socket.marker.file  = coordinator.socket
# Append-only audit trail of socket operations.
neorunbase.iam.audit.dir = ${neorunbase.base.data.dir}/iam-audit
```

The socket follows `neorunbase.base.data.dir`. If you launch the coordinator with
`-Dneorunbase.base.data.dir=/var/lib/neorunbase` the socket moves to
`/var/lib/neorunbase/admin.sock` — you do not have to restate it.

### How the CLI finds the socket

Re-deriving the socket path from `conf/neorunbase.properties` is not reliable on its own:
`neorunbase.base.data.dir` can be overridden with `-D` at launch or edited after
startup, and the file does not record which value the live process used. So the
coordinator **publishes the path it actually bound to** into
`<install dir>/bin/coordinator.socket` when the socket comes up, and removes that file on
shutdown. `bin/neorunbase-cli.sh` prefers it.

Full resolution order, highest priority first:

1. `--socket /path/to/admin.sock` — read by the Java CLI, always wins.
2. `$NEORUNBASE_ADMIN_SOCKET` — if already exported in the caller's shell.
3. `<install dir>/bin/coordinator.socket` — the path published by the running coordinator.
   Used only when the file exists *and* the path in it is a live socket.
4. `neorunbase.admin.socket.path` from `conf/neorunbase.properties`, with
   `${neorunbase.base.data.dir}` expanded. A value that still contains a
   `${...}` placeholder is rejected rather than used literally.
5. `<install dir>/data/admin.sock`, then `/data/admin.sock`.

Step 3 is what makes a moved data dir work: with the socket at
`/data/admin.sock` and the properties file still saying `./data`, only the marker
knows where to connect.

To rename the marker, change one key — both ends read it:

```bash
# conf/neorunbase.properties
neorunbase.admin.socket.marker.file = neorunbase-recovery.socket
```

Restart the coordinator; it publishes `bin/neorunbase-recovery.socket`, and the CLI
picks the new name up from the same properties file.

### The master key is for the coordinator, not the CLI

`NEORUNBASE_MASTER_KEY` must be exported for the coordinator process. The start script does not pre-check it, but KMS is enabled by default and the coordinator cannot unseal its keystore without the key, so startup fails during KMS initialisation.
The variable name itself is configurable — `neorunbase.kms.master.key.env` in
`conf/neorunbase.properties` names the variable the coordinator reads:

```bash
export NEORUNBASE_MASTER_KEY='replace-with-a-32-char-or-longer-secret'
bin/start-coordinator.sh
```

`bin/neorunbase-cli.sh` does **not** need it. The CLI only opens the Unix socket and
hands the request to the running coordinator, which already holds the unsealed key,
so this works with the variable unset:

```bash
unset NEORUNBASE_MASTER_KEY
bin/neorunbase-cli.sh ping
# pong
```

If a CLI invocation complains about the key rather than the socket, you are
running a start script, not the CLI.

### Worked examples

```bash
# 1. On the host, as the same OS user that runs the coordinator:
cd /opt/neorunbase
bin/neorunbase-cli.sh ping
bin/neorunbase-cli.sh iam:reset-password

# 2. The coordinator runs as a service account and you are root:
sudo -u neorunbase /opt/neorunbase/bin/neorunbase-cli.sh iam:reset-password

# 3. Inside a container:
docker exec -it neorunbase-coordinator-1 /app/bin/neorunbase-cli.sh iam:reset-password

# 4. Data dir was relocated at launch — no extra flags needed, the CLI
#    reads the published marker:
cat /opt/neorunbase/bin/coordinator.socket
# /var/lib/neorunbase/admin.sock
bin/neorunbase-cli.sh ping

# 5. Socket in a non-standard place and no marker (the coordinator is stopped,
#    or you are on a host where the marker was cleaned up):
bin/neorunbase-cli.sh --socket /var/lib/neorunbase/admin.sock iam:reset-password

# 6. Non-interactive automation, password from stdin so it never reaches argv:
echo 'S0me!Strong!Pass' | bin/neorunbase-cli.sh iam:reset-password --new-password -
```

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
