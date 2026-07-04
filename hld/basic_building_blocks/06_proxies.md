# 06 — Proxies: Forward, Reverse, Transparent

> A proxy is a server that stands between a client and another server. "Which side" it protects (client or server) defines its type. LB / API Gateway / CDN are all specialized reverse proxies.

## The Three Kinds

| Type | Sits in front of | Purpose |
|------|------------------|---------|
| **Forward proxy** | Clients | Client-side outbound: filter, cache, hide identity |
| **Reverse proxy** | Servers | Server-side inbound: route, TLS, cache, defend |
| **Transparent proxy** | Either | Client doesn't know it exists (network-level intercept) |

## Forward Proxy

Client sends requests *to the proxy*, which forwards them upstream.

```
[Client]  →  [Forward Proxy]  →  [Internet / Origin]
```

**Uses:**
- **Corporate egress control** — proxy blocks Facebook, allows business apps.
- **Caching** — Squid caching common downloads for LAN users.
- **Anonymity** — Tor is a forward proxy chain.
- **Content filtering** — schools, libraries block sites.
- **Bypass regional restrictions** — VPN and geo-unblocker services.
- **Egress control in cloud** — outbound HTTP from a private VPC must go through a proxy (log + policy).

**Client configuration** required (browser proxy settings, HTTP_PROXY env var, PAC file).

## Reverse Proxy

Server-side. Client thinks it's talking to the origin; proxy routes to real backends.

```
[Client]  →  [Reverse Proxy]  →  [Backend 1]
                              →  [Backend 2]
                              →  [Backend 3]
```

**Uses (essentially every prod system):**
- **Load balancing** — see [05](05_load_balancers.md).
- **TLS termination** — one cert, plain HTTP inside.
- **Caching** — full-page or fragment cache.
- **Compression** — gzip/brotli responses.
- **Static file serving** — NGINX serves `/static/*` without touching the app.
- **Path-based routing** — `/api/*` → API tier, `/*` → SPA.
- **Rate limiting & WAF** — block abuse before it reaches your app.
- **A/B testing / canary** — route 5% to new version.
- **Auth offload** — validate JWT at edge; backend trusts.

Client does no configuration; the DNS record points at the proxy.

## Transparent Proxy

Intercepts traffic at network layer (e.g., iptables REDIRECT). Client doesn't know.

**Uses:**
- **ISP-level content injection** (frowned upon).
- **Enterprise TLS interception** (mitm-proxy with an installed CA cert).
- **Kubernetes service mesh** (Envoy sidecar catches all pod traffic via iptables).
- **Airport / hotel captive portals** — redirect all HTTP to login page.

**Ethical concern**: often used without user consent; TLS interception breaks the security model unless done with corporate device management.

## Proxy vs LB vs Gateway — Are They the Same?

Kind of. Overlapping terms:

| Term | Nuance |
|------|--------|
| **Load Balancer** | Reverse proxy focused on distributing traffic evenly |
| **Reverse Proxy** | General term; may or may not balance |
| **API Gateway** | Reverse proxy specialized for APIs (auth, throttle, transform, aggregate) — see [07](07_api_gateway_and_bff.md) |
| **CDN** | Distributed reverse proxy at edge PoPs (cache-first) — see [08](08_cdn.md) |
| **Service Mesh** | Sidecar reverse proxy per service instance (Envoy in Istio/Linkerd) |

Under the hood, they're all built on similar tech (NGINX, Envoy, HAProxy).

## Reverse Proxy Config Example — NGINX

```nginx
upstream app {
  least_conn;
  server 10.0.1.10:8080;
  server 10.0.1.11:8080;
  server 10.0.1.12:8080 backup;
}

server {
  listen 443 ssl http2;
  server_name api.example.com;
  ssl_certificate     /etc/ssl/live/example.com/fullchain.pem;
  ssl_certificate_key /etc/ssl/live/example.com/privkey.pem;

  # static assets served directly
  location /static/ {
    root /var/www;
    expires 1y;
    add_header Cache-Control "public, immutable";
  }

  # rate limit
  limit_req zone=api burst=20;

  location / {
    proxy_pass http://app;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_read_timeout 30s;
  }
}
```

Every proxy header set here matters — omit `X-Forwarded-*` and backends can't log real client IPs.

## Reverse Proxy Features Worth Knowing

- **Buffering**: proxy reads full request before forwarding — protects backend from slow clients (Slowloris attack).
- **Connection pooling** to backends — reuse TCP.
- **Retries** on 5xx with backoff.
- **Circuit breaker** — trip after N failures.
- **URL rewriting** — `/old` → `/new`.
- **Header manipulation** — strip / add / rewrite.
- **Response transformation** — compression, image resizing.

## Service Mesh (Envoy Sidecar)

Each pod has a sidecar reverse proxy. All inbound and outbound app traffic flows through it.

**Gives you:**
- mTLS between services (identity via SPIFFE certs).
- Traffic policy (canary, retry, timeout, circuit breaker) without code changes.
- Observability (distributed tracing, metrics per hop).
- Zero-trust networking (deny by default).

**Cost:** extra hop, extra memory per pod, operational complexity. Overkill for < 10 services.

## Interview Q&A

**Q: Forward vs reverse proxy — the one-line answer?**
- Forward = client's proxy (hides client from server).
- Reverse = server's proxy (hides server from client).

**Q: Why use a reverse proxy when the app could do everything?**
- Concentrate cross-cutting concerns (TLS, rate limit, logging) in one tuned tier.
- Independent scaling (front more proxies vs more app servers).
- Battle-tested C code (NGINX, HAProxy, Envoy) for hot paths; app in higher-level languages.

**Q: How do you get real client IP behind a proxy?**
- `X-Forwarded-For` header, or `Forwarded` (RFC 7239).
- Warning: client can forge; only trust when set by *your* trusted proxy. Strip client-supplied values at the edge.

**Q: What is the PROXY protocol?**
- L4 LBs (TCP mode) can't add HTTP headers. Instead, they prepend a small binary header with the original client IP/port. Backend parses this before HTTP.
- Enable in both LB and backend (NGINX `proxy_protocol on`).

**Q: How does a service mesh differ from an API gateway?**
- Gateway = north-south (outside → your services).
- Mesh = east-west (service → service inside).
- Some products (Envoy, Istio) do both.

**Q: When would you use a forward proxy in production?**
- Egress from private VPC to public APIs: audit trail, IP allowlisting, request filtering.
- CI/CD workers need to hit npm/PyPI with request logging.
- Enterprise policy for developer laptops.

**Q: What's the difference between a proxy and a NAT?**
- NAT operates at L3 (rewrites IP/port).
- Proxy operates at L7 (parses HTTP, can modify content).
- Both hide internal addresses; proxy is way smarter and slower.

**Q: TLS terminate at LB or backend?**
- Terminate at LB when internal network is trusted and you want to inspect/cache/route on HTTP.
- Terminate at backend (LB passthrough) for E2E encryption or when compliance requires (HIPAA, PCI in some cases).
- Compromise: re-encrypt at LB (decrypt-inspect-re-encrypt).

## Real-World Deployments

- **Cloudflare**: edge reverse proxy (WAF, DDoS, cache) → your origin.
- **NGINX**: reverse proxy at the top of virtually every LEMP stack.
- **Envoy**: sidecar in Istio meshes; also Lyft's edge proxy.
- **HAProxy**: still a favorite for TCP-heavy workloads (databases, gRPC).
- **AWS ALB**: managed reverse proxy with L7 routing.
- **Squid**: classic forward proxy for corporate egress.

## Gotchas
- **Request smuggling**: mismatched `Content-Length` / `Transfer-Encoding` handling between proxy and backend → attacker can inject requests. Keep both at same version; strict parsing.
- **Buffer size**: proxy buffers request body; huge uploads → memory pressure. Use streaming mode for large files.
- **Read-through vs on-hit**: proxy hits origin on cache miss; thundering herd. Enable single-flight.
- **Trust of X-Forwarded-*** headers**: without stripping, attacker sets fake client IP. Configure proxy to overwrite, not append.
- **Compression + encryption (BREACH attack)**: gzipping secret-containing responses over HTTPS leaks bits; mitigate with random padding.
