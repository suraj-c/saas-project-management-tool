# Phase 0 Research: Multi-Tenant SaaS Project Management Tool

**Feature**: 001-build-me-multi
**Date**: 2026-04-28

This document resolves all `NEEDS CLARIFICATION` items implied by the spec and records technology decisions with rationale and alternatives.

## R1. Tenant isolation strategy

- **Decision**: Shared database, shared schema, with PostgreSQL **Row-Level Security (RLS)** policies keyed on `tenant_id`. Application sets `SET LOCAL app.tenant_id = '<uuid>'` per transaction from the verified JWT claim. Every table that holds tenant data carries a `tenant_id` column with an RLS policy `USING (tenant_id = current_setting('app.tenant_id')::uuid)`.
- **Rationale**: Defense in depth — even if an application-layer bug forgets a `WHERE tenant_id = ?` clause, the database refuses cross-tenant rows. Operationally simpler than schema-per-tenant or database-per-tenant for SMB scale (Assumption: up to a few hundred members per tenant in v1). Honors **Principle I (Multi-Tenant Isolation, NON-NEGOTIABLE)**.
- **Alternatives considered**:
  - *Schema-per-tenant*: Strong isolation but expensive migrations and harder cross-tenant analytics; overkill for v1 SMB scale.
  - *Database-per-tenant*: Strongest isolation; high operational overhead and per-tenant cost; rejected for v1.
  - *Application-only filtering*: Single point of failure; one missed `WHERE` clause leaks data; rejected for violating defense-in-depth.

## R2. Authentication and identity

- **Decision**: OIDC-based authentication with email/password (managed via a hosted IdP such as Auth0, Clerk, or self-hosted Keycloak) and Google OAuth. Server issues short-lived (15-minute) access JWTs and longer-lived (7-day) rotating refresh tokens stored in HttpOnly, Secure, SameSite=Lax cookies. Each access JWT carries `sub` (user_id), `tid` (tenant_id), `role`, and `mid` (membership_id) claims. Tenant context is **never** accepted from request bodies/params.
- **Rationale**: OIDC is industry-standard, supports SSO expansion (per Assumption that SAML/SCIM is deferred), and aligns with **Principle II (Security & RBAC)**. Short-lived access tokens limit blast radius; HttpOnly refresh cookies mitigate XSS token theft. A single email may have multiple memberships across tenants (Edge Case in spec) — handled by requiring tenant selection at login when ambiguous.
- **Alternatives considered**:
  - *Server-side sessions only (no JWT)*: Simpler but requires sticky sessions or shared session store for WebSocket auth; chose hybrid (session cookie holding refresh token, JWT for API auth) for scalability.
  - *Long-lived JWTs*: Rejected; cannot revoke quickly enough for role changes (FR-008 requires immediate effect).
  - *Custom auth*: Rejected; reinvents wheels and adds security risk.

## R3. RBAC enforcement model

- **Decision**: Two-layer enforcement.
  1. **API gateway / controller layer**: NestJS `@Roles()` decorator + `RolesGuard` rejects requests whose JWT role does not satisfy the endpoint's minimum role.
  2. **Service layer**: For resource-scoped permissions (e.g., "manager of *this* project"), services call a `permission.check(user, action, resource)` function that re-validates against current DB state.

  Permission matrix lives in `backend/src/common/rbac/permissions.ts` and is unit-tested.
- **Rationale**: Defense in depth per **Principle II**. Controller-layer checks are fast and catch obvious misuse; service-layer checks handle dynamic, resource-specific authorization (e.g., a manager can edit time entries only for projects they own).
- **Alternatives considered**:
  - *Single-layer (controller only)*: Insufficient for resource-scoped rules; rejected.
  - *Policy engine (OPA/Cedar)*: Powerful but overkill for v1's small permission matrix; revisit if matrix grows.

## R4. Real-time board updates

- **Decision**: WebSocket via Socket.IO 4. The backend exposes a `/realtime` namespace; on connect, the client presents its JWT, the server validates it and joins the socket to room `tenant:<tid>:project:<pid>`. Domain events (task moved, comment added, etc.) are published to Redis pub/sub by the writer process; the gateway fans out to room subscribers. Reconnects use exponential backoff with token refresh.
- **Rationale**: Achieves SC-004 (updates within 5 s p95). Redis pub/sub allows horizontal scaling of gateway pods. Room-per-project keeps fan-out scoped and prevents leaking events to unauthorized projects (reinforces **Principle I**).
- **Alternatives considered**:
  - *Server-Sent Events*: Simpler but unidirectional; would still need a separate channel for client → server presence/typing.
  - *Long polling*: High latency, hard to meet 5 s p95 reliably under load.
  - *Raw WebSocket (no Socket.IO)*: More bare-metal but loses reconnection, room, and ack primitives that Socket.IO provides.

## R5. Database & migration strategy

- **Decision**: PostgreSQL 16 managed service (e.g., RDS, Cloud SQL). **Prisma 5** as ORM/migration tool. RLS policies are authored as raw SQL in Prisma migration files alongside generated DDL. A small wrapper around `PrismaClient` ensures every request transaction begins with `SET LOCAL app.tenant_id`.
- **Rationale**: Prisma gives type-safe queries (reducing the chance of cross-tenant bugs), good migration tooling, and is well-supported in NestJS. Postgres RLS is a first-class feature and battle-tested.
- **Alternatives considered**:
  - *TypeORM*: More flexible but historically less reliable migrations and weaker type inference.
  - *Drizzle*: Promising but younger ecosystem; deferred.
  - *Raw SQL only*: Maximum control, minimum safety; rejected for SMB-scale dev velocity.

## R6. Frontend stack

- **Decision**: React 18 + Vite + TypeScript. **TanStack Query** for server state (cache + invalidation), **Zustand** for small client-only state (e.g., timer state), **React Router** for routing, **Tailwind CSS** + a headless UI primitives library (Radix) for the design system. API client generated from `openapi.yaml` via `openapi-typescript` + a fetch wrapper.
- **Rationale**: React + Vite is a productive, modern combo with strong typing. Generating clients from OpenAPI enforces **Principle IV (API-First)**.
- **Alternatives considered**:
  - *Next.js*: Excellent for SSR and SEO, but a SaaS PM tool is mostly authenticated SPA — the SSR overhead isn't justified for v1.
  - *Vue/Svelte*: Capable but team familiarity and ecosystem favor React.

## R7. CSV export and long-running jobs

- **Decision**: Reporting and CSV exports run as background jobs in **BullMQ** (Redis-backed). The API enqueues an `export-report` job and returns a job ID; the client polls `GET /reports/exports/{id}` (or receives a WebSocket notification). Completed exports are uploaded to S3-compatible storage and served via short-lived signed URLs (15 min).
- **Rationale**: Meets SC-008 (50k entries < 10 s) without holding HTTP connections open and without blocking API workers. Signed URLs avoid streaming large files through the API, reducing cost and improving reliability.
- **Alternatives considered**:
  - *Synchronous streaming response*: Works for small exports but breaks for 50k+ rows over flaky networks; rejected.
  - *Heavy data warehouse (Snowflake/BigQuery)*: Overkill for v1; revisit when reporting becomes the primary product surface.

## R8. Observability stack

- **Decision**: Structured JSON logging via **Pino**; distributed tracing via **OpenTelemetry** with OTLP exporter (e.g., to Honeycomb, Tempo, or Datadog APM); metrics via **Prometheus** scrape endpoint. Every log line includes `correlation_id`, `tenant_id`, `user_id` (where present and not restricted by privacy policy), and `request_id`.
- **Rationale**: Fulfills **Principle V (Observability & Auditability)**. OpenTelemetry is vendor-neutral, allowing future backend changes without re-instrumentation.
- **Alternatives considered**:
  - *Vendor-specific agents (Datadog-only)*: Vendor lock-in; rejected.
  - *Unstructured `console.log`*: Explicitly prohibited by **Principle V**.

## R9. Audit logging

- **Decision**: Append-only `audit_log` table per tenant scope (single physical table, partitioned monthly, with `tenant_id` and RLS). Insertion is performed via a stored procedure that writes the row + a SHA-256 hash chain (each row's hash includes the previous row's hash) for tamper evidence. No `UPDATE` or `DELETE` grants on the table outside a privileged DBA migration role.
- **Rationale**: Satisfies **Principle V**'s "append-only, tamper-evident, queryable per tenant" requirement and the spec's 1-year retention floor.
- **Alternatives considered**:
  - *External SIEM only*: Adds dependency for tenant-facing queries; we still need an internal queryable copy.
  - *Per-tenant table*: Operational explosion; rejected.

## R10. Notifications

- **Decision**: In-app notifications via the same WebSocket channel (event type `notification.created`) plus persisted in a `notification` table for unread badge counts. Email delivery via a transactional email provider (e.g., Postmark, SES) using BullMQ jobs and idempotency keys.
- **Rationale**: Reuses real-time infrastructure; idempotency keys prevent duplicate emails on job retry.
- **Alternatives considered**:
  - *Polling for unread count*: Fine but adds load; reuse of WebSocket is essentially free.

## R11. Rate limiting and abuse handling

- **Decision**: Token-bucket rate limiting per `tenant_id` and per `user_id` enforced at the API gateway (using `@nestjs/throttler` backed by Redis). Global limits (per IP) layer beneath. Abnormal spikes for a single tenant trigger graceful 429 responses and an internal alert.
- **Rationale**: Edge-case requirement: one tenant cannot degrade others' performance (spec edge case).
- **Alternatives considered**:
  - *Per-IP only*: Insufficient — a single tenant behind a corporate NAT could still degrade others.

## R12. Data export & right-to-erasure

- **Decision**: Tenant export = a workspace dump (JSON + CSV bundles) generated by a background job and delivered via signed URL. Right-to-erasure (per FR-026) replaces user PII (`name`, `email`) with anonymized placeholders while preserving foreign keys and aggregated metrics for reporting integrity.
- **Rationale**: Aligns with the **Security & Compliance Standards** section of the constitution (GDPR), and the spec's edge case on data deletion.
- **Alternatives considered**:
  - *Hard delete*: Breaks historical reporting; rejected.

## R13. Branch / column workflow customization

- **Decision**: Boards have an ordered list of `Column` rows; default columns ("To Do", "In Progress", "Done") seeded on board creation. Tasks store `column_id` and `position` (a fractional index, e.g., LexoRank-style) to allow O(1) reordering without rewriting siblings.
- **Rationale**: Matches Story 2 acceptance scenarios; LexoRank is a well-known pattern for collaborative drag-and-drop ordering.
- **Alternatives considered**:
  - *Integer position with reindex on insert*: Causes write storms during rapid reorder; rejected.

## R14. Time entry concurrency (running timers)

- **Decision**: A user may have at most one running timer at any time across the entire workspace. Starting a new timer auto-stops the previous one (server-side enforcement via partial unique index `WHERE stopped_at IS NULL`). Timer state is reflected to other clients via WebSocket.
- **Rationale**: Matches user expectations and prevents double-counted time. Server-side enforcement aligns with **Principle III** (deterministic, testable behavior).
- **Alternatives considered**:
  - *Allow multiple running timers*: Confusing reporting and edge cases; rejected.

## R15. Testing pyramid and gates

- **Decision**:
  - Unit tests: business logic, RBAC matrix, position arithmetic.
  - Integration tests: every endpoint must have (a) a happy-path test, (b) a tenant-isolation negative test (foreign tenant returns 404/403), (c) an RBAC negative test (insufficient role returns 403).
  - Contract tests: validate request/response shapes against `openapi.yaml` in CI.
  - End-to-end (Playwright): the four user-story acceptance flows from `spec.md`.
  - Load test: a baseline scenario hitting SC-004, SC-007, SC-009 thresholds, run on a schedule.
- **Rationale**: Operationalizes **Principle III (Test-First, NON-NEGOTIABLE)** and the constitution's CI gates.

## Open Items

None. All `NEEDS CLARIFICATION` placeholders in the plan template have been resolved by the decisions above.