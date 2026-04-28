---
description: "Task list for Multi-Tenant SaaS Project Management Tool"
---

# Tasks: Multi-Tenant SaaS Project Management Tool

**Input**: Design documents from `/specs/001-build-me-multi/`
**Prerequisites**: plan.md ✅, spec.md ✅, research.md ✅, data-model.md ✅, contracts/openapi.yaml ✅

**Tests**: Tests are MANDATORY per Constitution Principle III (Test-First Development, NON-NEGOTIABLE). Every endpoint requires (a) contract test, (b) tenant-isolation negative test, (c) RBAC negative test before implementation.

**Organization**: Tasks are grouped by user story to enable independent implementation and testing of each story.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (US1, US2, US3, US4)
- All file paths are relative to repository root

## Path Conventions

This is a **web application** with backend + frontend split:
- Backend: `backend/src/`, `backend/tests/`
- Frontend: `frontend/src/`, `frontend/tests/`
- Contracts: `specs/001-build-me-multi/contracts/`

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Project scaffolding, tooling, and dev environment

- [ ] T001 Create monorepo structure: `backend/`, `frontend/`, `infra/`, `docs/` directories at repo root per plan.md
- [ ] T002 Initialize backend project: `cd backend && npm init`, install NestJS 10, TypeScript 5.4, Prisma 5, Pino, Zod, Passport, openid-client, Socket.IO 4, BullMQ, @nestjs/throttler in `backend/package.json`
- [ ] T003 Initialize frontend project: `cd frontend && npm create vite@latest . -- --template react-ts`, install React 18, TanStack Query, Zustand, React Router, Tailwind CSS, Radix UI, openapi-typescript in `frontend/package.json`
- [ ] T004 [P] Configure ESLint + Prettier for backend in `backend/.eslintrc.cjs` and `backend/.prettierrc`
- [ ] T005 [P] Configure ESLint + Prettier for frontend in `frontend/.eslintrc.cjs` and `frontend/.prettierrc`
- [ ] T006 [P] Configure TypeScript strict mode in `backend/tsconfig.json` and `frontend/tsconfig.json`
- [ ] T007 [P] Create `docker-compose.yml` at repo root with PostgreSQL 16, Redis 7, MinIO (S3-compatible) services
- [ ] T008 [P] Create `.env.example` files in `backend/.env.example` and `frontend/.env.example` with required environment variables (DATABASE_URL, REDIS_URL, JWT_SECRET, OIDC_ISSUER, etc.)
- [ ] T009 [P] Configure Jest for backend in `backend/jest.config.ts` with Testcontainers support
- [ ] T010 [P] Configure Vitest for frontend unit tests in `frontend/vitest.config.ts`
- [ ] T011 [P] Configure Playwright for end-to-end tests in `e2e/playwright.config.ts`
- [ ] T012 [P] Set up GitHub Actions CI workflow in `.github/workflows/ci.yml` running lint, typecheck, unit tests, integration tests, contract tests, and OpenAPI schema validation

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Core infrastructure that MUST be complete before ANY user story can be implemented

**⚠️ CRITICAL**: No user story work can begin until this phase is complete

### Database & ORM Foundation

- [ ] T013 Initialize Prisma in `backend/prisma/schema.prisma` with PostgreSQL provider and base configuration
- [ ] T014 Create initial Prisma migration for `tenant`, `user`, `membership` core tables in `backend/prisma/migrations/0001_init/migration.sql`
- [ ] T015 Author RLS policies migration in `backend/prisma/migrations/0002_rls_policies/migration.sql` enabling Row-Level Security on all tenant-scoped tables with `USING (tenant_id = current_setting('app.tenant_id')::uuid)`
- [ ] T016 Implement `TenantAwarePrismaService` in `backend/src/common/prisma/tenant-aware-prisma.service.ts` that wraps `PrismaClient` and runs `SET LOCAL app.tenant_id` on every request transaction
- [ ] T017 Write integration test for RLS enforcement in `backend/tests/integration/rls.spec.ts` proving cross-tenant queries return zero rows even with bypassed app filters

### Authentication & Authorization Foundation

- [ ] T018 [P] Implement JWT module in `backend/src/auth/jwt.module.ts` issuing access tokens (15min) and refresh tokens (7d) with `sub`, `tid`, `role`, `mid` claims
- [ ] T019 [P] Implement OIDC strategy (email/password + Google OAuth) in `backend/src/auth/strategies/oidc.strategy.ts` using openid-client
- [ ] T020 Implement `AuthGuard` in `backend/src/auth/guards/auth.guard.ts` validating JWT and attaching `{userId, tenantId, role, membershipId}` to request
- [ ] T021 Implement `TenantContextMiddleware` in `backend/src/common/middleware/tenant-context.middleware.ts` that extracts `tid` from JWT and sets `app.tenant_id` on the DB transaction
- [ ] T022 [P] Implement RBAC `@Roles()` decorator and `RolesGuard` in `backend/src/common/rbac/` with role hierarchy (admin > manager > member > viewer)
- [ ] T023 [P] Implement permission matrix in `backend/src/common/rbac/permissions.ts` with `permission.check(user, action, resource)` for resource-scoped checks
- [ ] T024 Write unit tests for permission matrix in `backend/tests/unit/rbac/permissions.spec.ts` covering every role × action combination

### API & Routing Foundation

- [ ] T025 Set up NestJS app bootstrap in `backend/src/main.ts` with global prefix `/api/v1`, CORS config, helmet, and structured error filter
- [ ] T026 [P] Implement global validation pipe with Zod in `backend/src/common/pipes/zod-validation.pipe.ts`
- [ ] T027 [P] Implement global exception filter emitting RFC 7807 problem-details responses in `backend/src/common/filters/problem-details.filter.ts`
- [ ] T028 [P] Configure `@nestjs/throttler` with Redis backend and per-tenant + per-user rate limits in `backend/src/common/throttler/throttler.module.ts`
- [ ] T029 Set up OpenAPI contract validation middleware in `backend/src/common/openapi/openapi-validator.middleware.ts` that validates requests/responses against `specs/001-build-me-multi/contracts/openapi.yaml` in dev/test

### Observability Foundation

- [ ] T030 [P] Configure Pino structured JSON logger in `backend/src/common/logging/logger.module.ts` injecting `correlation_id`, `tenant_id`, `user_id`, `request_id` into every log line
- [ ] T031 [P] Configure OpenTelemetry tracing with OTLP exporter in `backend/src/common/tracing/tracing.module.ts`
- [ ] T032 [P] Expose Prometheus metrics endpoint at `/metrics` in `backend/src/common/metrics/metrics.module.ts` with HTTP histogram, DB query timing, and custom counters
- [ ] T033 Implement append-only `audit_log` table migration with monthly partitioning and SHA-256 hash chain stored procedure in `backend/prisma/migrations/0003_audit_log/migration.sql`
- [ ] T034 Implement `AuditLogService` in `backend/src/common/audit/audit-log.service.ts` exposing `record(actor, action, resource, metadata)` invoking the stored procedure

### Background Jobs & Real-time Foundation

- [ ] T035 [P] Configure BullMQ module in `backend/src/common/queue/queue.module.ts` with Redis connection
- [ ] T036 [P] Set up Socket.IO gateway scaffold in `backend/src/realtime/realtime.gateway.ts` with `/realtime` namespace, JWT authentication on connect, and Redis adapter for pub/sub fan-out
- [ ] T037 [P] Configure S3-compatible storage client in `backend/src/common/storage/storage.module.ts` for exports and attachments

### Frontend Foundation

- [ ] T038 [P] Generate TypeScript API client from OpenAPI in `frontend/src/api/generated.ts` using `openapi-typescript`
- [ ] T039 [P] Implement authenticated fetch wrapper with refresh token rotation in `frontend/src/api/client.ts`
- [ ] T040 [P] Set up TanStack Query provider and React Router in `frontend/src/App.tsx`
- [ ] T041 [P] Implement auth context and protected route component in `frontend/src/auth/AuthContext.tsx` and `frontend/src/auth/ProtectedRoute.tsx`
- [ ] T042 [P] Set up Tailwind CSS with design tokens in `frontend/tailwind.config.ts` and base layout component in `frontend/src/components/layout/AppShell.tsx`

**Checkpoint**: Foundation ready — user story implementation can now begin in parallel

---

## Phase 3: User Story 1 - Tenant Sign-Up and Team Onboarding (Priority: P1) 🎯 MVP

**Goal**: A new organization can sign up, create a tenant workspace, invite teammates with roles, and confirm strict tenant isolation.

**Independent Test**: Register a new organization → invite a teammate → teammate accepts and signs in → verify they see only their workspace's data and a user from a different tenant cannot access it.

### Tests for User Story 1 (write FIRST, MUST FAIL before implementation) ⚠️

- [ ] T043 [P] [US1] Contract test for `POST /api/v1/auth/signup` in `backend/tests/contract/auth-signup.spec.ts`
- [ ] T044 [P] [US1] Contract test for `POST /api/v1/auth/login` in `backend/tests/contract/auth-login.spec.ts`
- [ ] T045 [P] [US1] Contract test for `POST /api/v1/auth/refresh` in `backend/tests/contract/auth-refresh.spec.ts`
- [ ] T046 [P] [US1] Contract test for `POST /api/v1/memberships/invite` in `backend/tests/contract/memberships-invite.spec.ts`
- [ ] T047 [P] [US1] Contract test for `POST /api/v1/memberships/accept` in `backend/tests/contract/memberships-accept.spec.ts`
- [ ] T048 [P] [US1] Contract test for `GET /api/v1/memberships` (list members) in `backend/tests/contract/memberships-list.spec.ts`
- [ ] T049 [P] [US1] Contract test for `PATCH /api/v1/memberships/{id}` (change role) in `backend/tests/contract/memberships-update.spec.ts`
- [ ] T050 [P] [US1] Contract test for `DELETE /api/v1/memberships/{id}` (remove user) in `backend/tests/contract/memberships-remove.spec.ts`
- [ ] T051 [P] [US1] Tenant-isolation integration test in `backend/tests/integration/us1-tenant-isolation.spec.ts` proving Tenant A cannot read/write Tenant B memberships
- [ ] T052 [P] [US1] RBAC negative test in `backend/tests/integration/us1-rbac.spec.ts` proving non-admin cannot invite/remove members or change roles
- [ ] T053 [P] [US1] Edge case test in `backend/tests/integration/us1-multi-tenant-email.spec.ts` proving same email can hold memberships in multiple tenants
- [ ] T054 [P] [US1] Audit log test in `backend/tests/integration/us1-audit.spec.ts` proving role changes and member removals emit audit entries

### Implementation for User Story 1

- [ ] T055 [P] [US1] Implement `Tenant` Prisma model in `backend/prisma/schema.prisma` (id, name, slug, plan, created_at)
- [ ] T056 [P] [US1] Implement `User` Prisma model (id, email, hashed_password, name, oauth_provider, oauth_subject) — global, no tenant_id
- [ ] T057 [P] [US1] Implement `Membership` Prisma model (id, tenant_id, user_id, role, status, invited_by, invited_at, accepted_at) with unique constraint on (tenant_id, user_id)
- [ ] T058 [US1] Run migration: `cd backend && npx prisma migrate dev --name us1_tenant_user_membership`
- [ ] T059 [P] [US1] Implement `TenantService` in `backend/src/tenants/tenant.service.ts` with `create(name, founderEmail, founderPassword)` provisioning tenant + admin membership atomically
- [ ] T060 [P] [US1] Implement `UserService` in `backend/src/users/user.service.ts` with `findOrCreateByEmail`, `verifyPassword`, `hashPassword` (bcrypt)
- [ ] T061 [US1] Implement `AuthController` (`POST /auth/signup`, `POST /auth/login`, `POST /auth/refresh`, `POST /auth/logout`) in `backend/src/auth/auth.controller.ts` (depends on T055-T060)
- [ ] T062 [US1] Implement tenant-selection flow in `backend/src/auth/auth.service.ts` for users with multiple memberships (returns list of tenants on login when ambiguous, then issues JWT for chosen tenant)
- [ ] T063 [P] [US1] Implement `MembershipService` in `backend/src/memberships/membership.service.ts` with `invite`, `accept`, `list`, `updateRole`, `remove`
- [ ] T064 [US1] Implement `MembershipController` (`POST /memberships/invite`, `POST /memberships/accept`, `GET /memberships`, `PATCH /memberships/{id}`, `DELETE /memberships/{id}`) in `backend/src/memberships/membership.controller.ts` with `@Roles('admin')` guards on mutating endpoints
- [ ] T065 [US1] Implement invitation email job in `backend/src/memberships/jobs/send-invitation.job.ts` using BullMQ + Postmark/SES (idempotency key = invitation_id)
- [ ] T066 [US1] Hook audit logging into role changes, invitations, and removals via `AuditLogService` calls in `MembershipService`
- [ ] T067 [US1] Implement session invalidation on member removal: revoke refresh tokens for the removed (user_id, tenant_id) pair in `backend/src/auth/session-revocation.service.ts`
- [ ] T068 [P] [US1] Build sign-up page in `frontend/src/pages/SignUp.tsx` (organization name + admin email/password form)
- [ ] T069 [P] [US1] Build sign-in page with tenant selector in `frontend/src/pages/SignIn.tsx`
- [ ] T070 [P] [US1] Build invitation accept page in `frontend/src/pages/AcceptInvitation.tsx` (reads invite token from URL, prompts for password if new user)
- [ ] T071 [P] [US1] Build workspace settings → Members tab in `frontend/src/pages/settings/Members.tsx` with invite form, members list, role dropdowns, and remove buttons
- [ ] T072 [US1] Add structured logging and correlation IDs to all auth + membership endpoints

**Checkpoint**: User Story 1 fully functional — a new org can sign up, invite teammates, assign roles, and tenant isolation is verified by tests.

---

## Phase 4: User Story 2 - Task Boards for Project Work (Priority: P1)

**Goal**: Teams can create projects, add Kanban boards with columns, create/move/assign/comment on tasks, with near-real-time updates within 5s.

**Independent Test**: Manager creates project + board → adds tasks → assigns to members → drags task across columns → other members see updates within 5s. Viewer cannot mutate. Comments notify assignees.

### Tests for User Story 2 (write FIRST, MUST FAIL before implementation) ⚠️

- [ ] T073 [P] [US2] Contract test for `POST /api/v1/projects` in `backend/tests/contract/projects-create.spec.ts`
- [ ] T074 [P] [US2] Contract test for `GET /api/v1/projects` and `GET /api/v1/projects/{id}` in `backend/tests/contract/projects-read.spec.ts`
- [ ] T075 [P] [US2] Contract test for `PATCH /api/v1/projects/{id}` and `DELETE /api/v1/projects/{id}` (archive) in `backend/tests/contract/projects-update.spec.ts`
- [ ] T076 [P] [US2] Contract test for `POST /api/v1/projects/{id}/boards` and `GET /api/v1/boards/{id}` in `backend/tests/contract/boards.spec.ts`
- [ ] T077 [P] [US2] Contract test for `POST /api/v1/boards/{id}/columns`, `PATCH /api/v1/columns/{id}`, `DELETE /api/v1/columns/{id}` in `backend/tests/contract/columns.spec.ts`
- [ ] T078 [P] [US2] Contract test for `POST /api/v1/projects/{id}/tasks` in `backend/tests/contract/tasks-create.spec.ts`
- [ ] T079 [P] [US2] Contract test for `GET /api/v1/projects/{id}/tasks`, `GET /api/v1/tasks/{id}` in `backend/tests/contract/tasks-read.spec.ts`
- [ ] T080 [P] [US2] Contract test for `PATCH /api/v1/tasks/{id}` (edit + move + reassign), `DELETE /api/v1/tasks/{id}` in `backend/tests/contract/tasks-update.spec.ts`
- [ ] T081 [P] [US2] Contract test for `POST /api/v1/tasks/{id}/comments`, `GET /api/v1/tasks/{id}/comments` in `backend/tests/contract/comments.spec.ts`
- [ ] T082 [P] [US2] Tenant-isolation integration test in `backend/tests/integration/us2-tenant-isolation.spec.ts` (Tenant A cannot read/edit Tenant B projects/boards/tasks)
- [ ] T083 [P] [US2] RBAC negative test in `backend/tests/integration/us2-rbac.spec.ts` (viewer cannot create/edit/move/delete; member cannot delete others' tasks)
- [ ] T084 [P] [US2] Real-time integration test in `backend/tests/integration/us2-realtime.spec.ts` proving WebSocket clients in the same project room receive task-moved events within 5s
- [ ] T085 [P] [US2] Soft-delete preservation test in `backend/tests/integration/us2-task-soft-delete.spec.ts` (deleting a task with time entries preserves the task reference)
- [ ] T086 [P] [US2] Concurrent edit test in `backend/tests/integration/us2-concurrent-edit.spec.ts` (last-write-wins on column status; both edits in activity log)
- [ ] T087 [P] [US2] LexoRank position test in `backend/tests/unit/lexorank.spec.ts` covering insert-between, edge cases, and rebalancing trigger

### Implementation for User Story 2

- [ ] T088 [P] [US2] Implement `Project` Prisma model (id, tenant_id, name, description, owner_membership_id, status, deleted_at) with RLS policy
- [ ] T089 [P] [US2] Implement `Board` Prisma model (id, tenant_id, project_id, name) with RLS policy
- [ ] T090 [P] [US2] Implement `Column` Prisma model (id, tenant_id, board_id, name, position INT, wip_limit nullable) with RLS policy
- [ ] T091 [P] [US2] Implement `Task` Prisma model (id, tenant_id, project_id, board_id, column_id, title, description, assignee_membership_id, reporter_membership_id, due_date, priority, position TEXT for LexoRank, deleted_at) with RLS policy
- [ ] T092 [P] [US2] Implement `Comment` Prisma model (id, tenant_id, task_id, author_membership_id, content, created_at) with RLS policy
- [ ] T093 [P] [US2] Implement `Activity` Prisma model (id, tenant_id, task_id, actor_membership_id, action, before_json, after_json, created_at) with RLS policy
- [ ] T094 [P] [US2] Implement `Notification` Prisma model (id, tenant_id, recipient_membership_id, event_type, payload_json, read_at, delivered_channels) with RLS policy
- [ ] T095 [US2] Run migration: `cd backend && npx prisma migrate dev --name us2_projects_boards_tasks`
- [ ] T096 [P] [US2] Implement LexoRank utility in `backend/src/common/lexorank/lexorank.ts` (between, head, tail, rebalance)
- [ ] T097 [P] [US2] Implement `ProjectService` in `backend/src/projects/project.service.ts` (create, list, get, update, archive)
- [ ] T098 [P] [US2] Implement `BoardService` in `backend/src/boards/board.service.ts` (create with default columns, get, update); seeds "To Do", "In Progress", "Done" columns on creation
- [ ] T099 [P] [US2] Implement `ColumnService` in `backend/src/columns/column.service.ts` (create, reorder, update, delete with reassignment)
- [ ] T100 [P] [US2] Implement `TaskService` in `backend/src/tasks/task.service.ts` (create, list with filters, get, update, move with LexoRank, soft-delete) — depends on T096
- [ ] T101 [P] [US2] Implement `CommentService` in `backend/src/comments/comment.service.ts` (create, list, parse @-mentions)
- [ ] T102 [P] [US2] Implement `ActivityService` in `backend/src/activity/activity.service.ts` recording every task mutation (including concurrent edits)
- [ ] T103 [US2] Implement `ProjectController` in `backend/src/projects/project.controller.ts` with role guards (admin/manager create; member read; viewer read-only)
- [ ] T104 [US2] Implement `BoardController` and `ColumnController` in `backend/src/boards/board.controller.ts` and `backend/src/columns/column.controller.ts`
- [ ] T105 [US2] Implement `TaskController` in `backend/src/tasks/task.controller.ts` with resource-scoped permission checks (assignee or manager can edit; only manager/admin or creator can delete)
- [ ] T106 [US2] Implement `CommentController` in `backend/src/comments/comment.controller.ts`
- [ ] T107 [US2] Wire WebSocket broadcasts: every successful task/comment/column mutation publishes `{eventType, payload}` to Redis channel `tenant:{tid}:project:{pid}` consumed by `RealtimeGateway` (depends on T036)
- [ ] T108 [US2] Implement `NotificationService` in `backend/src/notifications/notification.service.ts` creating notification rows + WebSocket `notification.created` events on task assignment and @-mention
- [ ] T109 [US2] Implement email notification job in `backend/src/notifications/jobs/send-email-notification.job.ts` (BullMQ, idempotency key, respects user notification preferences)
- [ ] T110 [P] [US2] Build Projects list page in `frontend/src/pages/Projects.tsx`
- [ ] T111 [P] [US2] Build Project detail / Board view in `frontend/src/pages/BoardView.tsx` with drag-and-drop (e.g., dnd-kit) and TanStack Query for tasks
- [ ] T112 [P] [US2] Implement WebSocket hook in `frontend/src/realtime/useRealtimeProject.ts` joining `tenant:{tid}:project:{pid}` room and invalidating relevant TanStack Query caches on events
- [ ] T113 [P] [US2] Build Task detail modal/panel with comments and activity feed in `frontend/src/components/tasks/TaskDetail.tsx`
- [ ] T114 [P] [US2] Build NotificationBell component in `frontend/src/components/notifications/NotificationBell.tsx` showing unread count via WebSocket
- [ ] T115 [US2] Add latency metrics for task move and board load endpoints (p50/p95/p99) tied to SC-004 SLO
- [ ] T116 [US2] Hook audit logging on project deletion/archive

**Checkpoint**: User Stories 1 AND 2 fully functional — a team can sign up, invite members, create projects, manage tasks on a board, and see real-time updates. **MVP COMPLETE**.

---

## Phase 5: User Story 3 - Time Tracking on Tasks (Priority: P2)

**Goal**: Members can start/stop timers and create manual time entries on tasks; managers can review and adjust entries within their projects.

**Independent Test**: Member starts timer on a task → stops it → entry appears with duration. Member also creates a manual entry. Manager edits one of the entries. Viewer is denied.

### Tests for User Story 3 (write FIRST, MUST FAIL before implementation) ⚠️

- [ ] T117 [P] [US3] Contract test for `POST /api/v1/tasks/{id}/timer/start` in `backend/tests/contract/timer-start.spec.ts`
- [ ] T118 [P] [US3] Contract test for `POST /api/v1/tasks/{id}/timer/stop` in `backend/tests/contract/timer-stop.spec.ts`
- [ ] T119 [P] [US3] Contract test for `GET /api/v1/timer/active` (current running timer for user) in `backend/tests/contract/timer-active.spec.ts`
- [ ] T120 [P] [US3] Contract test for `POST /api/v1/time-entries` (manual) in `backend/tests/contract/time-entries-create.spec.ts`
- [ ] T121 [P] [US3] Contract test for `GET /api/v1/time-entries` with filters (project_id, user_id, date range) in `backend/tests/contract/time-entries-list.spec.ts`
- [ ] T122 [P] [US3] Contract test for `PATCH /api/v1/time-entries/{id}`, `DELETE /api/v1/time-entries/{id}` in `backend/tests/contract/time-entries-update.spec.ts`
- [ ] T123 [P] [US3] Tenant-isolation integration test in `backend/tests/integration/us3-tenant-isolation.spec.ts`
- [ ] T124 [P] [US3] RBAC negative test in `backend/tests/integration/us3-rbac.spec.ts` (viewer cannot create entries; member cannot edit others' entries; manager can edit entries within owned projects only)
- [ ] T125 [P] [US3] Single-running-timer integration test in `backend/tests/integration/us3-single-timer.spec.ts` proving starting a new timer auto-stops the previous one
- [ ] T126 [P] [US3] Auto-stop on member removal test in `backend/tests/integration/us3-auto-stop-on-removal.spec.ts` (running timer is stopped and entry preserved)
- [ ] T127 [P] [US3] Soft-deleted task time entry retention test in `backend/tests/integration/us3-soft-deleted-task.spec.ts` (entries remain queryable referencing soft-deleted tasks)

### Implementation for User Story 3

- [ ] T128 [P] [US3] Implement `TimeEntry` Prisma model (id, tenant_id, user_membership_id, task_id, started_at, stopped_at, duration_seconds, source ENUM[timer,manual], notes, created_at, updated_at) with RLS policy and partial unique index `(user_membership_id) WHERE stopped_at IS NULL`
- [ ] T129 [US3] Run migration: `cd backend && npx prisma migrate dev --name us3_time_entries`
- [ ] T130 [P] [US3] Implement `TimerService` in `backend/src/time-tracking/timer.service.ts` (start auto-stops previous, stop, getActive)
- [ ] T131 [P] [US3] Implement `TimeEntryService` in `backend/src/time-tracking/time-entry.service.ts` (createManual, list with filters, update, delete, with manager scope checks)
- [ ] T132 [US3] Implement `TimerController` in `backend/src/time-tracking/timer.controller.ts`
- [ ] T133 [US3] Implement `TimeEntryController` in `backend/src/time-tracking/time-entry.controller.ts`
- [ ] T134 [US3] Hook into US1 session-revocation flow: on member removal, stop any running timer for that membership and preserve the partial entry (extend `session-revocation.service.ts`)
- [ ] T135 [US3] Broadcast timer state changes via WebSocket so other devices of the same user reflect the running timer
- [ ] T136 [P] [US3] Build Timer widget in `frontend/src/components/time/TimerWidget.tsx` (persistent header bar showing active timer with start/stop controls)
- [ ] T137 [P] [US3] Build Manual time entry form in `frontend/src/components/time/ManualEntryForm.tsx`
- [ ] T138 [P] [US3] Build Time entries list/table per project in `frontend/src/pages/ProjectTimeEntries.tsx` with manager edit/delete actions

**Checkpoint**: Stories 1, 2, AND 3 work independently — full project + task + time tracking stack.

---

## Phase 6: User Story 4 - Reporting Dashboard (Priority: P3)

**Goal**: Admins/managers see a filterable reporting dashboard (tasks completed, hours logged, active projects, overdue tasks) and can export CSVs.

**Independent Test**: Open dashboard with existing data → see metrics for current tenant only → filter by project + date range → export CSV → file downloads with filtered rows.

### Tests for User Story 4 (write FIRST, MUST FAIL before implementation) ⚠️

- [ ] T139 [P] [US4] Contract test for `GET /api/v1/reports/summary` (totals by tenant) in `backend/tests/contract/reports-summary.spec.ts`
- [ ] T140 [P] [US4] Contract test for `GET /api/v1/reports/tasks-completed` with filters in `backend/tests/contract/reports-tasks-completed.spec.ts`
- [ ] T141 [P] [US4] Contract test for `GET /api/v1/reports/hours-logged` with filters in `backend/tests/contract/reports-hours-logged.spec.ts`
- [ ] T142 [P] [US4] Contract test for `GET /api/v1/reports/overdue-tasks` in `backend/tests/contract/reports-overdue.spec.ts`
- [ ] T143 [P] [US4] Contract test for `POST /api/v1/reports/exports` (enqueue) and `GET /api/v1/reports/exports/{id}` (status + signed URL) in `backend/tests/contract/reports-exports.spec.ts`
- [ ] T144 [P] [US4] Tenant-isolation integration test in `backend/tests/integration/us4-tenant-isolation.spec.ts` proving report aggregations exclude other tenants
- [ ] T145 [P] [US4] RBAC test in `backend/tests/integration/us4-rbac.spec.ts`: admin/manager get full reports; member gets only personal metrics; viewer denied or personal-only per policy
- [ ] T146 [P] [US4] CSV export performance test in `backend/tests/integration/us4-export-performance.spec.ts` proving 50k time entries complete in <10s (SC-008)
- [ ] T147 [P] [US4] Dashboard latency test in `backend/tests/integration/us4-dashboard-latency.spec.ts` proving <3s p95 for 12 months of typical data (SC-007)

### Implementation for User Story 4

- [ ] T148 [P] [US4] Add database indexes optimizing report queries (composite indexes on `(tenant_id, project_id, completed_at)`, `(tenant_id, user_membership_id, started_at)`, `(tenant_id, due_date)` for overdue) in `backend/prisma/migrations/0004_report_indexes/migration.sql`
- [ ] T149 [P] [US4] Implement `ReportService` in `backend/src/reports/report.service.ts` with `summary`, `tasksCompleted`, `hoursLogged`, `overdueTasks` methods accepting filters
- [ ] T150 [P] [US4] Implement personal-metrics scope in `ReportService` for member role (auto-filter to actor's membership)
- [ ] T151 [US4] Implement `ReportController` in `backend/src/reports/report.controller.ts` with role-based filter enforcement
- [ ] T152 [P] [US4] Implement `Export` Prisma model (id, tenant_id, requested_by_membership_id, type, filters_json, status ENUM[pending,running,completed,failed], file_url, error, created_at, completed_at) with RLS
- [ ] T153 [US4] Run migration: `cd backend && npx prisma migrate dev --name us4_exports`
- [ ] T154 [P] [US4] Implement `ExportService` in `backend/src/reports/export.service.ts` enqueueing BullMQ `export-report` jobs and returning job IDs
- [ ] T155 [US4] Implement `ExportProcessor` worker in `backend/src/reports/jobs/export.processor.ts` that streams CSV to S3 and writes signed URL (15min TTL) on completion
- [ ] T156 [US4] Implement `ExportController` (`POST /reports/exports`, `GET /reports/exports/{id}`) in `backend/src/reports/export.controller.ts`
- [ ] T157 [US4] Emit WebSocket `export.completed` event when job finishes so frontend can offer download
- [ ] T158 [US4] Hook audit logging on every export request and download
- [ ] T159 [P] [US4] Build Dashboard page in `frontend/src/pages/Dashboard.tsx` with summary cards (tasks completed, hours logged, active projects, overdue) and charts (e.g., recharts)
- [ ] T160 [P] [US4] Build filter controls (project, member, date range) in `frontend/src/components/reports/ReportFilters.tsx`
- [ ] T161 [P] [US4] Build Export button + status modal in `frontend/src/components/reports/ExportButton.tsx` polling job status and offering download link
- [ ] T162 [P] [US4] Build personal metrics view for members in `frontend/src/pages/MyMetrics.tsx`

**Checkpoint**: All four user stories independently functional. Full v1 product complete.

---

## Phase 7: Polish & Cross-Cutting Concerns

**Purpose**: Hardening, performance, documentation, and final validation

- [ ] T163 [P] Implement tenant data export (workspace dump) in `backend/src/tenants/export/tenant-export.service.ts` producing JSON+CSV bundle via background job (FR-026)
- [ ] T164 [P] Implement right-to-erasure flow in `backend/src/users/erasure/erasure.service.ts` anonymizing PII while preserving foreign keys (FR-026)
- [ ] T165 [P] Implement notification preferences UI and API in `frontend/src/pages/settings/NotificationPreferences.tsx` and `backend/src/users/preferences.controller.ts`
- [ ] T166 [P] Add per-tenant rate limiting alerts (Prometheus alert when 429s spike for one tenant) in `infra/alerts/tenant-throttle.yml`
- [ ] T167 [P] Add database backup configuration (daily automated, tested restore procedure) documented in `docs/operations/backups.md`
- [ ] T168 [P] Run dependency vulnerability scan (npm audit, Snyk) in CI; document SLA remediation in `docs/security/vulnerability-management.md`
- [ ] T169 [P] Write quickstart validation script in `specs/001-build-me-multi/quickstart.md` covering all four user-story acceptance flows end-to-end
- [ ] T170 [P] Run Playwright end-to-end tests for all four user stories in `e2e/tests/`
- [ ] T171 [P] Run load test scenario (k6 or Artillery) hitting SC-004, SC-007, SC-008, SC-009 thresholds in `e2e/load/scenarios.js`
- [ ] T172 [P] Performance optimization pass: review p95 latency dashboards, add caching where measured beneficial (Redis cache for hot project metadata)
- [ ] T173 [P] Security hardening: helmet config review, CSP headers, CSRF protection on cookie-bearing endpoints in `backend/src/main.ts`
- [ ] T174 [P] Update `README.md` at repo root with architecture overview, dev setup, and deployment links
- [ ] T175 [P] Update `CLAUDE.md` with conventions discovered during implementation
- [ ] T176 [P] Document API in `docs/api/README.md` referencing `contracts/openapi.yaml` and publishing Swagger UI at `/api/docs`
- [ ] T177 Final constitutional compliance review: verify every endpoint has tenant-isolation test + RBAC test + audit log + structured logs (per Principles I, II, III, V)
- [ ] T178 Run quickstart.md validation against staging environment before production cutover

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies — start immediately
- **Foundational (Phase 2)**: Depends on Setup completion — **BLOCKS all user stories**
- **User Story 1 (Phase 3)**: Depends on Phase 2 — independently testable
- **User Story 2 (Phase 4)**: Depends on Phase 2 — independently testable (uses US1 auth/membership but does not modify it)
- **User Story 3 (Phase 5)**: Depends on Phase 2; integrates with US2 tasks (foreign key) but is independently testable
- **User Story 4 (Phase 6)**: Depends on Phase 2; reports aggregate data from US2 and US3 but the dashboard endpoints and export pipeline are independently testable with seeded data
- **Polish (Phase 7)**: Depends on all desired user stories being complete

### User Story Dependencies

- **US1 (P1)**: No dependencies on other stories — pure foundation slice
- **US2 (P1)**: Depends on US1 (uses memberships for assignees, reporters, comment authors) — but can be developed in parallel after Phase 2 by mocking US1 data fixtures, then integrating
- **US3 (P2)**: Depends on US2 tasks (`task_id` foreign key) — develop after US2 entities exist, or mock with seeded tasks
- **US4 (P3)**: Depends on US2 + US3 data shapes — develop after both, or mock with seeded data

### Within Each User Story

- Tests MUST be written and FAIL before implementation (Constitution Principle III, NON-NEGOTIABLE)
- Prisma models → migration → services → controllers → realtime/jobs → frontend
- Audit logging hooks added during service implementation
- Story complete when all acceptance scenarios pass + tenant-isolation + RBAC negative tests pass

### Parallel Opportunities

- All Setup tasks marked [P] can run in parallel (T004–T012)
- Many Foundational tasks marked [P] can run in parallel (T018–T019, T022–T023, T026–T032, T035–T037, T038–T042)
- Once Phase 2 completes, **all four user stories can be worked on in parallel** by different developers
- Within each story: all contract tests [P] in parallel; all Prisma models [P] in parallel; frontend pages [P] in parallel

---

## Parallel Example: User Story 1 Kickoff