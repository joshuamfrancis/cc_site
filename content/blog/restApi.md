---
title: "RestApi"
date: 2026-04-10T20:34:33+10:00
draft: true
tags: ["api", "rest", "micro-services", "integration"]
categories: ["development", "Programming"]
featured_image: ""
summary: "A short writeup on typical concerns on development and use of RESTful APIs"
---

# RESTful API Development & Integration Concerns

A practical reference covering the key concerns when designing, building, and consuming RESTful APIs in production environments.

---

## 1. Authentication & Authorisation

Authentication verifies *who* the caller is; authorisation determines *what* they can do.

### Common Schemes

| Scheme | When to Use | Notes |
|---|---|---|
| API Keys | Server-to-server, low-sensitivity | Easy to implement; rotate regularly. Pass via header (`X-API-Key`) not query string. |
| OAuth 2.0 + JWT | User-context APIs, third-party access | Industry standard. Use short-lived access tokens + refresh tokens. |
| mTLS | High-trust service mesh / internal APIs | Both client and server present certificates. Strong but operationally complex. |
| HMAC Signatures | Webhook callbacks, tamper-proof requests | Sign the payload + timestamp to prevent replay attacks. |

### Token Lifecycle

```
┌──────────┐   credentials    ┌────────────┐  access_token (short-lived)
│  Client   │ ───────────────►│  Auth / IdP │ ──────────────────────────►  API
│           │                 │  Server     │
│           │  refresh_token  │             │  401 Expired ──► refresh flow
└──────────┘ ◄───────────────└────────────┘
```

**Key practices:**

- Never embed secrets in client-side code or version control.
- Use scopes and claims to enforce least-privilege.
- Validate JWTs on every request (signature, `exp`, `aud`, `iss`).
- Store refresh tokens securely (HTTP-only cookies or secure vaults).

---

## 2. CORS (Cross-Origin Resource Sharing)

Browsers enforce the Same-Origin Policy. CORS headers tell the browser which cross-origin requests to allow.

### Preflight Flow

```
Browser                           API Server
  │                                    │
  │  OPTIONS /api/resource             │
  │  Origin: https://app.example.com   │
  │  Access-Control-Request-Method: PUT│
  │ ──────────────────────────────────►│
  │                                    │
  │  204 No Content                    │
  │  Access-Control-Allow-Origin: *    │
  │  Access-Control-Allow-Methods: PUT │
  │  Access-Control-Max-Age: 86400     │
  │ ◄──────────────────────────────────│
  │                                    │
  │  PUT /api/resource                 │
  │ ──────────────────────────────────►│
  │  200 OK                            │
  │ ◄──────────────────────────────────│
```

**Key headers:**

- `Access-Control-Allow-Origin` — whitelist specific origins; avoid `*` with credentials.
- `Access-Control-Allow-Methods` — only expose the HTTP methods you support.
- `Access-Control-Allow-Headers` — explicitly list custom headers (`Authorization`, `Content-Type`).
- `Access-Control-Max-Age` — cache the preflight to reduce OPTIONS traffic.

**Common pitfall:** Wildcarding (`*`) while also sending `Access-Control-Allow-Credentials: true` is rejected by browsers. You must echo back the specific origin.

---

## 3. Timeouts & Resilience

Timeouts prevent cascading failures and resource exhaustion.

### Timeout Layers

```
┌───────────────────────────────────────────────────────────────┐
│                        Client / Consumer                      │
│  ┌──────────────┐  ┌───────────────┐  ┌────────────────────┐ │
│  │ Connect      │  │ Read / Socket │  │ Total Request      │ │
│  │ Timeout      │  │ Timeout       │  │ Timeout            │ │
│  │ (e.g. 5s)    │  │ (e.g. 30s)    │  │ (e.g. 60s)         │ │
│  └──────────────┘  └───────────────┘  └────────────────────┘ │
└────────────────────────────┬──────────────────────────────────┘
                             │
                             ▼
┌───────────────────────────────────────────────────────────────┐
│                    Load Balancer / Gateway                     │
│              Idle timeout · Request timeout                    │
└────────────────────────────┬──────────────────────────────────┘
                             │
                             ▼
┌───────────────────────────────────────────────────────────────┐
│                       API Server                              │
│              Request timeout · DB query timeout                │
│              Downstream service call timeout                   │
└───────────────────────────────────────────────────────────────┘
```

**Best practices:**

- Set timeouts at every layer; never rely on defaults.
- Use circuit breakers (e.g. Polly, Resilience4j, Hystrix) to fail fast on degraded dependencies.
- Implement retry with exponential back-off and jitter.
- Make operations idempotent so retries are safe (use `Idempotency-Key` headers).

---

## 4. Long-Running Operations (Async Request Pattern)

Synchronous HTTP falls apart when server-side work takes minutes or hours (report generation, video encoding, bulk imports). The standard approach is the **Asynchronous Request-Reply** pattern.

### Pattern: Polling

```
Client                              API Server                   Worker
  │                                      │                          │
  │  POST /api/jobs                      │                          │
  │  { "type": "report", ... }           │                          │
  │ ────────────────────────────────────►│                          │
  │                                      │   enqueue job            │
  │  202 Accepted                        │ ────────────────────────►│
  │  Location: /api/jobs/abc-123         │                          │
  │ ◄────────────────────────────────────│                          │
  │                                      │                          │
  │  GET /api/jobs/abc-123               │                          │
  │ ────────────────────────────────────►│                          │
  │  200 OK  { "status": "processing" }  │                          │
  │ ◄────────────────────────────────────│                          │
  │         ... poll again ...           │          ... working ... │
  │                                      │                          │
  │  GET /api/jobs/abc-123               │                          │
  │ ────────────────────────────────────►│                          │
  │  200 OK                              │                          │
  │  { "status": "complete",             │                          │
  │    "result": "/api/reports/xyz" }    │                          │
  │ ◄────────────────────────────────────│                          │
```

### Pattern: Webhooks / Callbacks

Instead of polling, the client registers a callback URL. The server POSTs the result when ready. Combine with HMAC signatures to verify authenticity.

### Pattern: Server-Sent Events (SSE) / WebSockets

For progress updates or streaming results, keep a persistent connection open. SSE is simpler (unidirectional); WebSockets support bidirectional communication.

**Key considerations:**

- Always return `202 Accepted` (not `200`) with a `Location` header for the status resource.
- Include a `Retry-After` header to guide polling intervals.
- Set a TTL on completed job results so storage doesn't grow unbounded.
- Provide cancellation endpoints (`DELETE /api/jobs/{id}`).

---

## 5. HATEOAS (Hypermedia as the Engine of Application State)

HATEOAS is the REST constraint where responses include links that describe available actions, making the API self-navigable.

### Example Response

```json
{
  "id": 42,
  "status": "pending",
  "total": 120.50,
  "_links": {
    "self":    { "href": "/api/orders/42" },
    "approve": { "href": "/api/orders/42/approve", "method": "POST" },
    "cancel":  { "href": "/api/orders/42/cancel",  "method": "POST" },
    "items":   { "href": "/api/orders/42/items" }
  }
}
```

After approval, the response would no longer include the `approve` or `cancel` links — the available state transitions change dynamically.

**Practical reality:** Full HATEOAS adoption is rare. A pragmatic middle ground:

- Include `self` links on all resources.
- Include pagination links (`next`, `prev`, `first`, `last`).
- Include action links for state-machine resources (orders, workflows).
- Use a standard format like HAL, JSON:API, or Siren.

---

## 6. API Gateway Concerns

An API Gateway sits between consumers and your backend services, centralising cross-cutting concerns.

### Gateway Feature Map

```
                         ┌──────────────────────────────────┐
                         │          API Gateway              │
  Consumers ────────────►│                                  │────────► Backend
                         │  ┌────────────────────────────┐  │         Services
                         │  │  Authentication & AuthZ    │  │
                         │  │  (API key, OAuth, JWT)     │  │
                         │  ├────────────────────────────┤  │
                         │  │  Rate Limiting & Throttling│  │
                         │  │  (per-client, per-endpoint)│  │
                         │  ├────────────────────────────┤  │
                         │  │  Quota & Usage Tracking    │  │
                         │  │  (plans, metering, billing)│  │
                         │  ├────────────────────────────┤  │
                         │  │  Request Validation        │  │
                         │  │  (schema, size, content)   │  │
                         │  ├────────────────────────────┤  │
                         │  │  Transformation & Routing  │  │
                         │  │  (path rewrite, header     │  │
                         │  │   injection, versioning)   │  │
                         │  ├────────────────────────────┤  │
                         │  │  Caching                   │  │
                         │  ├────────────────────────────┤  │
                         │  │  Logging, Metrics, Tracing │  │
                         │  └────────────────────────────┘  │
                         └──────────────────────────────────┘
```

### 6.1 Rate Limiting & Throttling

Protects backend services from abuse and ensures fair usage.

| Strategy | Description |
|---|---|
| Fixed Window | Count requests per clock-aligned window (e.g. 100/min). Simple but bursty at boundaries. |
| Sliding Window | Weighted average of current + previous window. Smoother. |
| Token Bucket | Tokens replenish at a fixed rate; each request costs a token. Allows controlled bursts. |
| Leaky Bucket | Requests queue and drain at a constant rate. Strict smoothing. |

**Response headers to include:**

```
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 847
X-RateLimit-Reset: 1719532800
Retry-After: 30
```

Return `429 Too Many Requests` when limits are exceeded.

### 6.2 Quota & Usage Management

Quotas operate at a higher level than rate limits — they define *how much* a consumer is entitled to use over a billing period.

- Tie quotas to API keys or subscription plans (Free: 1,000/day, Pro: 100,000/day).
- Track usage asynchronously (don't add latency to the hot path).
- Provide a usage dashboard or `/api/usage` endpoint.
- Send alerts at 80% / 90% / 100% thresholds.

### 6.3 Request / Model Validation

Validate early, fail fast — reject malformed requests at the gateway before they hit business logic.

- **Schema validation** — Validate request bodies against JSON Schema or OpenAPI specs.
- **Size limits** — Cap `Content-Length` to prevent oversized payloads.
- **Content-Type enforcement** — Reject unexpected media types.
- **Parameter validation** — Type-check path/query params, enforce enums, ranges.
- **Response validation** — Optionally validate outbound responses in dev/staging to catch contract drift.

---

## 7. Versioning

APIs evolve. Breaking changes need a strategy.

| Approach | Example | Pros | Cons |
|---|---|---|---|
| URI Path | `/v2/users` | Explicit, easy to route | URL pollution, hard to sunset |
| Header | `Accept: application/vnd.api.v2+json` | Clean URLs | Less discoverable |
| Query Param | `/users?version=2` | Simple | Easy to forget, cache key issues |

**Recommendations:**

- Use URI path versioning for public APIs (most common and discoverable).
- Only increment the major version for breaking changes.
- Maintain at least N-1 version in parallel during a deprecation window.
- Communicate deprecation via `Sunset` and `Deprecation` headers.

---

## 8. Error Handling

Consistent, informative error responses reduce debugging time for consumers.

### Standard Error Envelope

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Request body failed validation.",
    "details": [
      {
        "field": "email",
        "issue": "Must be a valid email address."
      }
    ],
    "traceId": "abc-123-def-456",
    "documentation": "https://api.example.com/docs/errors#VALIDATION_ERROR"
  }
}
```

**Use appropriate HTTP status codes:**

- `400` — Bad request / validation failure
- `401` — Unauthenticated
- `403` — Forbidden (authenticated but not authorised)
- `404` — Resource not found
- `409` — Conflict (e.g. duplicate creation)
- `422` — Unprocessable entity (semantically invalid)
- `429` — Rate limited
- `500` — Internal server error (never leak stack traces)
- `503` — Service unavailable (maintenance, overload)

---

## 9. Pagination, Filtering & Sorting

Large collections need controlled access.

### Pagination Styles

```
Offset-based:    GET /api/items?page=3&page_size=25
Cursor-based:    GET /api/items?cursor=eyJpZCI6MTAwfQ&limit=25
Keyset-based:    GET /api/items?after_id=100&limit=25
```

- **Offset** — Simple, but degrades on large datasets (DB has to skip rows).
- **Cursor** — Opaque token; stable under inserts/deletes; preferred for feeds.
- **Keyset** — Uses indexed columns; fast but requires a stable sort key.

Include pagination metadata and HATEOAS links:

```json
{
  "data": [ ... ],
  "pagination": {
    "total": 2340,
    "limit": 25,
    "has_more": true
  },
  "_links": {
    "next": { "href": "/api/items?cursor=eyJpZCI6MTI1fQ&limit=25" },
    "prev": { "href": "/api/items?cursor=eyJpZCI6MTAwfQ&limit=25&dir=prev" }
  }
}
```

---

## 10. Caching

Caching reduces latency and backend load.

```
Client Cache          CDN / Gateway Cache         Server
  │                        │                        │
  │  GET /api/products/42  │                        │
  │ ──────────────────────►│  MISS                  │
  │                        │───────────────────────►│
  │                        │  200 OK                │
  │                        │  ETag: "a1b2c3"        │
  │                        │  Cache-Control:         │
  │                        │    max-age=300          │
  │ ◄──────────────────────│◄───────────────────────│
  │                        │                        │
  │  GET /api/products/42  │                        │
  │  If-None-Match: "a1b2c3"                        │
  │ ──────────────────────►│  HIT (not stale)       │
  │  200 OK (from cache)   │                        │
  │ ◄──────────────────────│                        │
  │                        │                        │
  │  ... after max-age ... │                        │
  │  GET /api/products/42  │                        │
  │  If-None-Match: "a1b2c3"                        │
  │ ──────────────────────►│  STALE ───────────────►│
  │                        │  304 Not Modified       │
  │ ◄──────────────────────│◄───────────────────────│
```

**Key headers:**

- `Cache-Control` — `max-age`, `no-cache`, `no-store`, `private`, `public`
- `ETag` / `If-None-Match` — Content-hash-based conditional requests
- `Last-Modified` / `If-Modified-Since` — Timestamp-based conditional requests
- `Vary` — Indicate which request headers affect the cached response

---

## 11. Security Hardening

Beyond authentication, harden the API surface:

- **TLS everywhere** — No exceptions; enforce HSTS.
- **Input sanitisation** — Prevent injection (SQL, NoSQL, LDAP, header injection).
- **Output encoding** — Escape data appropriately for the response context.
- **Request size limits** — Reject oversized bodies early.
- **Security headers** — `X-Content-Type-Options: nosniff`, `X-Frame-Options`, CSP.
- **Dependency scanning** — Audit and patch third-party libraries.
- **Secrets management** — Use vaults (HashiCorp Vault, AWS Secrets Manager), never env vars in plaintext.
- **Audit logging** — Log all auth events, permission changes, and data access patterns.

---

## 12. Observability

You can't fix what you can't see.

```
┌────────────────────────────────────────────────────────────┐
│                     Observability Stack                     │
│                                                            │
│   Logs              Metrics              Traces            │
│   (what happened)   (how much/fast)      (request flow)   │
│                                                            │
│   Structured JSON   Counters, Gauges,    Distributed       │
│   Correlation IDs   Histograms           tracing spans     │
│   Log levels        p50/p95/p99 latency  (OpenTelemetry)   │
│   ELK / Loki        Prometheus / DD      Jaeger / Zipkin   │
└────────────────────────────────────────────────────────────┘
```

**Essentials:**

- Assign a unique `X-Request-Id` / `traceparent` to every request; propagate it downstream.
- Track the four golden signals: latency, traffic, errors, saturation.
- Alert on error rate spikes and latency percentile degradation, not just averages.

---

## 13. Documentation & Developer Experience

The best API is useless if nobody can figure out how to call it.

- **OpenAPI / Swagger spec** — Single source of truth; generate docs, SDKs, and tests from it.
- **Interactive sandbox** — Let developers try endpoints without writing code.
- **Code samples** — Provide examples in at least 3 languages (curl, Python, JavaScript).
- **Changelog** — Document every change; highlight breaking changes.
- **Status page** — Real-time health and incident history.
- **SDKs** — Auto-generate from the OpenAPI spec where possible.

---

## Quick Reference: HTTP Method Semantics

| Method | Idempotent | Safe | Typical Use |
|---|---|---|---|
| GET | Yes | Yes | Read a resource |
| POST | No | No | Create a resource / trigger action |
| PUT | Yes | No | Full replacement of a resource |
| PATCH | No* | No | Partial update |
| DELETE | Yes | No | Remove a resource |
| HEAD | Yes | Yes | Metadata only (no body) |
| OPTIONS | Yes | Yes | CORS preflight / capability discovery |

*PATCH can be made idempotent with JSON Merge Patch, but is not guaranteed by the spec.

---

## Summary Checklist

```
[ ] Authentication scheme selected and enforced
[ ] CORS configured per-origin (no wildcard with credentials)
[ ] Timeouts set at every layer (client, gateway, server, DB)
[ ] Long-running ops use 202 + polling or webhooks
[ ] HATEOAS links on stateful resources and pagination
[ ] API Gateway handles auth, rate limiting, validation, logging
[ ] Rate limits and quotas documented and surfaced in headers
[ ] Request/response validation against OpenAPI schema
[ ] Versioning strategy in place with deprecation policy
[ ] Consistent error envelope with trace IDs
[ ] Pagination on all collection endpoints
[ ] Caching strategy with proper Cache-Control / ETag headers
[ ] TLS enforced, security headers set, inputs sanitised
[ ] Observability: structured logs, metrics, distributed tracing
[ ] OpenAPI spec maintained and published
```