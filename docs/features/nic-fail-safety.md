# NIC Silent-Fail Safety

NeorunBase's internal NIO plane survives the trickiest failure mode in distributed systems: **a peer whose NIC dies after the socket was opened, but ZooKeeper hasn't expired the session yet**. From ZooKeeper's POV the node is alive. From every neighbour's POV its TCP connection is also alive — the kernel does not surface a half-open peer for tens of seconds, sometimes minutes, while it exhausts its retransmit budget. RPCs that route to that peer hang. Membership-driven failover doesn't fire because membership is still healthy.

The 1.0.0 release bounds this gap on the **caller side** of every internal RPC — `InternalNioClient`, which every coordinator → data node and data node → coordinator call goes through. Because the caller is the party that actually hangs on a wedged peer, bounding the client is what keeps the cluster responsive. The response side (`InternalNioServer`) runs a non-blocking `Selector` reactor for its accept and read paths; its response-write path is covered below.

## Why the legacy NIO loop is dangerous

A blocking `SocketChannel`-based loop in three places hangs on a silently-dead peer:

1. **`connect()`** in blocking mode parks the caller inside the TCP three-way handshake until the OS retransmit budget drains. On Linux this is dictated by `tcp_syn_retries` and runs ~60–130 seconds by default.
2. **`channel.write()`** parks the caller the instant the kernel send buffer fills. For a peer that stopped draining, the buffer never frees up until the OS gives up — which is the *retransmit* budget, not the SYN budget. Linux `tcp_retries2` ranges from 5 to 15 minutes depending on tuning.
3. **`channel.read()`** in blocking mode is similar — no application-visible deadline.

A single hung writer takes its calling thread out of service. If that thread is the master's heartbeat-sweep loop or a worker's coordinator-pull, the entire cluster appears wedged from outside even though only one peer is sick.

## What changed in the client

`com.cloudcheflabs.neorunbase.protocol.internal.InternalNioClient`:

- **Non-blocking for its lifetime.** `connect()` opens the channel in non-blocking mode, registers `OP_CONNECT` on a per-call `Selector`, and waits for completion bounded by a 10s deadline. Then the channel **stays** non-blocking — no `configureBlocking(true)` flip on return, which used to race the reader thread.
- **Selector-driven reader.** `readLoop()` waits on `OP_READ` with a 200ms tick so an externally-set `connected=false` propagates within a tick. No more blocking `channel.read()` calls that the test harness can't shut down.
- **Bounded `sendRequest`.** Writers serialise under a `writeLock`, open a per-call `Selector`, register `OP_WRITE`, and bound the write with a 10s deadline. On timeout, `connected.set(false)` flips so the pool re-mints the next time around. The same channel can sit in the reader's selector (`OP_READ`) while a per-call writer selector (`OP_WRITE`) is active — Java NIO supports multiple selector registrations per channel as long as each selector sees it at most once.
- **SO_KEEPALIVE + TCP_NODELAY** enabled on every new socket so the OS itself eventually flags long-dead peers, independent of any application protocol.

Both deadlines are overridable per JVM:

```text
-Dneorunbase.protocol.internal.connect.timeout.seconds=10
-Dneorunbase.protocol.internal.write.timeout.seconds=10
```

## The server side

`com.cloudcheflabs.neorunbase.protocol.internal.InternalNioServer`:

- **Non-blocking selector reactor.** The listening channel registers `OP_ACCEPT`; every accepted client channel is put in non-blocking mode and registered for `OP_READ` (`accept()`). A slow or silent peer can therefore never park the server's accept/read loop.
- **Off-selector request handling.** Inbound messages are decoded on the selector thread and dispatched to a worker pool (`processRequest`), so a slow handler does not stall the reactor.
- **Response-write path is not independently deadline-bounded.** The reply is written back in `processRequest()` under `synchronized(channel)` with a plain `while (encoded.hasRemaining()) channel.write(encoded);` loop. There is no per-call `OP_WRITE` selector or write deadline on this path — the protection against a wedged peer comes from the *caller* side, `InternalNioClient`'s bounded `sendRequest` (above): when the client's own 10s write deadline trips it flips `connected=false` and re-mints the connection, so callers stop waiting on a dead peer even though the server's response write is not itself bounded.

Only the client honours `-Dneorunbase.protocol.internal.write.timeout.seconds`; the server response-write loop above does not read it.

## Failure-handling invariants

| Scenario | Old behaviour | 1.0.0 behaviour |
|---|---|---|
| Peer NIC dies mid-handshake | `connect()` hangs for OS SYN budget (~60–130s) | `SocketTimeoutException` raised at 10s |
| Peer NIC dies after socket established, mid-write | `channel.write()` hangs for OS retransmit budget (5–15 min) | `SocketTimeoutException` raised at 10s; caller sees a fast failure and can fall back |
| Peer process paused (kernel still ACKs but app doesn't drain) | Same as above — kernel send buffer fills, then writer blocks indefinitely | `Selector.select()` wakes on the 10s deadline; writer fails out cleanly |
| Reader thread in a long blocking `read()` during shutdown | Reader could not be interrupted cleanly | 200ms select tick — next loop iteration sees `connected=false` and exits |

## Verifying it on your stack

`tests/test-nic-failure-channel-health.sh` (shipped with the 1.0.0 release) drives the failure on a 2-node compose:

```bash
docker compose -f tests/docker-compose-neorunbase.yml up -d
docker pause neorunbase-worker-2          # silent NIC sim
./tests/test-nic-failure-channel-health.sh    # expect PASS
docker unpause neorunbase-worker-2
```

The test hammers the surviving master's admin endpoint while one peer is paused. With the fix the master returns `200 OK` for every call within sub-second latencies; without it the master would block, often for minutes, on the dead peer's internal RPC.

## Operational notes

- **Defaults are conservative**. 10 seconds is far below any realistic peer-to-peer round trip on a healthy cluster but above the worst-case TCP retransmit on a busy network. If you run with very low-latency expectations, tighten the deadlines via the JVM flags above.
- **No backwards-compatible setting reverts to the old blocking behaviour**. There is no safe value here — the old behaviour was always a hang.
- The fix changes only the *internal* RPC plane. External clients (Arrow Flight, Postgres wire, S3 wire, HTTP admin, etc.) keep their existing transport-specific timeouts.
