# TestAI – Connect to GitHub (Production Plan v2)

This is a production-grade, phased plan for integrating GitHub into TestAI. It supersedes the initial draft with stronger security, scalability, and enterprise readiness. Use it phase-by-phase (you can group phases 1–2 and 3–4 when convenient).

---

## What changed vs. v1 (summary)
- **GitHub App-first** design (fine-grained, short-lived tokens) and optional **GitHub Enterprise Server (GHES)** support.
- **User-centric auth** (magic link by default) with an option to add **OIDC (Google/Microsoft)** for enterprise.
- **Vault Transit** encryption for all sensitive secrets; **no plaintext tokens** in Postgres.
- **RFC 7807** error model (Problem+JSON), **idempotency keys** for write APIs, **OpenTelemetry** tracing end-to-end.
- **ETag + Redis** (optional) for cluster-wide caching; **Resilience4j** with circuit breakers, jittered retries, and concurrency limits.
- **Owner-scoped Mongo** access with a hard guard to block cross-tenant reads/writes.
- **Job queue + webhook dedupe** for PR/merge events; **outbox pattern** for reliable eventing.
- **Security hardening**: strict CORS/CSRF, SSRF prevention, content-size guards, dependency/container/SBOM scanning, signed images (cosign), and runbooks.

---

## Tech context
- **Backend:** Java 21, Spring Boot 3.4.2, Maven
- **Frontend:** Flutter
- **Datastores:** PostgreSQL (users, tokens, settings, caches), MongoDB (Points, NotebookNodes)
- **Infra:** DigitalOcean (Kubernetes + autoscaler, LB, Managed Postgres/Mongo, Vault, DDoS), Ubuntu 24.04
- **Observability:** OpenTelemetry (traces/metrics/logs), Prometheus/Grafana

---

## Phase 0 — Prerequisites & Foundation

### 0.1 GitHub App (not OAuth App)
- Permissions (minimum for read): **Repository Contents: Read**, **Metadata: Read**.
- For write phases: allow **Contents: Read & Write** (enable only after user attempts writes).
- Generate and store: **APP_ID**, **APP_SLUG**, **WEBHOOK_SECRET**, **PRIVATE KEY (.pem)**.
- **GHES support**: make the GitHub base URL configurable (e.g., `GITHUB_API_BASE=https://github.mycorp.com/api/v3`).

### 0.2 Vault & secrets
- Enable **Transit** (key: `testai-transit`).
- Store KV: `GITHUB_APP_ID`, `GITHUB_WEBHOOK_SECRET`, and the **private key** as PEM.
- App loads non-sensitive values on boot and calls **Transit** to encrypt/decrypt sensitive values on demand.

### 0.3 Build & supply chain
- Add **Flyway**.
- Add **Resilience4j** (retry/circuit breaker/bulkhead).
- Add **SBOM** (CycloneDX) generation; enable dependency scanning (OWASP DepCheck/Snyk) in CI.
- **Container**: distroless base, read-only FS, non-root user; **sign images** with cosign; push attestation (SLSA provenance).

### 0.4 Networking & security headers
- Enforce TLS only. Strict **CORS** allowlist per env. Enable **HSTS** and **Content-Security-Policy** for any web UI.
- **SSRF protection**: outbound allowlist to `api.github.com` (or GHES host) only for GitHub clients.

---

## Phase 1 — Users & Authentication (PostgreSQL-backed)

### 1.1 Postgres schema (Flyway `V1__users.sql`)
- `app_user(id UUID, email CITEXT UNIQUE, display_name, ... )`
- `auth_magic_link(id UUID, user_id, token_hash, expires_at, used_at)` — **hash only**, strong KDF (argon2id/scrypt), TTL ≤ 15 min.
- `auth_session(id UUID, user_id, jwt_id, expires_at)` — store **jti** for revocation.
- `user_setting(user_id, key, value_json)` — per-user JSON settings.
- `audit_log(id, user_id, action, resource, details_json, created_at)` — immutable audit.
> Enable the **CITEXT** extension in Postgres for case-insensitive email uniqueness.

### 1.2 Auth flows
- Default: **passwordless magic link**.
- Enterprise option: add **OIDC** (Google/Microsoft/Azure AD) later without breaking API contracts.

### 1.3 Spring Security
- API uses **Bearer JWT** with `sub=<userId>`, `email`, `jti`, `iat`, `exp`.
- **CSRF enabled** for browser flows (CookieCsrfTokenRepository); mobile uses header echo.
- **CORS**: explicit origins; **no `*`**.
- **Logout** revokes `jti` in DB.

### 1.4 API
- `POST /auth/magic-link/request { email }`
- `POST /auth/magic-link/consume { token }` → issues JWT; persists `auth_session` with `jti`.
- `POST /auth/logout`
- `GET /auth/me`

### 1.5 Flutter
- Email input → magic link → deep link → `consume` → store JWT in **SecureStorage**.
- Use **HTTPS claimed link** (preferred) over custom URL scheme to avoid interception.

---

## Phase 2 — Per-user Mongo Access (Points & NotebookNodes)

### 2.1 Collections
- Add `ownerId` (UUID string) to **Points** and **NotebookNodes**.
- Indexes: `{ ownerId: 1, createdAt: -1 }`.
- Backfill script for existing docs; quarantine unknown owners.

### 2.2 Hard guard
- Wrap `MongoTemplate` with `OwnerScopedMongo` that **injects `ownerId` filter** and **rejects** operations missing it.
- For updates/deletes by id: verify the stored `ownerId` matches current user.

### 2.3 Contracts & tests
- DTOs must **not accept** `ownerId` from client; server infers from JWT.
- Integration tests confirm cross-user access is **403**.

---

## Phase 3 — GitHub App Integration (PostgreSQL-backed)

### 3.1 Schema (Flyway `V2__github.sql`)
- `github_installation(id, user_id, installation_id, account_login, account_type, permissions_json, created_at, updated_at)`.
- `github_cached_token(installation_id, token_ciphertext, expires_at, updated_at)` — optional cache of **short-lived** tokens (encrypted via Transit).
- `oauth_state(id, user_id, state_hash, expires_at)` — CSRF/nonce for browser flow.

### 3.2 Token flow
- Sign App **JWT** (PEM from Vault) → exchange for **installation token** (≈1h TTL).
- Optionally cache the token in Postgres (ciphertext + expiry) or in-memory Caffeine; prefer **Redis** if you need cluster-wide cache.
- On 401/403/expiration: refresh automatically with exponential backoff.

### 3.3 Endpoints
- `GET /github/app/installation/urls` → returns install URL with `state`.
- `POST /github/app/installation/callback` → validate `state`, upsert `github_installation` for current user.
- `DELETE /github/app/installations/{installationId}` → disconnect (does not uninstall from GitHub).

### 3.4 Webhooks (recommended)
- `POST /github/webhooks` — verify HMAC. **Deduplicate** by delivery id. Use a **job queue** (e.g., Spring `@Async` + persistent queue like Redis Streams/RabbitMQ) to process events.
- Store minimal metadata in `audit_log`.

---

## Phase 4 — Read-Only GitHub Operations (Resilient & Paginated)

### 4.1 HTTP client
- Timeouts: connect/read ≤ **5 s**; set explicit **connect pool** sizes.
- Retries: 3, **exponential backoff + jitter**, respect `Retry-After`.
- Rate limits: respect `X-RateLimit-*`; backpressure per **installation**.
- **Bulkhead/concurrency limits** per route to protect upstream.

### 4.2 Caching & conditional requests
- Cache **ETags** and small payloads via Caffeine; if running multiple pods, mirror ETags in **Redis** for coherence.
- Send `If-None-Match`; on `304`, serve from cache.

### 4.3 REST + GraphQL mix
- GraphQL for stitched listings (repos, branches, trees) to reduce round trips.
- REST for contents/raw downloads.

### 4.4 API (server)
- `GET /github/installations?page=&perPage=` — paginated.
- `GET /github/repos?installationId=&cursor=` — cursor pagination; return `Link` headers.
- `GET /github/trees?owner=&repo=&ref=&path=&cursor=` — paginated dir listing.
- `GET /github/file?owner=&repo=&path=&ref=` — return metadata and content **only if size ≤ 1–2 MB**; otherwise return a **pre-signed raw URL**.

### 4.5 Errors — RFC 7807 (Problem+JSON)
```json
{
  "type": "https://docs.testai.app/errors/rate-limited",
  "title": "Rate limited",
  "status": 429,
  "detail": "GitHub rate limit exceeded. Retry after reset.",
  "instance": "/github/repos",
  "extensions": {"reset":"2025-11-12T10:41:03Z", "requestId":"..."}
}
```

---

## Phase 5 — Safe Writes (Branch + PR, Idempotent)

### 5.1 Policy
- Never write to default branch directly.
- Branch name: `testai/<YYYYMMDD>/<short-id>`.
- Commit message: `feat(testai): <summary>

trace: <requestId>`.
- App commits are auto-signed by GitHub.

### 5.2 Idempotency & concurrency
- All write endpoints accept header `Idempotency-Key` (UUID). Store a short-lived record to dedupe retries.
- Use **optimistic concurrency**: require `baseSha`; send `If-Match`/file `sha` to avoid overwrites; return **409** on conflict with current `sha`.

### 5.3 Endpoints
- `POST /github/write/prepare` → validate permissions, resolve base branch, return `baseSha`, `branch` suggestion, PR title.
- `POST /github/write/commit` → create branch if missing, put/update file guarded by `sha`.
- `POST /github/write/pr` → open PR with labels (e.g., `from:testai`) and template.

### 5.4 Webhook integration
- On `pull_request` and `push`, update local status; dedupe deliveries; process via job queue; log to `audit_log`.

---

## Phase 6 — Flutter Integration

### 6.1 Auth
- Magic link flow with **SecureStorage**; session expiry handling; logout.
- Use **system browser** (Chrome Custom Tabs / SFSafariViewController) for GitHub install flow; no embedded WebViews.

### 6.2 UI/UX
- Installations/repo lists are **lazy & paginated**.
- Tree view loads on expand; show size badges; disable open for large files.
- Explicit messages for SSO required, insufficient permissions, rate limited (with reset time).

---

## Phase 7 — Observability, Security & Ops

### 7.1 Tracing & metrics
- **OpenTelemetry** everywhere: propagate W3C trace context; include GitHub `X-GitHub-Request-Id` as span attribute.
- Export Prometheus metrics: per-endpoint latency, error %, retry counts, breaker states, `rateRemaining`.

### 7.2 Logging & auditing
- Structured JSON logs with `requestId`, `userId`, `installationId`.
- Redact secrets; apply log sampling for high-volume paths.
- `audit_log` for connect/disconnect, reads of large files, commits, PRs, denials.

### 7.3 Security
- Dependency scanning, container scanning, SBOM, image signing (**cosign**).
- **TLS** only; HSTS; CSP for any web pages.
- Backups & DR: daily Postgres + Mongo snapshots; restore drills.
- **Feature flags** / kill-switch for write APIs.

### 7.4 Performance & DB
- Postgres: connection pool (HikariCP) sizing; `preparedStatementCache` on; consider **PgBouncer** if needed.
- Redis (optional) for cross-pod caches (ETags, idempotency records, webhook dedupe).

---

## Phase 8 — Testing & Compliance

### 8.1 Testing
- Unit: services (GitHub client mocked), token provider, OwnerScopedMongo.
- Integration: WireMock for 429/5xx/ETag; Flyway migrations; CSRF/CORS.
- E2E: ephemeral org/repo — install → list → read → write → PR → webhook.
- Chaos/failure drills: force timeouts, secondary limits, webhook redelivery.

### 8.2 Compliance & privacy
- Data minimization: store only what’s required; document retention.
- GDPR: user delete request removes Postgres rows and **soft-deletes** Mongo docs owned by the user.

---

## Rollout plan
1) Phase 1 → deploy → migrate Flutter to user auth.
2) Phase 2 → backfill `ownerId` and enable OwnerScopedMongo guard.
3) Phase 3 → GitHub App connect; verify per-user installations.
4) Phase 4 → read-only endpoints (ETag, pagination, backoff).
5) Phase 5 → safe writes with idempotency + PR flow.
6) Phase 6–8 → Flutter polish, observability/security, testing/compliance.

---

## Appendix — Snippets

### Spring Security (high-level)
```java
@Bean
SecurityFilterChain http(HttpSecurity h) throws Exception {
  h.csrf(c -> c.csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse()))
   .cors(Customizer.withDefaults())
   .authorizeHttpRequests(a -> a.requestMatchers("/auth/**", "/github/webhooks").permitAll()
                               .anyRequest().authenticated())
   .oauth2ResourceServer(o -> o.jwt());
  return h.build();
}
```

### Problem+JSON error response example
```json
{
  "type": "https://docs.testai.app/errors/conflict",
  "title": "Edit conflict",
  "status": 409,
  "detail": "Base SHA is outdated.",
  "instance": "/github/write/commit",
  "extensions": { "currentSha": "...", "requestId": "..." }
}
```

### Vault Transit helper
```java
public String encrypt(String plaintext) { return transit.encrypt("testai-transit", plaintext); }
public String decrypt(String ciphertext) { return transit.decrypt("testai-transit", ciphertext); }
```

### Owner-scoped Mongo guard (concept)
```java
public <T> List<T> find(Query q, Class<T> type, UUID userId) {
  var guarded = q.addCriteria(Criteria.where("ownerId").is(userId.toString()));
  return mongo.find(guarded, type);
}
```
