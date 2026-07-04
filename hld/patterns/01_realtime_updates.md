# Pattern 1 — Pushing Real-Time Updates

> **Problem:** The server has new data (a chat message, a stock tick, a driver moved). The user needs to see it *now*, without hitting refresh. HTTP is client-pull by default — how do we invert it?

## Core Options

| Approach | Direction | Best for | Overhead |
|----------|-----------|----------|----------|
| **Short polling** | Client → Server, repeat | Rare updates, simplest infra | Wasted requests |
| **Long polling** | Client holds a request open | Moderate updates, no WebSocket infra | 1 connection per user |
| **Server-Sent Events (SSE)** | Server → Client, one-way stream | Feeds, notifications, dashboards | HTTP/1.1 keep-alive; browser caps at 6 |
| **WebSockets** | Full duplex, persistent | Chat, games, collaborative editors | Stateful servers, LB stickiness |
| **HTTP/2 or gRPC streaming** | Duplex over multiplexed streams | Internal service-to-service | Requires HTTP/2 |
| **MQTT / AMQP** | Pub-sub over TCP | IoT, mobile with poor networks | Broker infra |

## Approach Deep-Dives

### 1. Short Polling
```
loop every 5s:
    GET /messages?since=lastId
```
- Simple, stateless, cache-friendly.
- **Bad when** updates are rare (mostly empty responses) or need sub-second latency.

![Hello there](diagrams/f1.excalidraw.png)

### 2. Long Polling
Client sends `GET /messages/wait`, server holds it until data arrives (or 30s timeout), returns, client reconnects immediately
- Near real-time without WebSocket infra.
- Facebook chat used this pre-WebSocket.
- **Watch out:** load balancer idle-timeout must exceed the hold window.

### 3. Server-Sent Events (SSE)
```
GET /stream          Accept: text/event-stream
--- server pushes ---
event: message
data: {"id":42,"text":"hi"}

event: presence
data: {"user":"bob","status":"online"}
```
- One-way (server → client), auto-reconnect, event IDs for replay.
- Great fit for **stock tickers, news feeds, live scores, deploy logs**.
- No native binary; text/UTF-8 only.

### 4. WebSockets
- Starts as HTTP `Upgrade: websocket`, then binary-framed persistent TCP.
- Bi-directional, low overhead per message (~6 bytes framing).
- **Use for**: chat, multiplayer games, collaborative docs, trading UIs.
- **Complexity**: stateful connections → sticky sessions or a **pub-sub bus** so any WS server can push to any user.

## Reference Architecture — Chat App

```
[Client]  <--WS-->  [WS Gateway (stateful)]
                          |
                     subscribes to Redis Pub/Sub channel "user:{id}"
                          |
[Sender API]  →  Kafka  →  [Fanout Worker]  →  Redis PUBLISH user:{id}
                     also   → Cassandra (message history)
```

**Why this shape?**
- WS gateways are stateful → hard to scale. Decouple message *ingest* (stateless API) from *delivery* (WS gateway).
- Redis Pub/Sub lets any WS gateway deliver to any connected user without gateway-to-gateway routing.
- Kafka persists messages for offline delivery + audit.

## Interview Q&A

**Q: WebSockets vs SSE — how do you choose?**
- One-way from server? SSE. Cheaper, works over vanilla HTTP, auto-reconnects.
- Client also pushes frequently (chat typing indicator, cursor position)? WebSocket.

**Q: How do you scale WebSockets to 1M concurrent users?**
- Each connection holds a file descriptor + ~10 KB RAM. 1M conns ≈ 10 GB across a fleet.
- Use `epoll`/`kqueue` servers (Netty, Node, Go). Tune `ulimit -n`, `net.core.somaxconn`, ephemeral port range.
- Terminate TLS at a Layer-4 LB with connection draining; use consistent hashing on `userId` for stickiness.
- Fan-out via Redis Pub/Sub, Kafka, or a dedicated fan-out service (LinkedIn's *Rush*, Slack's *Flannel*).

**Q: How do you deliver messages when the user is offline?**
- Persist to a store (Cassandra, DynamoDB) *before* attempting delivery.
- On reconnect, client sends `last_seen_id`; server replays gaps.
- Add a per-user **inbox** table sharded by userId.

**Q: How do you guarantee ordering?**
- Assign a monotonic server-side sequence per conversation (not per user).
- Client shows in received order but reorders on `seq` from the server.

**Q: Presence (online/offline)?**
- Heartbeat every N seconds; server keys `presence:{userId}` in Redis with TTL 2N.
- Missing heartbeat → mark offline, publish to that user's contacts.

**Q: What about mobile clients?**
- Long-lived WS drains battery. Use APNs/FCM push instead → client wakes and pulls.
- Hybrid: WS while foreground, push while background.

## Real-World Examples
- **WhatsApp**: XMPP over persistent TCP; Erlang for millions of conns per node.
- **Google Docs**: WebSocket + Operational Transforms for concurrent edits.
- **Slack**: WebSocket + "Flannel" edge cache to reduce fan-out.
- **Robinhood**: SSE for stock ticks.
- **Discord**: WebSocket + Elixir/Erlang, per-guild Erlang processes.

## Gotchas
- **Load balancer idle timeout** (ALB default 60s) will kill idle WS/SSE. Send heartbeat pings.
- **Corporate proxies** sometimes strip `Upgrade` header — fall back to long polling.
- **Reconnect storms** after a gateway crash — add jitter (`sleep = base + rand(0, base)`).
- **Head-of-line blocking** on SSE if one slow event stalls the stream.
