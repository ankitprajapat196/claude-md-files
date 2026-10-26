# Production-Ready API Checklist (Industry Standards)

> Order = the path a request travels: **Design → Request in → Processing → Response out → Operate → Evolve**.
> Sources: OWASP API Security Top 10 (2023), OWASP REST Security Cheat Sheet, RFC 9457, RFC 9745, RFC 8594, IETF Idempotency-Key draft, Google / Microsoft / Zalando API guidelines.

```mermaid
flowchart LR
    A[Design & Contract] --> B[Edge & Transport] --> C[Rate Limit] --> D[AuthN] --> E[AuthZ]
    E --> F[Input Validation] --> G[Business Logic] --> H[Data & External Calls]
    H --> I[Response] --> J[Errors] --> K[Observability] --> L[Test & Deploy] --> M[Versioning & Docs]
```

---

## Phase 1 — Design (before code)

### 1. API Design & Contract
- [ ] **Contract-first** — write the OpenAPI spec before code; spec is the single source of truth.
- [ ] **Resource-oriented URLs** — plural nouns (`/orders/{id}/items`), no verbs, lowercase, consistent casing.
- [ ] **Correct HTTP methods** — GET read, POST create, PUT replace, PATCH partial update, DELETE remove.
- [ ] **Method safety/idempotency respected** — GET has no side effects; PUT/DELETE are idempotent.
- [ ] **Custom actions** for non-CRUD operations, kept consistent (e.g. `POST /orders/{id}:cancel`).
- [ ] **Standard fields** on every resource — `id`, `created_at`, `updated_at` (ISO 8601, UTC).
- [ ] **Consistent naming** — one casing style (snake_case or camelCase) across all fields.
- [ ] **Versioning strategy decided** — e.g. `/v1/`; major versions only.
- [ ] **Lint the spec** — enforce style rules automatically (e.g. Spectral).

---

## Phase 2 — Request Comes In

### 2. Edge & Transport
- [ ] **HTTPS only** — TLS 1.2+, HSTS enabled, no plain HTTP endpoints.
- [ ] **API gateway / reverse proxy** — central place for TLS, routing, rate limits, auth.
- [ ] **CORS** — explicit allow-list of origins; never `*` with credentials.
- [ ] **Request size limits** — max body size, max header size, max upload size.
- [ ] **Timeouts** — request timeout at gateway and server.
- [ ] **Only allowed methods** — reject others with `405 Method Not Allowed`.

### 3. Rate Limiting & Resource Protection (OWASP API4)
- [ ] **Rate limit** per user / API key / IP; return `429 Too Many Requests` + `Retry-After`.
- [ ] **Expose limit headers** (remaining quota, reset time) so clients can back off.
- [ ] **Quotas & cost limits** — on expensive operations (exports, AI calls, SMS/email).
- [ ] **Protect sensitive business flows** (OWASP API6) — bot protection on signup, checkout, OTP.

### 4. Authentication (OWASP API2)
- [ ] **Standard protocol** — OAuth 2.0 / OIDC, JWT, or API keys; never custom crypto.
- [ ] **Validate JWT fully** — signature, `exp`, `iss`, `aud`, algorithm allow-list (reject `none`).
- [ ] **Short-lived access tokens** + refresh tokens with rotation and revocation.
- [ ] **Secrets never in URLs** — tokens in `Authorization` header only.
- [ ] **Brute-force protection** — lockout/throttle on login, OTP and password reset.
- [ ] **401 vs 403** — 401 = not authenticated, 403 = authenticated but not allowed.

### 5. Authorization (OWASP API1, API3, API5)
- [ ] **Object-level checks (BOLA)** — on every request, verify the user owns/can access *this* object ID.
- [ ] **Property-level checks** — user can only read/write fields they're allowed to (no mass assignment).
- [ ] **Function-level checks (BFLA)** — admin endpoints protected by role, not by "hidden" URLs.
- [ ] **Deny by default** — centralised policy (RBAC/ABAC), not scattered `if` checks.
- [ ] **Permission check before existence check** — return 404 if existence itself is sensitive.
- [ ] **Multi-tenant isolation** — tenant ID enforced in every query.

### 6. Input Validation
- [ ] **Never trust input** — validate type, length, range, format, enum for every field.
- [ ] **Schema validation** — use strong types / validators (e.g. Pydantic, JSON Schema).
- [ ] **Allow-list fields** — reject or ignore unknown fields; block read-only fields (`id`, `role`, `balance`).
- [ ] **Content-Type check** — reject unexpected types with `415 Unsupported Media Type`.
- [ ] **Injection prevention** — parameterized queries / ORM; never build SQL or shell commands from input.
- [ ] **File uploads** — check type, size, scan for malware, store outside web root.
- [ ] **Validation errors** → `400` (malformed) or `422` (semantically invalid) with field-level details.

---

## Phase 3 — Processing

### 7. Business Logic & Data Integrity
- [ ] **Idempotency-Key** on POST/PATCH that create side effects (payments, orders, emails); store key + response.
- [ ] **Duplicate detection** — same key + same payload → return stored response; same key + different payload → reject.
- [ ] **Transactions** — multi-step writes are atomic; rollback on failure.
- [ ] **Concurrency control** — optimistic locking with ETag / `If-Match` → `412 Precondition Failed` on conflict.
- [ ] **Async for long tasks** — return `202 Accepted` + job/status URL; don't block requests.
- [ ] **Business rule errors** → `409 Conflict` or `422` with a clear reason.

### 8. Data Layer
- [ ] **Connection pooling** with limits and timeouts.
- [ ] **Indexes** for every filter, sort and lookup used by the API.
- [ ] **No N+1 queries** — eager loading / batching.
- [ ] **Migrations** versioned, reversible, and run in CI/CD.
- [ ] **Sensitive data encrypted** at rest; passwords hashed (bcrypt/argon2).
- [ ] **Soft delete / audit trail** where data history matters.

### 9. External Calls & Dependencies (OWASP API7, API10)
- [ ] **Timeouts** on every outbound call — no unbounded waits.
- [ ] **Retries** only for transient failures, with exponential backoff + jitter + max attempts.
- [ ] **Circuit breaker** for flaky dependencies; fallback / degraded response.
- [ ] **SSRF protection** — allow-list outbound hosts; block internal IPs and metadata endpoints for user-supplied URLs.
- [ ] **Treat third-party responses as untrusted** — validate them like user input.
- [ ] **Background queue** for slow/unreliable work (emails, webhooks) with retries and dead-letter queue.

---

## Phase 4 — Response Goes Out

### 10. Response Design
- [ ] **Correct status codes** — 200 OK, 201 Created (+ `Location`), 202 Accepted, 204 No Content, 4xx client, 5xx server.
- [ ] **Consistent response envelope** — same shape for every endpoint.
- [ ] **Return only needed fields** — no internal IDs, password hashes, or excessive data (OWASP API3).
- [ ] **Pagination on every list** — cursor-based for large/changing data; enforce max page size; return `next_cursor`.
- [ ] **Filtering & sorting** — consistent query params (`?status=active&sort=-created_at`).
- [ ] **Partial success** for bulk operations — per-item results (`succeeded`, `failed` with reasons); `207` or 200 with item statuses.
- [ ] **Caching** — `Cache-Control`, `ETag` / `If-None-Match` → `304`; `no-store` for sensitive data.
- [ ] **Compression** — gzip/br for large responses.
- [ ] **Security headers** — `Strict-Transport-Security`, `X-Content-Type-Options: nosniff`, `Cache-Control`, correct `Content-Type`.

### 11. Error Handling
- [ ] **Standard error format: RFC 9457 Problem Details** — `application/problem+json` with `type`, `title`, `status`, `detail`, `instance`.
- [ ] **Stable machine-readable error code** per error type (clients branch on it, not on message text).
- [ ] **Field-level validation errors** in an `errors` array (field, message).
- [ ] **Correlation / request ID** in every error response (and every response).
- [ ] **No stack traces, SQL or internals** leaked to clients.
- [ ] **Global exception handler** — unexpected errors → `500` with generic message, full detail only in logs.
- [ ] **`503 Service Unavailable` + `Retry-After`** during overload or maintenance.

---

## Phase 5 — Operate

### 12. Logging, Monitoring & Observability
- [ ] **Structured JSON logs** — timestamp, level, request ID, user/tenant ID, endpoint, status, latency.
- [ ] **Never log secrets or PII** — mask tokens, passwords, card numbers.
- [ ] **Audit logs** for security events (login, permission changes, deletes, failed auth).
- [ ] **Distributed tracing** — OpenTelemetry; propagate trace ID across services.
- [ ] **Metrics** — request rate, error rate, latency (p50/p95/p99), saturation (RED/USE).
- [ ] **Alerts** on SLO breaches — error rate, latency, 5xx spikes, auth failure spikes.
- [ ] **Health endpoints** — `/health/live` (process up) and `/health/ready` (dependencies OK).

### 13. Performance & Scalability
- [ ] **Stateless servers** — session/state in DB/Redis, so you can scale horizontally.
- [ ] **Caching layer** (Redis/CDN) for hot reads, with clear invalidation.
- [ ] **Load tested** — know max RPS and breaking point before launch (k6, Locust).
- [ ] **Graceful shutdown** — finish in-flight requests, drain connections.
- [ ] **Autoscaling** configured with sensible min/max.

### 14. Testing
- [ ] **Unit tests** — business logic and validators.
- [ ] **Integration tests** — real DB, real auth flow.
- [ ] **Contract tests** — implementation matches the OpenAPI spec.
- [ ] **Negative tests** — invalid input, missing auth, wrong user's object ID (BOLA), oversized payloads.
- [ ] **Security scanning** — SAST, dependency scan (SCA), DAST / API fuzzing.
- [ ] **Load & chaos tests** — dependency down, slow DB, timeouts.

### 15. Configuration, Deployment & Security Hygiene (OWASP API8)
- [ ] **Secrets in a vault / env** — never in code or repo; rotate regularly.
- [ ] **Environment config** separate per env (dev/staging/prod); debug mode off in prod.
- [ ] **CI/CD pipeline** — lint, test, scan, build, migrate, deploy automatically.
- [ ] **Safe rollouts** — blue-green or canary, with fast rollback.
- [ ] **Dependencies pinned** and patched; minimal container images.
- [ ] **Backups** tested with real restores; disaster recovery plan.

---

## Phase 6 — Evolve

### 16. Versioning & Lifecycle (OWASP API9)
- [ ] **Backward compatibility** — additive changes only within a version (new optional fields OK; removing/renaming = breaking).
- [ ] **New major version** for breaking changes; run old and new in parallel.
- [ ] **Deprecation signalling** — `Deprecation` header (RFC 9745) + `Sunset` header (RFC 8594) + migration guide link.
- [ ] **API inventory** — every endpoint, version, environment and owner tracked; retire old/shadow APIs.
- [ ] **Changelog** published for every release.

### 17. Documentation & Developer Experience
- [ ] **Interactive docs** generated from OpenAPI (Swagger UI / Redoc).
- [ ] **Examples for success AND error responses** on every endpoint.
- [ ] **Auth guide, rate limits, pagination, error codes and idempotency** documented.
- [ ] **Sandbox / test environment** with test credentials.
- [ ] **SDKs or Postman collection** for consumers.

---

## OWASP API Security Top 10 (2023) — Quick Map
| Risk | Covered in |
|---|---|
| API1 Broken Object Level Authorization | §5 |
| API2 Broken Authentication | §4 |
| API3 Broken Object Property Level Authorization | §5, §6, §10 |
| API4 Unrestricted Resource Consumption | §2, §3 |
| API5 Broken Function Level Authorization | §5 |
| API6 Unrestricted Access to Sensitive Business Flows | §3 |
| API7 Server-Side Request Forgery | §9 |
| API8 Security Misconfiguration | §2, §15 |
| API9 Improper Inventory Management | §16 |
| API10 Unsafe Consumption of APIs | §9 |

---

## Quick Summary
Design contract-first with OpenAPI and resource-oriented URLs. Every request passes through HTTPS, a gateway with size limits and timeouts, rate limiting (`429` + `Retry-After`), authentication (validated tokens), and authorization on every object, property and function (BOLA/BFLA), then strict schema validation. Processing uses idempotency keys for side-effecting writes, transactions, optimistic locking, and async jobs for long work; external calls get timeouts, backoff retries, circuit breakers and SSRF protection. Responses use correct status codes, a consistent shape, minimal fields, cursor pagination, partial-success reporting for bulk work, caching headers and security headers; errors follow RFC 9457 with stable error codes and a request ID, never leaking internals. In production: structured logs without PII, tracing, metrics, alerts, health checks, stateless scaling, full testing (including security and load), secrets in a vault, CI/CD with safe rollouts. Evolve with backward-compatible changes, versioned breaking changes, `Deprecation`/`Sunset` headers, an API inventory, and docs with success and error examples.
