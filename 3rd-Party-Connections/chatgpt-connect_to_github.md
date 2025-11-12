# TestAI – Connect to GitHub (Implementation Plan)

This document provides a detailed, production-ready plan for implementing GitHub repository integration in the TestAI application. The plan reflects best practices and phased improvements from the initial draft, ensuring secure, scalable, and maintainable architecture.

---

## Overview

**Goal:** Enable TestAI users to connect to their GitHub repositories through a secure, user-based integration similar to ChatGPT, Claude, or Perplexity.

**Technologies:**
- **Backend:** Java 21, Spring Boot 3.4.2, Maven
- **Frontend:** Flutter
- **Databases:** PostgreSQL (user data, tokens, settings), MongoDB (Points & NotebookNodes)
- **Infra:** DigitalOcean (Kubernetes, Vault, Managed DBs)

---

## Phase 0 – Prerequisites & Foundation

### GitHub App Setup
- Create a **GitHub App** (not OAuth App) with:
  - Permissions: Repository Contents (Read), Metadata (Read)
  - Optional: Webhooks for PR, Push, Installation events
- Save the App ID, Slug, Webhook Secret, and Private Key.

### Vault Configuration
- Use **Vault Transit** for encrypting secrets.
- Store:
  - `GITHUB_APP_PRIVATE_KEY`
  - `GITHUB_APP_ID`
  - `GITHUB_WEBHOOK_SECRET`
- Use Transit for runtime encryption/decryption of tokens stored in PostgreSQL.

### Application Setup
- Add Flyway (PostgreSQL migrations).
- Add Resilience4j (retries, circuit breakers).
- Add Jackson Afterburner for JSON performance.
- Inject secrets at runtime (never bake into Docker images).

---

## Phase 1 – User & Auth Model

### PostgreSQL Schema
Create Flyway migration `V1__users.sql` for tables:
- `app_user`: user profiles
- `auth_magic_link`: passwordless login tokens
- `auth_session`: JWT session tracking
- `user_setting`: per-user preferences
- `audit_log`: access auditing

### Authentication Flow
- Magic link–based, passwordless login.
- Use **Bearer JWT** for API authentication.
- Enable **CSRF** for state-changing endpoints.
- CORS: strict allowlist per environment.

**Endpoints:**
- `POST /auth/magic-link/request` – send magic link
- `POST /auth/magic-link/consume` – verify and issue JWT
- `POST /auth/logout` – revoke session
- `GET /auth/me` – return profile

### Frontend
- Replace device ID–based auth with user login.
- Store JWT in secure storage.
- Handle deep links for magic link callbacks.

---

## Phase 2 – MongoDB Access Control

### Goal
Users can only access their own `Points` and `NotebookNodes`.

### Implementation
- Add `ownerId` (UUID string) to both collections.
- Create indexes: `{ ownerId: 1, createdAt: -1 }`.
- Backfill existing documents with correct owners.

### Backend Enforcement
- All Mongo queries must include `ownerId = currentUserId`.
- Implement an `OwnerScopedMongoTemplate` that enforces this automatically.
- Add integration tests to verify cross-user access is denied.

---

## Phase 3 – GitHub App Integration

### PostgreSQL Schema (Flyway `V2__github.sql`)
Tables:
- `github_installation`: stores user ↔ installation mapping.
- `github_cached_token`: optional cache of short-lived installation tokens (encrypted).
- `oauth_state`: prevents CSRF in browser flows.

### Token Flow
- Backend signs a JWT with the App’s private key.
- Exchanges JWT for an **installation token**.
- Optionally cache encrypted token + expiry in PostgreSQL.

### API Endpoints
- `GET /github/app/installation/urls` – returns GitHub installation URL.
- `POST /github/app/installation/callback` – validates and stores installation.
- `DELETE /github/app/installations/{id}` – disconnects user installation.

### Security
- Encrypt all tokens with Vault Transit.
- Never store plaintext tokens.
- Validate webhook signatures with `GITHUB_WEBHOOK_SECRET`.

---

## Phase 4 – Read-Only GitHub Operations

### HTTP Client
- Use short timeouts (≤5s) and retry with exponential backoff.
- Circuit breaker per installation.
- Use real GitHub rate-limit headers (`X-RateLimit-*`).

### Conditional Requests
- Store and reuse **ETags** for caching.
- Handle `304 Not Modified` and minimize API usage.

### API Endpoints
- `GET /github/installations` – list user installations.
- `GET /github/repos` – list repositories (paginated).
- `GET /github/trees` – list directory entries.
- `GET /github/file` – fetch file contents (limit ≤1–2 MB).

### Response Model
Return consistent JSON:
```json
{
  "code": "RATE_LIMITED",
  "message": "GitHub rate limit exceeded.",
  "details": { "reset": "2025-11-12T10:41:03Z" },
  "requestId": "..."
}
```

---

## Phase 5 – Safe Write Operations

### Principles
- Never write to default branch directly.
- Create branch: `testai/<YYYYMMDD>/<short-id>`.
- Commit message includes trace ID and is signed by GitHub App.

### Endpoint Flow
- `POST /github/write/prepare` – validate and fetch `baseSha`.
- `POST /github/write/commit` – create branch and commit (with optimistic concurrency).
- `POST /github/write/pr` – open PR with labels and trace ID.

### Conflict Handling
- Return 409 on outdated `sha`.
- Include diff and retry hints.

---

## Phase 6 – Flutter Integration

### Auth
- Email-based magic link login.
- JWT stored securely.
- Auto logout and refresh flow.

### GitHub Connect
- System browser OAuth (Custom Tabs / SFSafariViewController).
- Callback handled via deep link.
- UI for installations and repos with lazy pagination.

### UX Safety
- Lazy file loading.
- Size guards for large files.
- Clear feedback for rate limits and permission issues.

---

## Phase 7 – Observability & Ops

### Metrics
- Per-endpoint latency, error %, retry counts.
- GitHub rate-limit usage.
- Business metrics (PRs created, conflicts resolved).

### Logging & Auditing
- Structured logs (JSON) with `requestId`, `userId`, `installationId`.
- Never log secrets or file contents.
- Record key user actions in `audit_log`.

### Runbooks
- Procedures for rate limit handling, key rotation, and revocation.

---

## Phase 8 – Testing

### Unit Tests
- Mocked GitHub client.
- Token encryption/decryption.
- Access control enforcement.

### Integration Tests
- WireMock for rate limit & ETag tests.
- Migration validation.
- Owner ID access enforcement.

### E2E Tests
- Temporary GitHub org & repo for real API tests.
- Verify install → list → read → write → PR lifecycle.

---

## Rollout Sequence
1. **Phase 1:** User accounts and authentication.
2. **Phase 2:** Mongo owner scoping.
3. **Phase 3:** GitHub App integration.
4. **Phase 4:** Read-only endpoints.
5. **Phase 5:** Safe write/PR flow.
6. **Phase 6–8:** Flutter UI, observability, and testing.

---

## Summary
This phased plan establishes a secure, scalable foundation for integrating GitHub repositories into TestAI. It replaces device-based tokens with user accounts, encrypts secrets via Vault, adds per-user access control in Mongo, and implements a modern GitHub App integration model with safe write operations and observability baked in.

