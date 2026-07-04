# 05 — Load Balancers

> A load balancer sits in front of your servers, spreading traffic across them for scale and fault tolerance. Get L4 vs L7, algorithms, and health checks straight — most LB questions are variations of these.

## L4 vs L7

| | L4 (Transport) | L7 (Application) |
|---|---|---|
| What it inspects | TCP/UDP header only (IP, port) | HTTP headers, URL, body |
| Decisions | Forward this connection | Route by path/host/header |
| Throughput | Very high (per-packet) | Lower (parses HTTP) |
| Features | TCP-level failover | Path routing, WAF, rate limit, TLS termination |
| Examples | AWS NLB, HAProxy TCP, IPVS | AWS ALB, NGINX, Envoy, Traefik |

**Rule:** L7 for HTTP/S (99% of web workloads). L4 for raw TCP (DB proxies, gRPC bidirectional streaming, WebSocket at scale).

## Balancing Algorithms

| Algorithm | How | When |
|-----------|-----|------|
| **Round Robin** | Next server in list | Uniform servers, stateless |
| **Weighted Round Robin** | Weights reflect capacity | Mixed instance sizes |
| **Least Connections** | Fewest active conns | Long-lived requests (streams, WS) |
| **Least Response Time** | Fastest observed | Latency-sensitive |
| **IP Hash / Source Hash** | `hash(clientIP)` maps to server | Sticky sessions without cookies |
| **Consistent Hashing** | Ring — adds/removes minimal remap | Cache backends, sharded services |
| **Random with Two Choices** | Pick 2 random, choose less loaded | Simple, near-optimal |
| **Power of D Choices** | Same idea, D>2 | Better in high load |

**Interview favorite:** *Power of Two Choices* — theoretical result that picking the less loaded of 2 random servers gives near-optimal load with almost no state.

## Health Checks

- **Active**: LB probes each backend (`GET /health` every 5 s). Mark down after N failures, mark up after M successes.
- **Passive**: track real request failure rate; eject on threshold.
- **Deep vs shallow**: shallow (`/health` returns 200) doesn't catch a hung DB. Deep (`/ready` checks dependencies) does — but risks cascading failure if dependency blips.
- **Startup vs runtime probes** (Kubernetes): `startupProbe` runs at boot with long timeout, `livenessProbe` restarts stuck pods, `readinessProbe` gates traffic.

## Sticky Sessions (Session Affinity)

Route all requests from one client to the same backend.

**Techniques:**
- **Cookie-based** (LB inserts `AWSALB=...` cookie).
- **Source IP hash** (fragile behind corporate NAT).
- **Connection stickiness** (WS naturally sticky per connection).

**When you need it:**
- In-memory session store (avoid; externalize to Redis).
- WebSocket / SSE connections.
- Server-side caching that isn't shared.

**Downside:** hot backend if one client dominates; harder to drain for deploys.

## TLS Termination

Two placements:
- **At the LB**: cheaper (one cert), backends speak plain HTTP; requires trust in internal network.
- **End-to-end (passthrough)**: LB just forwards TLS bytes (L4 mode); backend terminates. Needed for E2E encryption, mTLS, HIPAA.

Modern stacks often terminate at the edge CDN, re-encrypt to LB, and use mTLS inside the mesh.

## Global vs Regional LBs

- **GeoDNS / Global LB** (Route 53 latency-based, Cloudflare load balancing): resolves domain to nearest region.
- **Regional LB** (ALB, NLB): distributes across AZs within a region.
- **Zonal LB** (rack-level): distributes within an AZ.

Users hit: DNS → global LB IP → regional LB → pod.

## Anycast

Multiple servers announce the same IP; BGP routes clients to the "closest" one. Cloudflare, Google DNS (8.8.8.8) use this. Enables **any datacenter** to serve *any* user IP.

## Reference — HTTP Request Path (Typical)

```
Client
  │  DNS → global LB IP (Anycast)
  ▼
CDN / Edge (TLS termination, WAF, caching)
  │  cache miss → origin
  ▼
Regional ALB (L7, TLS re-terminate optional, path routing, HTTP/2)
  │
  ▼
Target Group (auto-scaling app tier)
  │
  ▼
Backend service — possibly behind another internal LB / service mesh
```

## Load Balancer Deployment Patterns

**Active-Passive**: one LB serves traffic, other stands by. Cheap, but wastes capacity.
**Active-Active**: multiple LBs share load via ECMP or DNS round-robin. Higher availability.
**LB in the DNS**: multiple A records → client picks; primitive but works (Netflix used this for years).

## Service Discovery + LB

Ephemeral service instances register with a discovery layer (Consul, etcd, Eureka, Kubernetes Endpoints). LB reads from discovery, keeps target list fresh.

## Interview Q&A

**Q: L4 vs L7 for gRPC?**
- gRPC uses long-lived HTTP/2 connections. Naive L4 LB pins a connection to one backend → uneven load.
- L7 LB (Envoy, ALB) can balance individual **streams** across backends.

**Q: How do you drain traffic from a backend for deploy?**
- Mark unhealthy in LB → LB stops sending new connections.
- Wait for existing connections to close (drain timeout).
- Then shut down. Zero-downtime deploy pattern.

**Q: What about long-lived WebSockets during deploy?**
- Send close frame to client with reason "server draining".
- Client reconnects with backoff → LB routes to new backend.
- Set drain timeout > longest reasonable session, but shorter than "forever".

**Q: How do you scale the LB itself?**
- Cloud LBs (ALB, GCLB) scale automatically.
- Self-hosted: multiple LB instances behind Anycast IP + BGP; or DNS round-robin between LB VIPs.
- L4 hardware LBs (F5, Citrix) still used at large enterprises for absolute performance.

**Q: What happens when a backend gets stuck (not crashed, just slow)?**
- Health check `/ready` should timeout aggressively and mark unhealthy.
- Passive: outlier detection ejects backends with elevated latency.
- Circuit breakers in the LB (Envoy) trip after failure rate threshold.

**Q: How does consistent hashing help LB?**
- Cache tier: consistent hash on request key → same key hits same cache node → high hit rate even when nodes come/go.
- Without it, adding one cache node invalidates ~all keys.

**Q: LB adds latency — how much and where?**
- Cloud LB adds ~1-3 ms per hop.
- Extra TLS handshake if re-terminating.
- Trade for reliability, scale, and observability.

**Q: Can an LB be a SPOF?**
- Single LB instance, yes. Solution: LB deployed multi-AZ, health-checked by DNS; or Anycast.

## Popular LBs

| LB | Level | Strengths |
|----|-------|-----------|
| **AWS ALB** | L7 | Managed, path routing, WAF integration |
| **AWS NLB** | L4 | Extreme throughput, static IP, TLS passthrough |
| **NGINX** | L4 + L7 | Ubiquitous, config flexible |
| **HAProxy** | L4 + L7 | Very fast, mature |
| **Envoy** | L7 (HTTP/gRPC) | Modern, service-mesh core (Istio) |
| **Traefik** | L7 | Auto-discovery in Docker/K8s |
| **Cloudflare** | Global edge | Anycast, DDoS mitigation, cache |

## Gotchas
- **Health check that hits the DB** → DB blip → LB marks all backends down → 100% error.
- **Sticky sessions + auto-scale**: scale-down kicks users off. Externalize state.
- **`Connection: close` from LB** breaks keep-alive → connection storm.
- **Idle timeout mismatch** (client 5m, LB 60s) → mysterious dropped connections.
- **Slow-start**: newly added backend gets a fraction of load ramping up, else cold cache melts.
- **PROXY protocol** to preserve client IP when LB terminates TCP; backend must speak it.
