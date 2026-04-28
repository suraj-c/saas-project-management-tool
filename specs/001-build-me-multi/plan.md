# Implementation Plan: Multi-Tenant SaaS Project Management Tool

**Branch**: `001-build-me-multi` | **Date**: 2026-04-28 | **Spec**: [spec.md](./spec.md)
**Input**: Feature specification from `/specs/001-build-me-multi/spec.md`

## Summary

Build a multi-tenant SaaS project management tool delivering four prioritized capabilities: (P1) tenant sign-up with role-based team onboarding and strict workspace isolation; (P1) Kanban-style task boards with near-real-time collaboration; (P2) time tracking via timers and manual entries; (P3) a reporting dashboard with filtering and CSV export.

Technical approach: a contract-first web application with a TypeScript/Node.js backend (NestJS) exposing an OpenAPI-defined REST API plus a WebSocket channel for live board updates, a React/TypeScript frontend, and PostgreSQL with row-level security for tenant isolation. Authentication uses OIDC (email/password + Google OAuth) issuing short-lived JWTs that carry verified `tenant_id` and `user_id` claims. RBAC is enforced server-side at API and service layers. Structured logging, OpenTelemetry tracing, Prometheus metrics, and an append-only audit log fulfill the observability principle. TDD is mandatory: contract tests, tenant-isolation tests, and RBAC tests precede every implementation commit.

## Technical Context

**Language/Version**: TypeScript 5.4 (Node.js 20 LTS) for backend and frontend
**Primary Dependencies**: NestJS 10 (backend framework), Prisma 5 (ORM with RLS support), Socket.IO 4 (WebSockets), React 18 + Vite + TanStack Query (frontend), Zod (schema validation), Passport + openid-client (auth), BullMQ (background jobs)
**Storage**: PostgreSQL 16 with Row-Level Security policies enforcing `tenant_id`; Redis 7 for sessions, rate limiting, queues, and pub/sub for WebSocket fan-out; S3-compatible object storage for exports and attachments
**Testing**: Jest (unit + integration), Supertest (API contract tests against OpenAPI), Playwright (end-to-end), `pact` (contract verification optional for future integrators), Testcontainers (Postgres/Redis for integration)
**Target Platform**: Linux containers (Docker) deployed to a managed Kubernetes service; modern evergreen browsers (Chrome, Edge, Firefox, Safari) for the web client
**Project Type**: Web application (frontend + backend services)
**Performance Goals**: Board interactions reflected to other clients within 5 s p95 (per SC-004); dashboard initial render < 3 s p95 over 12 months of typical data (SC-007); CSV export of 50k time entries < 10 s (SC-008); 500 concurrent users per tenant, 10k platform-wide (SC-009); 99.9% monthly uptime (SC-010)
**Constraints**: Strict tenant isolation (NON-NEGOTIABLE); TLS 1.2+ on all external endpoints; AES-256 at rest; structured JSON logs only (no plaintext PII); audit log retention ≥ 1 year; single-region data residency for v1; English-only UI for v1
**Scale/Scope**: v1 targets small-to-medium organizations (up to a few hundred members per tenant); ~30–40 REST endpoints, ~10 core entities, ~25–35 UI screens, one OpenAPI contract document, one WebSocket event schema

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle | Status | Evidence |
|---|---|---|
| I. Multi-Tenant Isolation (NON-NEGOTIABLE) | PASS | Every entity in `data-model.md` carries `tenant_id`; Postgres RLS policies enforce filtering at the DB layer; tenant context is derived server-side from JWT claims, never from client params; cross-tenant tests are mandatory in the test plan. |
| II. Security & RBAC | PASS | OIDC auth with short-lived JWTs; RBAC roles (admin/manager/member/viewer) evaluated at API gateway and service layer (defense in depth); audit log entries emitted for sensitive actions per `contracts/openapi.yaml` and `data-model.md`. |
| III. Test-First Development (NON-NEGOTIABLE) | PASS | `quickstart.md` mandates failing tests before implementation; contract tests bound to OpenAPI; tenant-isolation and RBAC integration tests required for every endpoint. |
| IV. API-First & Contract-Driven | PASS | `contracts/openapi.yaml` defines all REST endpoints; `contracts/websocket-events.md` defines live event schema; both are authored before implementation begins; CI runs schema validation. |
| V. Observability & Auditability | PASS | Structured JSON logs, OpenTelemetry traces with correlation IDs (including `tenant_id`/`user_id`), Prometheus metrics with documented SLOs, append-only `audit_log` table with ≥1y retention. |

**Initial Gate Result**: PASS — proceed to Phase 0.

**Post-Design Gate Result** (after Phase 1): PASS — design artifacts uphold all five principles. RLS in `data-model.md` reinforces isolation; OpenAPI contract codifies RBAC and audit behavior; `quickstart.md` codifies the TDD workflow. No deviations recorded; **Complexity Tracking** section below remains empty.

## Project Structure

### Documentation (this feature)