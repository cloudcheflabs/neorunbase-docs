# pg-wire TLS

NeorunBase exposes its PostgreSQL wire protocol over a TLS-capable endpoint so that clients can connect with `sslmode=require` (or stronger) without any change to driver or application code. TLS on the SQL surface is **optional** and **dynamically enabled** — the cluster ships in plaintext mode and turns on TLS the moment an operator uploads a certificate through the Admin UI. Every Coordinator picks up the new bundle without a restart, and existing plaintext connections are not interrupted.

## How clients negotiate TLS

NeorunBase implements the standard PostgreSQL `SSLRequest` startup negotiation:

1. The client sends a special 8-byte `SSLRequest` message before the regular `StartupMessage`.
2. The Coordinator replies with a single byte: `S` if a TLS certificate is installed, `N` if not.
3. On `S`, the client and the Coordinator perform a TLS 1.2 / 1.3 handshake on the same TCP connection. All subsequent pg-wire traffic — auth, queries, results, terminate — is encrypted.
4. On `N`, the client either downgrades to plaintext (`sslmode=prefer`) or aborts (`sslmode=require`).

This is the same flow used by upstream PostgreSQL, CockroachDB, YugabyteDB, and pgvector-compatible servers, so any PostgreSQL-compatible driver works out of the box.

## Cluster-wide certificate management

The TLS certificate and private key live in the Coordinator's encrypted metadata store, not on individual hosts. Operators upload the bundle once and it propagates automatically.

- **Upload through the Admin UI.** The dedicated "pg-wire TLS" page (Governance section in the sidebar) accepts a PEM-encoded certificate chain plus a PKCS#8 unencrypted private key. The leaf certificate's subject, issuer, validity window, signature algorithm, and SHA-256 fingerprint are displayed for verification before activation.
- **Encrypted at rest.** The bundle is envelope-encrypted with a KMS key before being written to the leader's metadata RocksDB. The plaintext private key never touches disk in the clear.
- **Synchronous propagation.** The leader broadcasts the metadata snapshot to every non-leader Coordinator over the internal protocol. Each peer receives the new bundle, atomically swaps its in-memory `SSLContext`, and starts answering `SSLRequest` with `S` on every new connection.
- **Hot-swappable.** Replacing the certificate is the same upload action with new PEM material. Already-handshaken TLS sessions stay alive on the prior context until they close on their own; new connections use the new context. There is no PG-wire downtime during a rotation.
- **Removable.** Clearing the bundle reverts the cluster to plaintext-only — `SSLRequest` returns `N` again.

## Accepted formats

To keep the operational surface small NeorunBase requires these specific formats:

- **Certificate chain** — one or more `-----BEGIN CERTIFICATE-----` blocks in a single PEM stream. The first block is treated as the leaf; subsequent blocks form the intermediate chain.
- **Private key** — unencrypted PKCS#8 (`-----BEGIN PRIVATE KEY-----`), RSA or EC. Both are routinely produced by the standard ACME clients (Let's Encrypt, certbot, AWS ACM-PCA, internal CAs).

Older formats are rejected with a clear conversion hint:

- **PKCS#1 RSA** (`-----BEGIN RSA PRIVATE KEY-----`) → convert with `openssl pkcs8 -topk8 -nocrypt -in key.pem -out key-pkcs8.pem`.
- **SEC1 EC** (`-----BEGIN EC PRIVATE KEY-----`) → same `openssl pkcs8 -topk8 -nocrypt` conversion.
- **Encrypted PKCS#8** (`-----BEGIN ENCRYPTED PRIVATE KEY-----`) → re-export with `-nocrypt`.

The leaf certificate's public key is checked against the uploaded private key before the bundle is accepted, so a mismatched pair fails fast at upload time rather than at the first client handshake.

## Security model

- **In-flight encryption** — pg-wire TLS encrypts the client → Coordinator hop. It is independent from the cluster's at-rest envelope encryption and the always-on KMS-encrypted internal protocol between Coordinators and Data Nodes.
- **Server-side TLS only** — mutual TLS (mTLS) and client-certificate authentication are not yet supported. Authentication on the SQL endpoint continues to use the existing IAM password / access-key flow.
- **No `require` mode yet** — a Coordinator with TLS enabled still accepts plaintext clients. Force-TLS (reject plaintext) is on the roadmap; today this is enforced at the network layer (e.g., listening only on a private subnet).
- **Master key dependency** — the bundle's at-rest encryption uses the same KMS hierarchy as every other persisted secret in the cluster. A lost master key makes the on-disk bundle unrecoverable, but the operator can simply re-upload through the Admin UI after rebuilding the cluster.

## Operator checklist

To enable TLS on a running cluster:

1. Obtain a certificate for the hostname clients use to connect. For a NeorunBase fleet behind an NGINX reverse proxy that is the public hostname (see [NGINX Reverse Proxy](../installation/nginx-proxy.md)). For direct connections, it is each Coordinator's reachable hostname.
2. If the private key is not already PKCS#8 unencrypted, convert it: `openssl pkcs8 -topk8 -nocrypt -in key.pem -out key-pkcs8.pem`.
3. Open the Admin UI, sign in as an administrator, and visit the **pg-wire TLS** page.
4. Paste the certificate chain into "Certificate (PEM)" and the converted PKCS#8 key into "Private Key", then click **Install**.
5. Verify with a TLS-required client: `psql "host=<coordinator> port=5432 user=admin dbname=neorunbase sslmode=require" -c "SELECT 1"`.

To rotate, repeat steps 1–4 with new material; existing connections continue undisturbed. To disable TLS cluster-wide, click **Remove** on the same page — `SSLRequest` reverts to `N` and clients with `sslmode=prefer` fall through to plaintext.

## Interaction with NGINX

When NeorunBase sits behind NGINX (or HAProxy) for multi-Coordinator load balancing, the proxy operates in `stream` (TCP passthrough) mode for the pg-wire port. The TLS handshake is end-to-end between the client and the Coordinator that the proxy routes the new connection to — the proxy never decrypts pg-wire traffic. The certificate therefore lives on the Coordinator (uploaded via the Admin UI), not on NGINX. See [NGINX Reverse Proxy](../installation/nginx-proxy.md) for the recommended layout.
