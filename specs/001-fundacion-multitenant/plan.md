# Implementation Plan: Multi-Tenant SaaS Foundation

**Branch**: `001-fundacion-multitenant` | **Date**: 2026-06-05 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/001-fundacion-multitenant/spec.md`

## Summary

Build the **Platform Core** of the ERP SaaS: tenant onboarding and lifecycle,
authentication, per-tenant RBAC, subscription plans, and module activation — with
multi-tenant isolation guaranteed by design.

Technical approach: modular monolith in .NET 10 on Azure, with shared SQL Server
(shared-schema) where isolation is enforced **by convention, not by discipline**:
all tenant-owned entities are subject to an EF Core Global Query Filter applied
automatically, and `SaveChanges` assigns and validates the `TenantId` on write. Authorization
is resolved in the pipeline (role + plan module + tenant state) so modules 002-007 inherit
it without reimplementing. Angular frontend with lazy loading per module, aligned with
per-tenant module activation.

## Technical Context

**Language/Version**: C# / .NET 10 (LTS) — backend; TypeScript 5.x / Angular (latest LTS) — frontend

**Primary Dependencies**: ASP.NET Core Web API, Entity Framework Core 10, ASP.NET Core
Identity, Angular CLI

**Storage**: Azure SQL Database (SQL Server) — shared database, shared schema, with
`TenantId` in each tenant-owned table

**Testing**: xUnit + FluentAssertions + Testcontainers (real SQL Server in integration);
Jasmine/Karma on the SPA

**Target Platform**: Azure App Service (API + SPA), Azure SQL Database, Azure Key Vault
for secrets

**Project Type**: Web application (backend + frontend), modular monolith

**Performance Goals**: p95 < 300 ms on Foundation reads; plan change reflected
in < 1 s (SC-004)

**Constraints**: No physical isolation between tenants → logical isolation is a
security requirement, not a quality one; append-only audit with 7-year retention; secrets
exclusively in Key Vault / user-secrets, never in repository

**Scale/Scope**: Initial design for ~500 tenants × ~50 concurrent users per tenant;
7 business modules planned (see `ROADMAP.md`)

> Detail and discarded alternatives for each decision: [research.md](./research.md) (D-001 … D-007).

## Constitution Check

*GATE: must pass before Phase 0 and re-evaluate after Phase 1.*

| Principle | How This Plan Fulfills It | Pre-Phase 0 | Post-Phase 1 |
|---|---|---|---|
| **I. Multi-Tenant Isolation First** | Global Query Filter by convention on `ITenantOwned`; `TenantId` resolved only from claim (D-002); assignment and validation in `SaveChanges`; `Plan`, `Module`, and `PlatformUser` explicitly declared as global exceptions in `data-model.md`; only cross-tenant access (platform operator) isolated behind `PlatformScope` policy and audited (FR-017/FR-018); cross-tenant leak test mandatory per endpoint (SC-002) | ✅ | ✅ |
| **II. Contract-First APIs** | `contracts/platform-api.openapi.yaml` written before implementation; typed error codes; Angular client generated from contract | ✅ | ✅ |
| **III. Test-First** | Order enforced in `tasks.md`: contract → integration (includes isolation) → unit → implementation | ✅ | ✅ |
| **IV. Modular Vertical Architecture** | `Platform.*` references no business module; modules 002-007 depend on `Platform.Core`/`SharedKernel` and never each other; module catalog in data, not code (D-004/D-006) | ✅ | ✅ |
| **V. Security & Compliance by Design** | Authorization in pipeline for all endpoints; append-only audit via interceptor (D-005); passwords managed by Identity; audit without secrets; no clinical data in this feature | ✅ | ✅ |
| **VI. Observability & Operability** | Structured logging with `TenantId` + correlation id; health checks; domain events (tenant created, plan changed) emitted for metrics | ✅ | ✅ |
| **VII. Simplicity & Incremental Delivery** | Modular monolith instead of microservices (D-006); 4 user stories deliverable separately, P1 = demonstrable MVP; no quantitative plan limits in v1 | ✅ | ✅ |

**Result**: no violations. The *Complexity Tracking* section is intentionally left empty.

**Traceability Note**: the constitution was amended to **v1.1.0** during this phase to
resolve `TODO(FRONTEND_FRAMEWORK)` → Angular and set Azure as deployment target.

## Project Structure

### Documentation (this feature)

```text
specs/001-fundacion-multitenant/
├── plan.md              # This file
├── spec.md              # Specification (clarified)
├── research.md          # Phase 0: D-001 … D-007
├── data-model.md        # Phase 1: entities, invariants, relationships
├── quickstart.md        # Phase 1: end-to-end validation guide
├── contracts/           # Phase 1: OpenAPI contract + error conventions
│   ├── README.md
│   └── platform-api.openapi.yaml
└── tasks.md             # Phase 2 (/speckit-tasks — not yet created)
```

### Source Code (repository root)

```text
backend/
├── src/
│   ├── Platform.Api/              # Endpoints, tenant/module middleware, DI, config
│   ├── Platform.Core/             # Domain: Tenant, User, Role, Plan, Module; invariants
│   ├── Platform.Infrastructure/   # DbContext, query filters, interceptors, migrations, Identity
│   └── SharedKernel/              # Shared abstractions: ITenantOwned, ITenantContext, errors
└── tests/
    ├── Platform.Contract.Tests/     # Verify OpenAPI (Principle II)
    ├── Platform.Integration.Tests/  # API + real DB; includes cross-tenant isolation suite
    └── Platform.Unit.Tests/         # Domain rules (last admin, state transitions)

frontend/
├── src/app/
│   ├── core/          # HTTP interceptors, auth, role and module guards
│   ├── shell/         # Layout, navigation filtered by enabled modules
│   └── features/      # auth/, tenants/, users/, roles/ (lazy loading)
└── src/tests/
```

**Structure Decision**: modular monolith with separation by projects (D-006). Future
business modules will be added as `backend/src/Modules/<Module>.{Api,Core,Infrastructure}`
and `frontend/src/app/features/<module>/`, referencing `Platform.Core`/`SharedKernel` but
never each other — Principle IV boundary remains verifiable at compile time.

## Complexity Tracking

> No entries: Constitution Check passed without violations to justify.
