# Phase 1 Data Model: Multi-Tenant SaaS Project Management Tool

**Feature**: 001-build-me-multi
**Date**: 2026-04-28

> All entities below carry a `tenant_id` (UUID) column **except** `User`, which is global (a single email may belong to multiple tenants via `Membership`). Every tenant-scoped table has a Postgres Row-Level Security policy:
>
> 