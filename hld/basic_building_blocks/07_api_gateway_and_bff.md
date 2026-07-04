# 07 — API Gateway & Backend-for-Frontend (BFF)

> An API gateway is a specialized reverse proxy that centralizes cross-cutting concerns for many services — auth, throttle, transform, aggregate. BFF is a per-client-type gateway. Both are patterns for taming microservices.

## Why an API Gateway

In a microservices world with 50 services:
- Every service reimplements auth, rate limiting, logging, TLS, CORS.
- Clients need to know 50 hostnames + credential setups.
- Cross-cutting policy (deprecate v1, add mTLS) becomes 50 code changes.

An **API gateway** centralizes it:

```
Client → [API Gateway]  → users-service
                       → orders-service
                       → catalog-service
                       → payments-service
```

## Gateway Responsibilities

| Concern | What the gateway does |
|---------|-----------------------|
| **Routing** | `/api/v1/users → users-service:8080` |
| **Auth** | Validate JWT / OAuth token; inject `X-User-Id` header |
| **Rate limiting** | Per-user, per-app, per-plan; see [17](17_rate_limiting.md) |
| **TLS termination** | One cert, backends plain HTTP |
| **Request/response transformation** | Rename fields, strip PII, change protocol (REST → gRPC) |
| **Aggregation** | One client call fans out to N backend calls (be careful!) |
| **Caching** | GET responses cached with `Cache-Control` |
| **Compression** | gzip/brotli |
| **Observability** | Distributed tracing, access logs, metrics per route |
| **WAF / bot protection** | Block SQLi, XSS, credential stuffing |
| **API key management** | Provision, revoke, quota |
| **Versioning** | `/v1/users` vs `/v2/users` route to different backends |
| **Circuit breaking** | Trip after N failures to a backend |
| **Retry / timeout policy** | Per-route rules |

## Popular Gateways

| Gateway | Note |
|---------|------|
| **AWS API Gateway** | Managed, integrates Lambda, Cognito |
| **Kong** | Open-source (NGINX + Lua); huge plugin ecosystem |
| **Apigee** (Google) | Enterprise, monetization features |
| **Envoy + custom control plane** | High-perf, cloud-native |
| **NGINX / NGINX Plus** | Config-based; simpler needs |
| **Tyk** | Open-source, Go-based |
| **Zuul / Spring Cloud Gateway** | Java ecosystem |

## Backend-for-Frontend (BFF)

**Pattern**: one gateway per client type. Mobile has different needs than web has different needs than smart TV.

```
Mobile App     → Mobile BFF     → services...
Web App        → Web BFF        → services...
Smart TV       → TV BFF         → services...
```

**Why?**
- **Payload shaping**: mobile wants tiny JSON, TV wants image URLs sized for 4K.
- **Aggregation**: home screen may need 5 backend calls; BFF does them server-side over fast internal network.
- **Chattiness**: reduce # of client round-trips.
- **Team ownership**: mobile team owns mobile BFF; independent deploys.

**Example — Netflix**:
- 1000+ device types (TVs, consoles, phones, browsers).
- Each device had a tailored BFF (Node.js) doing aggregation and shaping.
- Later evolved to GraphQL federation for even more flexibility.

## Aggregation — Powerful but Risky

**The dream:**
```
GET /home  →  gateway calls in parallel:
              user-service
              orders-service (last 3)
              recommendations-service
              banner-service
returns unified JSON
```

Client makes 1 request instead of 4. Great for mobile on flaky networks.

**The nightmare:**
- One slow backend blocks the whole response.
- One failing backend cascades error to `/home`.
- Gateway becomes a mini-app with business logic.

**Mitigations:**
- Per-call timeouts + circuit breakers.
- Partial responses: return what's ready, mark rest as `null` with error.
- Fallback data (cached recommendations if service is down).
- Move complex aggregation to a dedicated service (BFF), not into the main gateway.

## Gateway Anti-Patterns

**The god-gateway**: all business logic ends up in Lua/JS scripts inside the gateway. Loses per-service ownership.
- **Fix**: gateway does policy, backends do logic.

**Single point of failure**: gateway goes down → everything down.
- **Fix**: multi-region, multi-AZ, health-checked; stateless where possible.

**Chatty aggregation**: gateway waits on 20 backend calls per request. p99 = max of all p99s.
- **Fix**: parallel calls with hedging, timeouts, circuit breakers.

**Bloated per-route config**: 1000-line config file no one understands.
- **Fix**: declarative config (OpenAPI-driven), CI validation, per-team ownership.

## Reference — Auth Flow at Gateway

```
1. Client → Gateway: GET /api/v1/orders  Authorization: Bearer <jwt>
2. Gateway validates JWT signature + expiry (cached JWKS keys)
3. Gateway extracts userId from JWT claims
4. Gateway → orders-service: GET /orders  X-User-Id: 42  X-Roles: user
5. Backend trusts headers (network is not exposed externally)
6. Response back through gateway with any transformation
```

**Never** let the backend see the raw JWT unless it also validates it — defense in depth.

## Rate Limiting Placement

Best at the gateway *and* per-service:
- Gateway: coarse global limits (per API key, per IP, per plan).
- Service: fine-grained (per user, per resource, per operation).
- Downstream (DB): connection pool limits are the final backstop.

## API Composition Alternatives

- **GraphQL federation** (Apollo, StepZen): each service exposes a piece of the graph; gateway stitches. Alternative to REST BFF.
- **gRPC-Web at the edge**: gateway translates browser gRPC-Web to internal gRPC.
- **Event-driven BFF**: gateway subscribes to Kafka, keeps a materialized view; client reads that view.

## Interview Q&A

**Q: Should every microservice have an API gateway in front?**
- One shared gateway per client type (mobile, web, third-party).
- Not per service — that's just a reverse proxy.

**Q: Gateway vs service mesh — how do they differ?**
- Gateway = north-south (external → internal). Auth, throttling, aggregation.
- Mesh = east-west (service → service). mTLS, retries, observability.
- Often deployed together.

**Q: How does the gateway handle multi-tenancy?**
- API key per tenant → gateway maps key → tenant ID header.
- Rate limits per tenant (Redis counter keyed on tenant ID).
- Sometimes routing per tenant (dedicated backend for enterprise customers).

**Q: How to version APIs at the gateway?**
- URL versioning (`/v1`, `/v2`) with different route targets.
- Sunset headers on old versions to signal deprecation.
- Traffic shift: 10% of `/v1` calls silently route to `/v2` shadow endpoint, compare responses.

**Q: How do you monitor the gateway?**
- Per-route p50/p99 latency, error rate, throughput.
- Per-backend health.
- Log sampling (1% of successful requests, 100% of errors).
- Distributed tracing (trace ID propagated to backends).

**Q: What if the gateway becomes slow?**
- Profile: is auth slow (cache JWKS), aggregation slow (parallelize), TLS slow (session tickets)?
- Scale horizontally; stateless gateway is easy to fan out.
- Move hot logic (rate-limit lookups) to memory / near-cache.

**Q: How does a BFF differ from a monolithic backend?**
- BFF is client-specific and thin — it composes calls to other services.
- Monolith owns the data model.
- BFF talks to many services; monolith holds them internally.

## Real-World Examples
- **Netflix**: pioneered BFF pattern; every device had a Node.js BFF.
- **Stripe**: single global API gateway (custom) doing auth, versioning, idempotency, rate limit.
- **Shopify**: Kong at the edge; per-service internal service mesh.
- **Uber**: custom gateway (originally Node, now Go) handling billions of daily requests.
- **Slack**: edge gateway (Flannel) providing WS fan-out + REST.

## Gotchas
- **Auth cache staleness**: revoked JWTs still accepted for TTL. Short-lived tokens + refresh tokens.
- **CORS headaches**: gateway must set CORS headers; misconfig blocks browsers.
- **Header injection**: attacker sets `X-User-Id: admin` — gateway MUST strip client-supplied trust headers before forwarding.
- **Blocking I/O in Lua/JS filters**: gateway request loop stalls under load.
- **Rate limiter state**: if gateway has multiple instances, shared Redis is required — else 3× the effective limit.
- **N+1 aggregation**: `/home` fanning to 20 backends for every user; response worse than client-side calls. Measure.
