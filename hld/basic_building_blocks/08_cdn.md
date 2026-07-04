# 08 — CDN (Content Delivery Network)

> A CDN is a globally distributed reverse-proxy + cache network that serves content from the edge — close to users, close to your ideal ~50 ms budget for the first byte.

## The One-Line Value

**Move bytes close to users.** Users in Mumbai fetching from us-east-1 pay 200 ms RTT per round-trip. Edge caching at a Mumbai PoP drops that to 5 ms and offloads origin bandwidth.

## Anatomy

```
[User]  →  [Edge PoP nearest]  →  [Regional cache]  →  [Origin]
             (~100+ PoPs worldwide)   (fewer, mid-tier)    (your servers)
```

- **Edge PoP** (Point of Presence): server farm in a city. Cloudflare has 300+ PoPs.
- **Anycast IP**: all PoPs announce the same IP; BGP routes user to nearest.
- **Tiered cache**: if edge misses, ask regional; if regional misses, hit origin.

## What Belongs on a CDN

**Definitely:**
- Static assets: JS, CSS, images, fonts, videos.
- Downloadable files (installers, PDFs).
- Immutable content (versioned URLs).

**Often:**
- API GET responses with proper `Cache-Control`.
- Rendered HTML from CMS (with short TTL).
- Software updates, package registries.

**Rarely:**
- Personalized content (per-user).
- Real-time data (stock prices, live scores — use WebSocket to origin).
- Write endpoints (POST, PUT).

## Cache Keys

Default key = URL. Adjustments:
- `Vary: Accept-Encoding` — separate gzip vs brotli entries.
- `Vary: Accept-Language` — per-language versions.
- Custom: include cookie value, header, or query param.

**Bad key = 0 hit rate.** Including a session cookie in the key means every user has their own copy → cache useless. Strip cookies before caching.

## Cache-Control Headers

| Directive | Effect |
|-----------|--------|
| `public` | Anyone can cache |
| `private` | Only browser, not CDN |
| `no-store` | Don't cache anywhere |
| `no-cache` | Cache but revalidate before use |
| `max-age=3600` | Fresh for 1 hour |
| `s-maxage=86400` | CDN can hold for 1 day (overrides max-age) |
| `stale-while-revalidate=60` | Serve stale for 60s while refetching |
| `stale-if-error=600` | Serve stale on origin error |
| `immutable` | Content will never change; browser won't revalidate |

## Cache Invalidation

Two options:

**TTL expiry (passive):** simplest, but stale until TTL passes.

**Explicit purge:**
- **Purge by URL**: `POST https://api.cloudflare.com/.../purge {"files": ["https://example.com/logo.png"]}`
- **Purge by tag** (Fastly, Cloudflare Enterprise): tag responses with `Cache-Tag: product-42, category-shoes`; purge whole tag.
- **Purge everything**: nuclear option; causes origin traffic spike.

**Best practice**: **immutable versioned URLs** — never invalidate. Instead of updating `logo.png`, deploy `logo.a1b2c3.png`.

## Cache Miss vs Cache Hit

**Hit ratio** is the KPI. 90%+ = healthy static cache; 60-80% for dynamic-with-caching.

**Miss reasons:**
- First request to that URL.
- TTL expired.
- Purged.
- Cache-key mismatch (typo in `Vary`).
- Object too big (CDN has max object size — 5 GB on CloudFront, 500 MB Fastly by default).

## Origin Shielding

If 10 PoPs miss simultaneously, they'd each hit origin. **Shield tier** (regional cache) absorbs so origin sees at most one request per object.

```
Edge PoP (miss) → Regional Shield PoP (miss) → Origin
Edge PoP (miss) → Regional Shield PoP (hit)
```

## Beyond Caching — What Modern CDNs Do

**DDoS mitigation**: absorb Tbps floods at anycast edge; block volumetric attacks before they reach origin.

**WAF (Web Application Firewall)**: rules for SQLi, XSS, credential stuffing.

**Rate limiting**: per-IP, per-URL, at global edge.

**TLS termination**: certificate served from CDN; managed cert renewal.

**Edge compute**: Cloudflare Workers, Lambda@Edge, Fastly Compute@Edge. Run code (auth, personalization, A/B test) at the PoP without hitting origin.

**Image optimization**: resize, format-negotiate (WebP/AVIF), quality-adapt on the fly.

**Video / HLS delivery**: segment cache; adaptive bitrate.

**Bot management**: fingerprint clients, block scrapers.

## Streaming Media (HLS / DASH)

Video is chunked into segments (2-10 s each), listed in a manifest (`.m3u8` or `.mpd`).

```
1. Client → CDN: GET /video/playlist.m3u8    (small manifest, cached)
2. Client → CDN: GET /video/segment_001.ts   (cached, immutable)
3. Client → CDN: GET /video/segment_002.ts   ...
```

Segments are **immutable and cacheable forever** — huge cache hit rate. Adaptive bitrate: client switches to lower-quality manifest if bandwidth drops.

## Signed URLs (Private Content)

Public CDN URLs are cacheable but public. For private content:
- **Signed URL**: `https://cdn.example.com/vid42?Signature=abc&Expires=...` — CDN validates signature before serving.
- **Signed cookie**: cookie authorizes a set of URLs, avoids per-URL signing.
- Signature validation happens at edge, not origin — still cached, still fast.

## CDN Placement Choices

| Provider | Strengths |
|----------|-----------|
| **Cloudflare** | Huge network (300+ PoPs), free tier, integrated WAF/Workers |
| **CloudFront (AWS)** | Deep AWS integration, Lambda@Edge |
| **Fastly** | Programmable VCL, real-time purge, Compute@Edge |
| **Akamai** | Enterprise, largest global footprint, oldest CDN |
| **Google Cloud CDN** | Deep GCP integration |
| **BunnyCDN** | Cheap, simple, good for indie/mid-sized |

## Reference — E-commerce Site with CDN

```
Static:
  - /static/*.js, css, images  → CDN, cache 1yr, immutable versioned URLs

Product pages (semi-dynamic):
  - GET /product/42  → CDN with s-maxage=300, Vary: Cookie=none
  - On edit: purge by Cache-Tag: product-42

Personalized (per-user):
  - GET /cart  → private, no-store  (bypasses CDN entirely)

APIs:
  - GET /search?q=shoes  → CDN with s-maxage=60, key includes ?q
  - POST /checkout  → CDN passthrough, no caching
```

## Interview Q&A

**Q: How does a CDN work end-to-end for a first-time visitor in Mumbai fetching a US-hosted image?**
1. DNS resolves `cdn.example.com` to nearest edge (Mumbai PoP) via Anycast.
2. Client TCP + TLS handshake with edge (5 ms).
3. Edge checks cache — miss.
4. Edge → regional shield (Singapore, ~30 ms).
5. Shield miss → origin (US, ~250 ms). Now cached at both tiers.
6. Response back to client (5 ms).
- **Second visitor in Mumbai**: hit at edge, ~5 ms total.

**Q: Why versioned URLs (`logo.a1b2.png`) over invalidation?**
- Immutable = infinite cache TTL = perfect hit rate.
- No purge race conditions (some PoPs still serve old version).
- Rollback is trivial (change the reference in HTML).

**Q: How do you handle a spike (celebrity tweets your product link)?**
- CDN absorbs the read spike at edge; origin sees maybe 10 QPS instead of 1M.
- Warm the cache preemptively if you know it's coming.
- Set `stale-while-revalidate` so bursts don't hammer origin.

**Q: When is a CDN useless?**
- Per-user content with no shared cache benefit.
- Real-time data changing every second (use WebSocket / SSE from origin edge).
- Very small user base concentrated in one region.

**Q: What's cache poisoning?**
- Attacker sends a request with headers that make it different from normal traffic (e.g., `X-Forwarded-Host: evil.com`), CDN caches the malicious response under normal cache key.
- Defense: normalize keys, strip untrusted headers, keep cache key strict.

**Q: CDN vs edge compute — how do they combine?**
- Static content: served from cache, no compute.
- Personalization (recommendations, geo-adapted content): run at edge, still avoid origin latency.
- Origin: only for uncacheable dynamic data.

**Q: How to test cache behavior?**
- Add response header logging (`X-Cache: HIT/MISS`).
- Use `curl -I` and inspect `age`, `x-cache`, `cf-cache-status`.
- Monitor hit ratio in CDN dashboard.

**Q: CDN storage cost vs origin storage?**
- CDN caches, doesn't durably store — origin is source of truth.
- Cost is bandwidth (egress) + edge storage tier for popular objects.

## Gotchas
- **Cookies breaking cache**: any `Cookie` header can turn every request into unique key. Strip at CDN edge.
- **Wrong `Vary`**: `Vary: User-Agent` = one cache entry per browser version → 0.1% hit rate.
- **`Set-Cookie` on cacheable response**: browser caches the response with the cookie → cross-user leak. Always `Cache-Control: private` when setting cookies.
- **Purge lag**: even after purge, edge PoPs may serve stale for seconds; not consistent globally.
- **DNS TTL for CDN**: too high and you can't fail over; too low and DNS traffic spikes.
- **Chunked uploads**: CDN may not proxy `Transfer-Encoding: chunked` correctly to origin; use multipart or presigned S3 (see [Pattern 6](../patterns/06_large_blobs.md)).
- **PoPs in Iran, China, etc.**: geopolitical constraints; Cloudflare doesn't serve some regions. Plan for it.
