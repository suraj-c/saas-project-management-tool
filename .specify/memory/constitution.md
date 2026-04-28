<!--
SYNC IMPACT REPORT
==================
Version Change: TEMPLATE → 1.0.0 (initial ratification)
Bump Rationale: MAJOR version 1.0.0 establishes the initial constitution for the project.

Modified Principles:
  - [PRINCIPLE_1_NAME] → I. Multi-Tenant Isolation (NON-NEGOTIABLE)
  - [PRINCIPLE_2_NAME] → II. Security & Role-Based Access Control
  - [PRINCIPLE_3_NAME] → III. Test-First Development (NON-NEGOTIABLE)
  - [PRINCIPLE_4_NAME] → IV. API-First & Contract-Driven
  - [PRINCIPLE_5_NAME] → V. Observability & Auditability

Added Sections:
  - Core Principles (5 principles)
  - Security & Compliance Standards
  - Development Workflow & Quality Gates
  - Governance

Removed Sections: None (initial creation)

Templates Requiring Updates:
  - ✅ .specify/templates/plan-template.md (Constitution Check section will reference these principles)
  - ✅ .specify/templates/spec-template.md (compatible — no changes required)
  - ✅ .specify/templates/tasks-template.md (compatible — supports test-first and observability tasks)
  - ✅ .specify/templates/checklist-template.md (compatible — no changes required)
  - ⚠ Runtime guidance file (CLAUDE.md) — pending creation if/when project context evolves

Deferred Items / TODOs:
  - TODO(RATIFICATION_DATE): Original adoption date set to today (2026-04-28) as project is newly initialized.
    Update if a different formal ratification date is established by stakeholders.
-->

# saas-project-management-tool Constitution

## Core Principles

### I. Multi-Tenant Isolation (NON-NEGOTIABLE)

Every feature, data model, query, and API endpoint MUST enforce strict tenant
isolation. No code path may access, return, or mutate data belonging to a
tenant other than the authenticated request's tenant context. Tenant context
MUST be derived from a verified authentication token and propagated through
all service layers — never accepted from client-supplied parameters.

**Rationale**: This is a multi-tenant SaaS product. A single cross-tenant
data leak constitutes a critical security incident and breach of customer
trust. Isolation must be designed in, tested explicitly, and verified at
every layer (database row-level filtering, API authorization, background
jobs, reporting queries).

**Rules**:
- Every persisted entity MUST include a `tenant_id` (or equivalent) and
  every query MUST filter by it.
- Cross-tenant tests MUST exist for every new endpoint or background job.
- Admin/internal tooling that crosses tenants MUST be explicitly flagged,
  audited, and restricted to authorized internal roles.

### II. Security & Role-Based Access Control

All user-facing functionality MUST enforce role-based access control (RBAC).
Authentication MUST use industry-standard mechanisms (OAuth 2.0 / OIDC or
signed session tokens). Authorization MUST be evaluated server-side for
every request — client-side checks are presentational only and never
authoritative. Sensitive operations (role changes, data exports, billing
actions) MUST emit audit log entries.

**Rationale**: User roles (admin, manager, member, viewer, guest) drive
visibility of boards, tasks, time entries, and reports. RBAC failures
directly translate into data exposure or unauthorized state changes.

**Rules**:
- Permission checks MUST occur at the API boundary AND in service layers
  for defense in depth.
- Role definitions and permission matrices MUST be documented and version-
  controlled alongside code.
- Secrets, credentials, and PII MUST never be logged in plaintext.

### III. Test-First Development (NON-NEGOTIABLE)

Test-Driven Development is mandatory for all production code paths. The
cycle is: write failing tests → obtain user/reviewer approval on test
intent → implement → refactor. Contract tests MUST exist for every API
endpoint. Integration tests MUST cover tenant isolation, RBAC enforcement,
and the primary user journeys for each feature (task boards, time tracking,
reporting).

**Rationale**: A project management tool stores customer-critical work
data. Regressions in task state, time entries, or report aggregation
directly damage customer workflows and trust. Tests are the contract that
prevents silent data corruption.

**Rules**:
- No production PR may merge without accompanying tests that fail prior to
  the implementation commit.
- Coverage of tenant isolation and RBAC paths is mandatory, not optional.
- Flaky tests MUST be fixed or quarantined within one sprint — they may
  not be ignored.

### IV. API-First & Contract-Driven

All features MUST be defined by an explicit API contract (OpenAPI or
equivalent schema) before implementation begins. Frontend, mobile, and
third-party integrations consume the same documented contract. Breaking
contract changes require a major version bump and a documented migration
path. The reporting dashboard and time-tracking UI MUST consume the same
public APIs available to customers.

**Rationale**: A SaaS product's API surface is a long-lived commitment.
Contract-first design prevents frontend/backend drift, enables parallel
development, and makes the product extensible (webhooks, integrations,
public API) without rework.

**Rules**:
- Contract artifacts live under `contracts/` in each feature spec.
- Contract tests MUST validate request/response schemas in CI.
- Deprecated endpoints MUST be supported for at least one major version
  with documented sunset dates.

### V. Observability & Auditability

Every service, background job, and user-impacting operation MUST emit
structured logs (JSON), metrics, and distributed traces. All authenticated
requests MUST be traceable end-to-end via correlation IDs that include
tenant_id and user_id (where permitted by privacy policy). Audit logs for
sensitive actions MUST be append-only, tamper-evident, and queryable per
tenant.

**Rationale**: Multi-tenant SaaS support, debugging, compliance (SOC 2,
GDPR), and incident response all depend on accurate, queryable telemetry.
Customers will request access logs, exported reports require integrity
guarantees, and on-call engineers cannot debug what they cannot see.

**Rules**:
- Structured logging is mandatory; unstructured `print`/`console.log` is
  prohibited in production code paths.
- Performance-sensitive paths (board load, report generation) MUST emit
  latency metrics with p50/p95/p99 SLOs documented.
- Audit log retention MUST meet or exceed contractual and regulatory
  minimums (default: 1 year for security events).

## Security & Compliance Standards

The following constraints apply to all features and infrastructure:

- **Data at rest**: All tenant data MUST be encrypted at rest using
  industry-standard algorithms (AES-256 or equivalent).
- **Data in transit**: TLS 1.2+ MUST be enforced on all external endpoints;
  internal service-to-service traffic MUST be encrypted or run on
  authenticated, isolated networks.
- **Secret management**: Secrets MUST live in a managed secrets store
  (e.g., AWS Secrets Manager, HashiCorp Vault) — never in source control,
  config files, or environment files committed to git.
- **Dependency hygiene**: Automated vulnerability scanning MUST run in CI;
  CRITICAL and HIGH CVEs MUST be remediated within defined SLAs (7 days
  CRITICAL, 30 days HIGH).
- **Privacy**: PII handling MUST comply with GDPR and applicable regional
  regulations; data export and deletion (right-to-erasure) workflows MUST
  be implemented per tenant.
- **Backups**: Tenant data MUST be backed up daily with tested restore
  procedures and a documented RPO/RTO.

## Development Workflow & Quality Gates

All changes flow through the following gates:

1. **Specification gate**: Every feature begins with a `spec.md` that
   captures user stories, acceptance scenarios, and measurable success
   criteria. Specifications are reviewed before planning begins.

2. **Planning gate**: A `plan.md` MUST verify Constitution Check (this
   document) before Phase 0 research and again after Phase 1 design.
   Violations require explicit justification in the Complexity Tracking
   section or the plan is rejected.

3. **Code review**: Every PR MUST be reviewed by at least one engineer
   other than the author. Reviewers MUST verify: (a) tenant isolation,
   (b) RBAC checks, (c) tests exist and meaningfully cover behavior,
   (d) observability hooks are in place, (e) no constitutional principle
   is violated without justification.

4. **CI gates**: PRs MUST pass: linting, type checks, unit tests,
   integration tests, contract tests, security scanning, and any
   feature-flagged smoke tests before merge.

5. **Deployment**: Production deploys MUST be gated by passing staging
   validation against the feature's quickstart scenarios. Rollback plans
   MUST exist for any migration that alters tenant data.

## Governance

This constitution supersedes ad-hoc practices and individual preferences.
All pull requests, design reviews, and architectural decisions MUST verify
compliance with the principles above. When a principle appears to conflict
with a delivery deadline, the principle wins by default; exceptions require
explicit, documented approval and a remediation plan.

**Amendment procedure**:
1. Propose changes via PR modifying this file with a Sync Impact Report.
2. Require approval from at least two senior engineers and the project
   owner.
3. Bump the constitution version per semantic rules:
   - **MAJOR**: Removing or redefining a principle in a backward-incompatible way.
   - **MINOR**: Adding a new principle or materially expanding guidance.
   - **PATCH**: Clarifications, wording fixes, non-semantic refinements.
4. Update dependent templates (`plan-template.md`, `spec-template.md`,
   `tasks-template.md`) in the same PR when amendments affect them.
5. Communicate amendments to the team and update onboarding materials.

**Compliance review**: A quarterly review MUST audit recent PRs for
constitutional compliance. Recurring violations trigger a process or
tooling improvement, not blame.

**Runtime guidance**: Day-to-day development guidance, conventions, and
agent-specific instructions live in `CLAUDE.md` (or equivalent context
file) at the repository root. That file MUST remain consistent with this
constitution; in case of conflict, the constitution prevails.

**Version**: 1.0.0 | **Ratified**: 2026-04-28 | **Last Amended**: 2026-04-28