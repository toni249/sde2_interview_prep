# 03 — API Styles: REST, GraphQL, gRPC, RPC

> "Which API style?" — a top HLD question. Answer by matching the *access pattern* and *coupling model* to the style's strengths.

## The Landscape

| Style | Transport | Encoding | Shape | Best For |
|-------|-----------|----------|-------|----------|
| **REST** | HTTP/1.1, 2 | JSON | Resource-oriented | Public APIs, web frontends |
| **GraphQL** | HTTP (POST) | JSON | Query language | Aggregating heterogeneous data, mobile |
| **gRPC** | HTTP/2 | Protobuf (binary) | RPC methods | Service-to-service, low-latency |
| **Classic RPC** (JSON-RPC, XML-RPC) | HTTP | JSON/XML | RPC | Legacy |
| **WebSocket** | TCP | Any | Bidirectional stream | Real-time; see [04](04_realtime_protocols.md) |

## REST — Representational State Transfer

Resources identified by URLs; HTTP methods = verbs.

```
GET    /users            → list
POST   /users            → create        → 201 Location: /users/42
GET    /users/42         → read
PUT    /users/42         → replace       (or PATCH for partial)
DELETE /users/42         → delete
GET    /users/42/orders  → nested resource
```

**Principles (from Fielding's dissertation):**
- Stateless (no server-side session; auth via token in every request).
- Client-server separation.
- Cacheable responses (via HTTP caching).
- Uniform interface (resources + standard methods).
- Layered (client can't tell if talking to LB or origin).
- HATEOAS (responses include links to next actions — rarely followed strictly).

**Pros:** ubiquitous, cacheable (browsers, CDNs know how), tooling everywhere (Postman, curl), easy to debug.
**Cons:** over-fetching (get more than you need), under-fetching (need multiple round-trips), no built-in schema (OpenAPI/Swagger is add-on).

## GraphQL — Query Language for APIs

Single endpoint (`POST /graphql`). Client specifies exactly what fields it wants.

```graphql
query {
  user(id: 42) {
    name
    orders(last: 5) {
      id
      total
      items { name price }
    }
  }
}
```

**Solves:**
- Over-fetching: mobile client gets only needed fields.
- Under-fetching: nested resources in one request.
- Schema-first: types are contract; introspection = tooling for free.

**Adds:**
- **Resolvers**: server-side function per field. N+1 problem is real; solve with DataLoader (batches + caches within request).
- **Subscriptions**: WebSocket-based push for real-time updates.
- **Mutations**: for writes (same query language shape).

**Pros:** flexible, single endpoint, great for BFF layer, strongly typed.
**Cons:** HTTP caching harder (all POSTs to same URL); performance analysis harder; auth per-field is complex; not naturally suited for RPC-style commands.

## gRPC — HTTP/2 + Protobuf

Define service in `.proto`; generate typed clients/servers in many languages.

```proto
service UserService {
  rpc GetUser (GetUserRequest) returns (User);
  rpc StreamUpdates (StreamRequest) returns (stream Update);
}

message User { string id = 1; string name = 2; }
```

**Four call types:**
- **Unary**: 1 req → 1 resp.
- **Server streaming**: 1 req → stream of resp.
- **Client streaming**: stream of req → 1 resp.
- **Bidirectional streaming**: two independent streams.

**Pros:** small payloads (Protobuf is 3-5× smaller than JSON), fast serialization, strong typing across languages, HTTP/2 multiplexing, built-in streaming.
**Cons:** binary format not human-readable; browser support requires gRPC-Web proxy; harder to debug without tooling; less discoverable than REST.

**Sweet spot**: internal microservices where you control both ends.

## Comparison Cheat-Sheet

| Concern | REST | GraphQL | gRPC |
|---------|------|---------|------|
| Payload size | Medium (JSON) | Medium (JSON, but leaner queries) | Small (Protobuf) |
| Latency | Medium | Medium | Low |
| Schema enforcement | Optional (OpenAPI) | Built-in | Built-in |
| Streaming | No native (SSE add-on) | Subscriptions | Yes (native) |
| Browser support | Native | Native | Needs gRPC-Web |
| Caching (HTTP) | Excellent | Poor (POST) | Poor |
| Discoverability | Docs + OpenAPI | Introspection | Reflection |
| Backward compat | URL versioning | Field deprecation | Field numbers frozen |
| Tooling | Best | Very good | Very good (internal) |

## When to Choose Which

**REST**:
- Public API where third parties integrate.
- Resource CRUD dominates.
- CDN caching desired.

**GraphQL**:
- Mobile clients over spotty networks (fewer round-trips).
- Aggregating data from many services (BFF).
- Diverse client types with different data needs.

**gRPC**:
- Service-to-service inside your DC.
- Latency-sensitive (trading, real-time systems).
- Streaming (chat, telemetry).
- Polyglot codebase — Protobuf-generated clients across languages.

## Real-World Combinations

Most large systems use *multiple* styles:
- **Public REST** for third-party developer API.
- **GraphQL** at the mobile BFF layer.
- **gRPC** for internal service mesh.
- **WebSocket** for real-time push.

Example — **GitHub**: REST v3 + GraphQL v4 API, both public.
**Netflix**: Falcor (Netflix's own graph API) → GraphQL migration, internal gRPC.
**Google**: gRPC everywhere internally, JSON APIs for public.

## Versioning Strategies

**REST**:
- URL: `/v1/users`, `/v2/users` — simple, discoverable.
- Header: `Accept: application/vnd.company.v2+json` — cleaner URLs, harder to test.
- Query param: `?version=2` — flexible but easy to forget.

**GraphQL**:
- Never break existing fields; add new ones with new names.
- Mark old fields `@deprecated(reason: "use newField")`.

**gRPC / Protobuf**:
- Never reuse field numbers. Never remove — mark reserved. Add new fields with new numbers.

## Interview Q&A

**Q: REST vs GraphQL — how do you decide?**
- Client shape uniform and caching matters → REST.
- Diverse clients with varying field needs → GraphQL.
- If in doubt, REST is safer; GraphQL adds complexity that must pay off.

**Q: What's the N+1 problem in GraphQL?**
- Query asks for 100 users each with their 5 posts → naive resolver hits DB 1 + 100 times.
- Fix: DataLoader batches (`getUsersByIds([1..100])`) + per-request cache.

**Q: Why is gRPC not used in browsers?**
- Browsers can't send arbitrary HTTP/2 frames from JS. Solution: **gRPC-Web** — subset that transpiles to HTTP/1.1 requests; a proxy (Envoy) converts to real gRPC.

**Q: How does Protobuf beat JSON on size?**
- Field names replaced by numbers (2 vs "customerAddress").
- Varint encoding — small integers use 1 byte.
- No whitespace, no field-value delimiters — length-prefixed.
- Typical 3-5× smaller for structured data.

**Q: Can REST do streaming?**
- Not natively. Options: SSE, chunked transfer encoding, long polling, or upgrade to WebSocket.

**Q: What's an idempotency key and where does it fit?**
- Header (`Idempotency-Key: uuid`) client attaches to POST/PATCH to safely retry.
- Server dedupes by key for N minutes. Common in payments (Stripe).
- Works in REST and GraphQL; gRPC via metadata.

**Q: gRPC deadlines vs timeouts?**
- Deadlines are absolute times ("done by 12:00:00.500"), propagated across services. Every hop inherits and can shorten.
- Timeouts are per-hop durations — easy to miscombine.

**Q: How do you handle partial failures in GraphQL?**
- Return partial data with a per-field `errors` array — client decides what to render.
- REST would return a single error status, losing partial success.

## Gotchas
- **REST HATEOAS**: theoretically clean, practically ignored. Don't bring it up in interviews unless asked.
- **GraphQL over-fetching (server side)**: query resolvers may still fetch unneeded fields from DB. Push the field selection down to SQL.
- **gRPC error semantics**: 16 canonical `Status` codes, not HTTP codes. Map them consistently.
- **Content negotiation** (`Accept`, `Accept-Encoding`) breaks cache if `Vary` is wrong.
- **Public GraphQL exposes complexity**: bad actors can craft expensive queries (deep nesting, aliases). Enforce depth limit + query cost budget.
