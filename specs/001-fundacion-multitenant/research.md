# Research — Multi-Tenant SaaS Foundation (001)

**Date**: 2026-06-05 | **Spec**: [spec.md](./spec.md) | **Plan**: [plan.md](./plan.md)

Record of technical decisions (ADRs) from Phase 0. Each decision resolves a
`NEEDS CLARIFICATION` from the Technical Context or a structural choice that conditions
all roadmap modules.

---

## D-001: Frontend Framework — Angular

**Decision**: Angular (latest LTS), TypeScript in strict mode, single SPA that
will host all business modules via lazy loading per module.

**Rationale**:
- "Batteries-included" framework (routing, reactive forms, HTTP, DI, i18n): with 7
  planned modules, it reduces library decision-making and the risk of each
  module diverging in its stack.
- Reactive forms and typed validation fit an ERP LOB app, which is
  essentially forms and grids with dense business rules.
- Angular's lazy loading per module maps 1:1 with module activation per tenant
  (Principle IV): a module not included in the plan simply doesn't load.

**Considered Alternatives**:
- *React*: larger ecosystem and easier hiring, but requires choosing and maintaining
  routing/state/forms/data-fetching separately — more decision surface and
  divergence across 7 modules.
- *Blazor*: would reuse C# and backend validations, but with smaller ecosystem of
  enterprise components (grids, reports) and fewer available profiles.

---

## D-002: Multi-tenant Isolation Strategy — Global Query Filters + Server-side TenantId Resolution

**Decision**:
- Every tenant-owned entity implements an `ITenantOwned { Guid TenantId }` interface.
- The `DbContext` applies an **EF Core Global Query Filter** by convention to **all**
  entities implementing `ITenantOwned`, filtering by the `TenantId` from the current
  request context.
- The `TenantId` is resolved **exclusively** from the authenticated user's claim
  (server-side); never from body, query string, route, or headers sent by the client.
- `SaveChanges` automatically assigns the `TenantId` to new entities and **rejects**
  (throws) any attempt to persist an entity with a `TenantId` different from the
  current context.
- Query filter bypass (`IgnoreQueryFilters`) is only allowed in an explicit,
  audited administrative service; forbidden elsewhere.

**Rationale**: Fulfills Principle I (non-negotiable). In a shared-schema model there is
no safety net at the database level, so the filter must be **default and by convention**,
not something every developer remembers to add in each query. Automatic assignment in
`SaveChanges` closes the write flank, which query filters don't cover.

**Considered Alternatives**:
- *Manually filter in each repository/query*: a single oversight causes data leakage
  between tenants; unacceptable given the planned module volume.
- *SQL Server Row-Level Security (RLS)*: real defense-in-depth at the engine level,
  but complicates migrations and local debugging, and requires propagating session context
  to each connection (problematic with pooling). **Deferred as future reinforcement**, not
  as primary mechanism.

---

## D-003: Authentication — ASP.NET Core Identity + Short-lived JWT with Revocable Refresh Token

**Decision**:
- ASP.NET Core Identity for credential storage (framework PBKDF2/bcrypt hashing,
  password policies, lockout).
- **Access token** short-lived JWT (~15 min) with claims `sub`, `tenant_id`, `roles`.
- **Refresh token** opaque, persisted in database and individually revocable.
- When revoking a role or disabling a user (FR-016), **invalidate their refresh tokens** and
  the change takes effect at most when the current access token expires.

**Rationale**: FR-016 requires that a role change takes effect without waiting for the next login.
A pure, long-lived JWT is not revocable; a 100% server-side session would complicate horizontal
scaling in App Service. The combination short access + revocable refresh is the standard pattern
that satisfies both, with a bounded and documented exposure window.

**Consequence Recorded in Spec**: the "immediate effect" of FR-016 is implemented as
"at most when the current access token expires (≤15 min)"; complete session closure
is immediate because the refresh token is revoked instantly.

**Considered Alternatives**:
- *Long-lived JWT without refresh*: simple, but impossible to revoke → violates FR-016.
- *Server-side cookie sessions (distributed cache)*: true immediate revocation, but
  adds Redis/distributed store dependency and complicates consumption from non-web clients
  (future integrations).

---

## D-004: Module Entitlements — Server-Resolved and Verified by Middleware

**Decision**:
- The `Module` catalog and `Plan → Modules` relationship live in the database (not in code),
  so roadmap modules can be added without deploying.
- An authorization middleware/filter validates in **every** request to an endpoint marked
  with its module (e.g., attribute `[RequiresModule("Invoicing")]`) that the
  tenant has it enabled; if not, responds **403 with an explicit error code**
  (`MODULE_NOT_IN_PLAN`) distinguishable from a 403 for role (FR-011).
- The tenant state (`Active`/`Suspended`) is verified at the same point: a suspended
  tenant receives `TENANT_SUSPENDED` in any write operation (FR-014).
- A disabled module allows **reads** of existing data and blocks writes
  (FR-012a), so verification distinguishes between read and write operations.
- Entitlements are cached in-memory per tenant with invalidation on plan change,
  to fulfill SC-004 (<1 s) without querying the database on each request.

**Rationale**: Centralizing control in the pipeline prevents each business module from
reimplementing (and forgetting) verification. Having the catalog in data allows specs
002-007 to be incorporated without touching the Foundation.

**Considered Alternatives**:
- *Verify entitlements inside each controller*: repetitive and easy to omit.
- *Enum of modules in code*: would require deploying the Foundation every time a
  roadmap module is added.

---

## D-005: Auditing — EF Core Interceptor on SaveChanges

**Decision**: A `SaveChangesInterceptor` automatically generates audit records
(actor, tenant, entity, action, timestamp, before/after values) for entities
marked as auditable. The audit table is **append-only**: no update or delete endpoints,
and database application permissions do not include `UPDATE`/`DELETE` on it.

**Rationale**: Principle V requires immutable auditing for financial and clinical data. A
centralized interceptor guarantees uniform coverage for modules 002-007 without each
implementing its own logging.

**Considered Alternatives**:
- *Log auditing manually in each service*: incomplete coverage guaranteed.
- *SQL Server Temporal Tables*: excellent for change history, but doesn't capture **who**
  made the change or the business context. **Deferred** as complement, not replacement.

---

## D-006: Solution Structure — Modular Monolith

**Decision**: Single deployment (modular monolith) with .NET projects separated by
module boundary: `Platform.Core` (tenants, identity, plans, auditing) plus one project
per business module (specs 002-007). Modules reference `Platform.Core`, never each other;
inter-module communication, when needed, will be via contracts published in a
`SharedKernel` project.

**Rationale**: Principles IV and VII. Module boundaries are enforced by project references
(verifiable at compile time) without paying the operational cost of microservices,
which would be unjustified for current volume and team size. If a module needed to
scale separately in the future, its boundaries would already be defined.

**Considered Alternatives**:
- *Microservices per module*: operational complexity (deployments, distributed consistency,
  observability) disproportionate to current benefit.
- *Single project without separation*: doesn't enforce Principle IV boundaries.

---

## D-007: Performance and Scale Targets (Deferred in `/speckit-clarify`)

**Decision** (starting values, revisable with real production data):
- Target latency p95 **< 300 ms** for Foundation read endpoints.
- Initial design scale: **up to ~500 tenants** and **~50 concurrent users per tenant**,
  with single shared database.
- Audit log retention: **7 years** for financial and clinical events, aligned with
  typical accounting/tax retention periods.

**Rationale**: The spec did not set numeric targets (deferred to plan). Concrete
values are needed to size indexes, cache, and pagination strategy. The 7-year retention
is chosen conservatively given that modules 004 (Accounting) and 006/007 (clinical)
will have legal retention requirements.

**Review Point**: if scale exceeds ~500 tenants or an individual tenant grows
disproportionately, reevaluate shared-schema against a hybrid model (large tenants
in dedicated database) — this would be a MAJOR amendment to the constitution.
