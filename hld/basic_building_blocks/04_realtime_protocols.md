# 04 — Real-Time Protocols: Polling, SSE, WebSocket, WebRTC

> The client wants live updates. Pick the *simplest* protocol that meets latency and directionality needs. Related to [Pattern 1 — Real-Time Updates](../patterns/01_realtime_updates.md).

## Directionality Matters

| Need | Protocol |
|------|----------|
| Server sends occasionally | Short polling |
| Server sends when it has news | Long polling / SSE |
| Server pushes a continuous stream | SSE |
| Client and server both push freely | WebSocket |
| Peer-to-peer media (audio/video) | WebRTC |

## Short Polling

```
Client: every 5s → GET /messages?since=lastId
Server: responds immediately with current state
```

- **Simplest.** Uses regular HTTP infra, cache-friendly, no state.
- **Wastes requests** when there's nothing new.
- OK for low-frequency updates (dashboard refresh every minute).

## Long Polling

```
Client → GET /wait?since=X
Server holds request open until data OR timeout (30s)
Server responds; client immediately reopens
```

- Near-real-time without WebSocket infrastructure.
- Facebook Chat used this before WebSocket was mature.
- **Costs**: one open connection per client; LB idle timeout must exceed hold window; abrupt server restart drops all held requests.

## Server-Sent Events (SSE)

Standard: [WHATWG EventSource](https://html.spec.whatwg.org/multipage/server-sent-events.html). One-way server → client stream over HTTP.

```
Client: new EventSource('/stream')

Server response:
Content-Type: text/event-stream
Cache-Control: no-cache

event: message
data: {"id":42,"text":"hi"}
id: 42

event: presence
data: {"user":"bob","status":"online"}
id: 43
```

**Features:**
- Auto-reconnect on connection drop.
- **Last-Event-ID** replay: server can resume from where client left off.
- Text-only (UTF-8); no binary support.
- Works through HTTP proxies and CDNs (regular HTTP request).
- **Browser limit**: 6 SSE connections per origin (per HTTP/1.1 spec); HTTP/2 lifts this to ~100.

**Great for:** notification feeds, live dashboards, deploy logs, stock tickers, LLM streaming (ChatGPT uses SSE).

**Bad for:** anything the client also needs to send in real-time.

## WebSocket

Full-duplex, persistent TCP connection with a small framing layer.

**Handshake starts as HTTP:**
```
GET /chat HTTP/1.1
Host: example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: <base64>
Sec-WebSocket-Version: 13

HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: <hash>

--- from here: binary frames both ways ---
```

**Framing overhead**: 2-14 bytes per message.

**Features:**
- Bidirectional, low latency, binary or text.
- Any payload format (JSON, Protobuf, MessagePack, raw bytes).
- Long-lived; you have to design heartbeats to detect dead connections.

**Costs:**
- Stateful: server holds one open FD per client → sticky sessions or a pub-sub bus for fan-out.
- Corporate proxies sometimes strip the `Upgrade` header → fall back to long polling.
- LB idle timeout: send ping frames every ~30 s.

**Great for:** chat, multiplayer games, collaborative editing, trading UIs, IDE remote sessions.

## SSE vs WebSocket

| | SSE | WebSocket |
|---|---|---|
| Direction | Server → Client | Bidirectional |
| Protocol | HTTP | Upgrade from HTTP, then binary |
| Reconnect | Built-in, with `Last-Event-ID` | Manual |
| Binary | No | Yes |
| Proxy/CDN friendly | Yes (regular HTTP) | Sometimes strips Upgrade |
| Server complexity | Low (regular HTTP handler) | Higher (stateful, framing) |
| Client complexity | Low (`EventSource`) | Medium |
| Load balancing | Any HTTP LB | Sticky sessions or bus |

**Rule:** if it's one-way, prefer SSE. Reach for WebSocket only when the client also needs to push.

## WebRTC — Peer-to-Peer Media

Not really an alternative to the above — different problem: low-latency peer-to-peer audio/video/data.

**Signaling** (WebSocket typically) coordinates:
- ICE candidates (public IPs discovered via STUN, relayed via TURN if NAT is stubborn).
- Session Description Protocol (SDP) exchange.

Then media flows peer-to-peer over UDP (SRTP for media, SCTP for data channel).

**Use for:** video calls (Zoom, Meet), P2P file sharing, low-latency multiplayer.

## MQTT / AMQP

Broker-based pub-sub over TCP. Popular for IoT and mobile.
- **MQTT** — lightweight (2-byte header minimum), QoS levels (0/1/2), last-will messages. Runs on constrained devices.
- **AMQP** — richer routing (exchanges, bindings). Used by RabbitMQ.

Not commonly in web app frontends; consider for device fleets.

## Server-Side Push (HTTP/2 / HTTP/3)

HTTP/2 server push was deprecated by browsers (bad cache interaction). HTTP/3 has no equivalent. **Don't rely on it.**

## Reference Architecture — Live Chat

```
Client ── WS ──► WS Gateway (stateful, e.g. Node/Go/Elixir)
                       │
                       ▼
                 subscribes to Redis Pub/Sub: chan:{roomId}

Sender ──HTTP──► REST API ──► Kafka
                                 │
                                 ├─► DB write (message history)
                                 └─► Fan-out worker
                                        │
                                        ▼
                                 Redis PUBLISH chan:{roomId}
                                        │
                                        ▼
                                 All WS Gateways subscribed
                                 → push to connected clients
```

**Why this shape?** Decouple *ingest* (stateless, easy to scale) from *delivery* (stateful, needs sticky routing). Redis Pub/Sub lets any WS gateway push to any user without gateway-to-gateway routing.

## Heartbeats & Reconnection

- Send ping every 30 s; if 2 missed, close.
- Client reconnect with **exponential backoff + jitter** (`sleep = base * 2^attempts + rand(0, base)`).
- Deduplicate on reconnect using message IDs.

## Interview Q&A

**Q: How do you scale WebSocket to 1M concurrent connections?**
- Each conn = 1 file descriptor + ~10 KB RAM. 1M ≈ 10 GB.
- epoll/kqueue-based server (Netty, Go net, Node). Tune `ulimit -n`, `somaxconn`, ephemeral port range.
- Shard by userId across a fleet; use consistent hashing at L4 LB.
- Pub-sub bus (Redis, Kafka) for cross-node fan-out.

**Q: How do you handle a WS gateway crash?**
- Client detects disconnect, reconnects (with backoff + jitter to avoid thundering herd).
- On reconnect, client sends last-seen message ID; server replays missing messages from DB.

**Q: Why send messages through Kafka before delivering via WS?**
- **Durability**: WS delivery is best-effort; Kafka gives at-least-once.
- **Offline delivery**: user reconnects hours later, replays from persistent store.
- **Fan-out**: multiple consumers (notifications, search index, analytics) can subscribe.

**Q: SSE vs polling for a stock ticker?**
- SSE. Persistent connection, server pushes each tick — no wasted "no new data" requests. Auto-reconnect. Only one direction needed.

**Q: How does WebSocket auth work?**
- Include token in the Upgrade URL query string or a subprotocol header.
- Some LBs strip Authorization on WS handshake; use `Sec-WebSocket-Protocol` or a cookie.
- Re-validate token periodically; kick user on expiry.

**Q: What's the difference between message-oriented WS and stream-oriented WS?**
- WS is *message-framed*: each `send()` is one message, delivered atomically.
- Underneath TCP is a stream; the WS layer adds framing.
- Contrast with raw TCP where you write your own framing.

**Q: How do you rate-limit WebSocket clients?**
- Per-connection token bucket for outbound messages from client (chat: 10 msgs/sec).
- Per-user across all connections (Redis-backed counter).
- Server can send `close` frame with policy-violation code (1008).

## Gotchas
- **LB idle timeout** (ALB default 60 s) kills idle WS/SSE. Send heartbeats.
- **CDN + WS**: Cloudflare supports WebSocket, but you must enable it explicitly; other CDNs may not.
- **Sticky sessions**: naive LB may send reconnects to different node; combine with pub-sub bus.
- **Backpressure**: client can't drain messages → server buffer grows unbounded. Drop or disconnect.
- **Reconnect storms**: after a gateway crash, all clients reconnect at once. Add jitter.
- **HTTP/1.1 SSE limit**: browser caps 6 concurrent connections per origin; use HTTP/2 or subdomain sharding.
