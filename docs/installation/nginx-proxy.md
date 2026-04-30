# NGINX Reverse Proxy

NeorunBase is a distributed database with multi-Coordinator high availability — every Coordinator can serve PostgreSQL wire protocol and Admin HTTP requests independently. In production this means clients should not point at a single Coordinator; they should hit a reverse proxy that load-balances across all Coordinators: connection-level TCP load balancing for the SQL endpoint, plus HTTP load balancing for the admin surface.

NGINX is the recommended proxy. Open-source NGINX is sufficient — the `ngx_stream_core_module` needed for PG wire load balancing has shipped in OSS NGINX since 1.9.0 and is included in the official `nginx:alpine` and distro packages (verify with `nginx -V 2>&1 | grep -- --with-stream`).

## Surfaces

| Surface | Protocol | Coordinator Port | Default Property | NGINX Module | TLS Termination at NGINX |
| --- | --- | --- | --- | --- | --- |
| Admin UI / REST API | HTTP/1.1 | `8080` | `neorunbase.admin.http.port` | `http` | Recommended |
| PostgreSQL wire | Raw TCP (PG wire protocol) | `5432` | `neorunbase.coordinator.pg.port` | `stream` | Pass-through only |

> NeorunBase supports pg-wire TLS but the certificate lives on the **Coordinator**, not on NGINX. The TLS handshake is end-to-end between the client and the Coordinator that the proxy routes the connection to — NGINX `stream` is a transparent TCP passthrough that never decrypts pg-wire traffic. (NGINX `stream` does not speak the PostgreSQL `SSLRequest` framing, so it cannot terminate pg-wire TLS even if you wanted it to.) Upload the cert + key once through the Admin UI and every Coordinator behind the proxy picks it up; see [pg-wire TLS](../features/pg-wire-tls.md). When TLS is not yet enabled cluster-wide, the proxy carries plaintext pg-wire — operate that mode over a trusted network (VPC, private subnet, VPN). The Admin UI is plain HTTP and is the appropriate surface for public TLS termination at NGINX.

## How the Load Balancing Works

PG wire is a connection-stateful binary protocol: a single TCP connection carries one session — its prepared statements, transactions, session variables, and temporary tables — and that session must remain on the Coordinator that opened it. NGINX `stream` therefore load-balances **per connection**, not per query. Each new TCP connection from a client is placed on a Coordinator (round-robin or `least_conn`); every query on that connection then runs through the same Coordinator until the client closes it.

Throughput multiplies because the Coordinators handle requests in parallel: distributed query planning, shard fan-out to Data Nodes, distributed transaction coordination, and Admin HTTP traffic all run on whichever Coordinator the connection landed on. The proxy's only job is to spread new connections evenly and to take a Coordinator out of rotation when it stops accepting connections (`max_fails` / `fail_timeout`).

For this model to actually balance, **clients must use a connection pool**. A single long-lived connection pinned to one Coordinator gives no benefit from the proxy. Most JDBC drivers, application connection pools (HikariCP, pgbouncer, etc.) open multiple connections — that is when the round-robin spreads work.

## Reference Configuration

The configuration below mirrors the canonical `nginx.conf` shipped with NeorunBase under `tests/`. It assumes two Coordinators, `coord-1` and `coord-2`, each running on their default ports.

```nginx
worker_processes auto;

events {
    worker_connections 1024;
}

# ----------------------------------------------------------------------------
# stream {} — PG wire (PostgreSQL protocol) load balancing on TCP :5432
# ----------------------------------------------------------------------------
stream {
    upstream pg_backends {
        # least_conn picks the Coordinator with the fewest active connections
        # at the moment a new connection arrives — a better fit than plain
        # round-robin when sessions are long-lived.
        least_conn;

        server coord-1.internal:5432 max_fails=3 fail_timeout=10s;
        server coord-2.internal:5432 max_fails=3 fail_timeout=10s;
    }

    server {
        listen 5432;
        proxy_pass             pg_backends;

        proxy_connect_timeout  10s;
        # PG sessions can be long-lived; do not impose an aggressive idle cap.
        proxy_timeout          1h;
    }
}

# ----------------------------------------------------------------------------
# http {} — Admin UI / REST API load balancing on HTTP/1.1 :8080
# ----------------------------------------------------------------------------
http {
    # Admin HTTP request body cap is governed by
    # neorunbase.admin.http.max.content.length (default 10 MB). Lift the
    # NGINX-side cap so it does not become the bottleneck.
    client_max_body_size 0;

    upstream admin_backends {
        least_conn;
        server coord-1.internal:8080 max_fails=3 fail_timeout=10s;
        server coord-2.internal:8080 max_fails=3 fail_timeout=10s;
        keepalive 16;
    }

    server {
        listen 80;

        # Coordinator-to-coordinator log tailing and large metric exports
        # may run longer than the NGINX defaults (60s). Raise read timeouts
        # so streamed responses do not get cut off.
        proxy_connect_timeout  60s;
        proxy_send_timeout     3600s;
        proxy_read_timeout     3600s;
        send_timeout           3600s;

        # Reuse upstream keepalive connections.
        proxy_http_version 1.1;
        proxy_set_header   Connection "";

        # Disable buffering for streamed admin responses (log tails, metric
        # ranges) so they reach the client without extra latency.
        proxy_request_buffering off;
        proxy_buffering         off;

        location / {
            proxy_pass http://admin_backends;

            proxy_set_header Host              $host;
            proxy_set_header X-Real-IP         $remote_addr;
            proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }
}
```

### Adding TLS for the Admin UI

In production the public-facing edge should be HTTPS. Replace the `server { listen 80; ... }` block above with the following, and optionally add an HTTP→HTTPS redirect:

```nginx
server {
    listen 443 ssl http2;
    server_name neorunbase.example.com;

    ssl_certificate     /etc/nginx/tls/neorunbase.fullchain.pem;
    ssl_certificate_key /etc/nginx/tls/neorunbase.key;
    ssl_protocols       TLSv1.2 TLSv1.3;
    ssl_ciphers         HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;

    client_max_body_size 0;

    proxy_connect_timeout  60s;
    proxy_send_timeout     3600s;
    proxy_read_timeout     3600s;
    send_timeout           3600s;
    proxy_http_version 1.1;
    proxy_set_header   Connection "";
    proxy_request_buffering off;
    proxy_buffering         off;

    location / {
        proxy_pass http://admin_backends;
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

server {
    listen 80;
    server_name neorunbase.example.com;
    return 301 https://$host$request_uri;
}
```

The Admin UI uses JWT-based authentication (signed with the master key — see `neorunbase.admin.jwt.expiry.ms`). Tokens are stateless, so any load-balancing strategy is safe; sticky sessions are not required.

## Health Checks

Open-source NGINX uses **passive** health checks: a Coordinator is taken out of rotation after `max_fails` connection or 5xx errors within `fail_timeout`. NeorunBase Coordinators register and de-register from ZooKeeper as ephemeral nodes, so within the cluster a downed Coordinator's shard map view stays consistent — but NGINX's passive ejection still needs to react to client-visible failures. The example above sets `max_fails=3` and `fail_timeout=10s`, which is a reasonable starting point; tune downward for faster failover, upward for noisier networks.

If you need **active** health probes against an admin liveness endpoint, that is provided by NGINX Plus's `health_check` directive in the `stream` and `http` blocks, or by third-party modules such as `nginx_upstream_check_module` for OSS NGINX.

## Operational Notes

- **Rolling upgrades.** Drain a Coordinator by marking its line in the `upstream` block as `down` and reloading NGINX (`nginx -s reload`). New PG wire connections and admin requests skip the drained backend; existing PG sessions stay on it until they close. Once the Coordinator is restarted on the new version, remove the `down` flag and reload again.
- **Multiple Admin endpoints.** If you scale beyond two Coordinators, just add more `server` lines to both `upstream` blocks. NeorunBase's leader election (metadata + KMS leadership) is independent of which Coordinator a client request lands on; non-leader Coordinators forward write-side admin operations to the leader internally over `neorunbase.coordinator.internal.port` and proxy log-tail requests via the admin HTTP proxy (`neorunbase.admin.http.proxy.read.timeout.ms`).
- **Connection pooling.** Make sure application clients use a real connection pool with multiple connections — that is what makes `least_conn` actually spread work. A pool of 1 connection ignores the proxy entirely.
- **PG wire encryption.** Enable pg-wire TLS by uploading a certificate through the Admin UI; every Coordinator in the cluster picks up the bundle and answers `SSLRequest` with `S` on the next connection (see [pg-wire TLS](../features/pg-wire-tls.md)). The TLS handshake terminates on the Coordinator, not on NGINX, so the cert covers connections through the proxy as well as direct connections to a Coordinator. Until TLS is enabled cluster-wide, keep the SQL endpoint on a private network. The internal Coordinator↔Data Node protocol is always KMS-encrypted (`neorunbase.kms.encrypt.internal.protocol`); the proxy hop only ever sees client→Coordinator traffic.

## Putting It Together

```text
                   ┌──────────────────────────────────┐
   :443 (HTTPS) ─▶ │  NGINX                           │
                   │   http {}    Admin UI / REST     │ ─▶  coord-1:8080  coord-2:8080
                   │   stream {}  PG wire             │ ─▶  coord-1:5432  coord-2:5432
   :5432 (TCP)  ─▶ │                                  │
                   └──────────────────────────────────┘
                              │
                              │  private network
                              ▼
                   ZooKeeper ensemble + Data Nodes
```

- Clients (`psql`, JDBC, pgAdmin, applications) connect to `neorunbase.example.com:5432`. When pg-wire TLS is enabled the handshake is end-to-end between the client and whichever Coordinator the proxy lands the connection on; otherwise traffic is plaintext and should stay on a trusted network. Either way, the proxy spreads new connections across all Coordinators.
- Operators reach the Admin UI over HTTPS at `https://neorunbase.example.com/`. NGINX terminates TLS and load-balances admin requests across all Coordinators.
- NGINX is the only component that needs a public certificate; everything behind it stays on the private network.
