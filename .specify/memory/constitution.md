<!--
Sync Impact Report
- Version change: 1.2.0 → 1.3.0
- Modified principles: VIII (Language & Naming Conventions) — added French as a
  supported locale (es/en/fr) and the hybrid catalog translation strategy
- Added sections: none
- Removed sections: none

- Templates checked for alignment:
  - .specify/templates/spec-template.md ✅ no changes required (principle-agnostic)
  - .specify/templates/plan-template.md ✅ no changes required (reads constitution at runtime)
  - .specify/templates/tasks-template.md ✅ no changes required
  - .specify/templates/checklist-template.md ✅ no changes required
- Follow-up TODOs: none outstanding

Previous entries:
- v1.2.0 (2026-06-05): added Principle VIII — English for all code, database and UI
  identifiers; i18n (es/en) from the start.
- v1.1.0 (2026-06-05): resolved TODO(FRONTEND_FRAMEWORK) to Angular; added
  hosting/deployment constraint (Azure).
- v1.0.0 (2026-06-05): initial ratification — Core Principles (I-VII), Technology &
  Architecture Constraints, Development Workflow & Quality Gates, Governance.
-->

# AEspejo ERP SaaS Constitution

## Core Principles

### I. Multi-Tenant Isolation First (NON-NEGOTIABLE)

Every table, query, cache entry, background job, and log line that carries business data
MUST be scoped by `TenantId`. The system uses a shared-database, shared-schema
multi-tenancy model: one SQL Server database serves all tenants, and isolation is
enforced in the application layer, not by physical separation. Consequently:

- Every EF Core entity representing tenant-owned data MUST include `TenantId` and MUST be
  covered by an EF Core Global Query Filter; filters MUST NOT be bypassable except through
  an explicitly audited administrative code path.
- No query, report, export, or integration may cross tenant boundaries unless it is an
  explicitly designated cross-tenant admin operation, which MUST be logged and
  authorized separately from normal tenant operations.
- Every new feature's plan MUST include a "Tenant Isolation" review step before
  `/speckit-plan` is considered complete.

**Rationale**: A single shared schema is the cheapest model to operate, but it is also the
easiest to get wrong. Because there is no database-level backstop, isolation bugs mean
one customer's financial or clinical data leaking to another — an unacceptable failure
mode for an ERP handling accounting and health records. Treating isolation as a
first-class, testable requirement (not an implementation detail) is the only way to keep
this model safe as the number of modules grows.

### II. Contract-First APIs

Before any controller/endpoint is implemented, its contract (OpenAPI spec or equivalent
schema: routes, request/response DTOs, status codes, error shapes) MUST be written and
reviewed. Breaking changes to a published contract require a version bump and a
migration/deprecation note.

**Rationale**: The backend (.NET) and frontend (SPA) are developed and versioned
somewhat independently, and multiple business modules will eventually consume shared
platform APIs (auth, tenants, billing). Contracts-first avoids integration drift and
gives `/speckit-plan` a concrete artifact to design against.

### III. Test-First Development (NON-NEGOTIABLE)

TDD is mandatory for domain logic and API endpoints: tests are written, reviewed, and
observed to fail before implementation begins (Red-Green-Refactor). Required layers:

- Unit tests for domain/business rules (per module).
- Integration tests for API endpoints and EF Core queries, including at least one test
  per endpoint that asserts cross-tenant data is NOT returned.
- Contract tests for any endpoint consumed by another module or the frontend.

**Rationale**: ERP and clinical-record features encode business/legal rules (tax
calculation, accounting balance, medical record retention) where silent regressions are
costly. Test-first is the cheapest guardrail against that class of bug.

### IV. Modular Vertical Architecture

The system is organized as a stable **Platform Core** (tenants, identity/auth, plans &
subscriptions, billing) plus independent **Business Modules** (Facturación, Inventario,
Contabilidad, CRM/Ventas, Gestión de Clínica, Clínica Dental, and future verticals).
Each business module:

- MUST depend only on the Platform Core and explicitly declared shared kernels (e.g., a
  common Facturación/Inventario base reused by clinic verticals); it MUST NOT depend
  directly on the internals of another business module.
- MUST be independently enable/disable-able per tenant (module activation is a
  Platform Core concern, driven by the tenant's subscribed plan).
- MUST ship as its own spec under `specs/NNN-<module>/` with its own user stories,
  requirements, and success criteria — modules are not bundled into a single mega-spec.

**Rationale**: The product vision spans multiple verticals (general ERP + clinic/dental
practice management) that share a platform but have distinct domain rules. Enforcing
module boundaries now prevents the codebase from calcifying into a monolith that can't
add or remove verticals per tenant later.

### V. Security & Compliance by Design

- Authentication and authorization (RBAC per tenant, per module) MUST be enforced at the
  API layer for every endpoint; there is no "trusted frontend."
  Secrets/connection strings MUST NOT be committed to the repository.
- Every write to financial (Contabilidad, Facturación) or clinical (Clínica, Clínica
  Dental) data MUST produce an immutable audit trail entry (who, tenant, when, before/after).
- Data classified as health information (clinic modules) MUST be identified explicitly in
  each relevant spec's Key Entities section so retention/compliance requirements can be
  reviewed before implementation.

**Rationale**: Combining financial and health-record verticals in one SaaS raises the
compliance bar beyond a typical business app; treating audit and access control as
constitutional rather than per-feature avoids gaps introduced by module-by-module teams.

### VI. Observability & Operability

Every module MUST emit structured logs (including `TenantId` and correlation/request ID)
for state-changing operations, expose health checks, and surface key domain events
(e.g., invoice issued, appointment booked) in a way that can be wired to metrics/alerts.
Errors surfaced to users MUST be distinguishable from errors that require operator
attention.

**Rationale**: A multi-tenant SaaS fails silently at the tenant level unless logs and
metrics are tenant-aware from day one; retrofitting observability after multiple modules
exist is significantly more expensive.

### VII. Simplicity & Incremental Delivery

Each feature spec MUST decompose into independently valuable, independently testable
user stories (P1/P2/P3 …) so a module can ship an MVP slice rather than waiting for full
scope. Prefer the simplest design that satisfies current requirements (YAGNI);
architectural complexity (new services, new datastores, cross-module coupling) MUST be
justified in the plan's Complexity Tracking section or avoided.

**Rationale**: With seven-plus planned modules, uncontrolled complexity growth is the
main delivery risk. Incremental, story-sliced delivery keeps every module shippable and
demoable early.

### VIII. Language & Naming Conventions (NON-NEGOTIABLE)

All technical artifacts MUST be written in **English**, without exception:

- **Code**: class, method, function, variable, parameter and file names; also comments and
  XML documentation. No mixed-language identifiers (`getFactura`, `ClienteService`).
- **Database**: table names, columns, indexes, constraints, stored procedures and
  migration names.
- **API contracts**: routes, DTO properties, query parameters and error codes.
- **UI**: component names, control identifiers, enum members and translation keys.

User-facing text MUST NOT be hardcoded in any language. It MUST go through i18n
resources, with **Spanish (es), English (en) and French (fr)** supported from the start;
`es` is the default locale. This applies to UI labels and to any API message intended for
display. Every screen MUST be reachable in all supported languages, and the user MUST be
able to switch language from the UI at any time.

**Translating stored data** — three categories, each with one correct mechanism:

| Category | Example | Mechanism |
|---|---|---|
| System catalog (fixed, ships with the product) | Modules, statuses, document types | i18n **key** stored in the row (e.g. `NameKey`), text lives in resource files |
| Tenant-editable catalog (rows created at runtime) | Product categories, payment terms, dental treatment types | **Translation table** per entity (`<Entity>Translation` with `LanguageCode` + translated columns), falling back to the tenant's default language |
| Captured data | Customer name, invoice description | **Never translated** — displayed exactly as entered |

A tenant-editable catalog MUST NOT use i18n keys: the tenant cannot add entries to our
resource files. Conversely, a system catalog MUST NOT use translation tables, or every
deployment would need data migrations to ship its own labels.

**Rationale**: Getting this wrong is expensive in both directions — hardcoding text blocks
selling to another market, and putting tenant-owned catalogs in resource files makes them
untranslatable by the only people who can translate them. Fixing the split now avoids
schema migrations across every module later.

**Rationale**: Mixed-language codebases force a permanent mental translation tax and produce
inconsistencies that compound across modules (`Invoice` vs `Factura` for the same concept).
English is the lingua franca of the frameworks in use (.NET, Angular, EF Core), so English
identifiers read continuously with the framework APIs surrounding them. Separating
identifiers from displayed text via i18n means selling to a non-Spanish-speaking market
later is a translation-file task, not a refactor. Because renaming entities, tables and
routes gets exponentially more expensive with each module added, this is fixed now rather
than negotiated per module.

## Technology & Architecture Constraints

- **Backend**: .NET (C#, ASP.NET Core Web API), Entity Framework Core as the ORM against
  SQL Server.
- **Frontend**: Angular single-page application (TypeScript, strict mode). Decided
  2026-06-05; rationale recorded in `specs/001-fundacion-multitenant/research.md`. All
  business modules MUST use the same frontend framework and shared UI shell — a module
  MUST NOT introduce a second frontend framework. Angular i18n (or `@ngx-translate`)
  MUST be wired from the Foundation so no module ships hardcoded user-facing strings,
  and the UI MUST expose a language selector (Principle VIII).
- **Hosting**: Azure — App Service for the API and SPA, Azure SQL Database for storage,
  Azure Key Vault for secrets. Application code MUST NOT hardcode environment-specific
  configuration; it MUST come from configuration/secret providers.
- **Database**: SQL Server, shared database/shared schema across tenants (see Principle
  I). All tenant-owned tables carry `TenantId`. Cross-cutting reference data (currencies,
  tax tables, module catalog) is not tenant-scoped and MUST be flagged as such in specs.
- **Identity**: A single Platform Core identity system issues sessions/tokens used by all
  modules; modules MUST NOT implement their own authentication.
- **Module activation**: Which business modules a tenant has access to is driven by the
  tenant's subscription plan, owned by the Platform Core.

## Development Workflow & Quality Gates

This project follows the Spec Kit workflow strictly, in this order, per feature:

1. `/speckit-specify` — write the feature spec (user stories, requirements, success
   criteria). No implementation detail.
2. `/speckit-clarify` — resolve `[NEEDS CLARIFICATION]` markers before planning.
3. `/speckit-plan` — produce the technical plan; MUST explicitly address Tenant Isolation
   (Principle I) and Complexity Tracking (Principle VII).
4. `/speckit-tasks` — generate the task breakdown.
5. `/speckit-analyze` (recommended for Platform Core and financial/clinical modules) —
   cross-artifact consistency check before implementation starts.
6. `/speckit-implement` — execute, following Test-First (Principle III).

Quality gates before a feature is considered done:

- All functional requirements in the spec are covered by at least one test.
- No cross-tenant data leakage test is missing for any new endpoint/query.
- Code review confirms compliance with this constitution; deviations MUST be recorded
  and justified (e.g., in the plan's Complexity Tracking section), not silently merged.

## Governance

This constitution supersedes ad-hoc practice for this repository. Amendments are made by
editing `.specify/memory/constitution.md` via `/speckit-constitution`, MUST update the
Sync Impact Report at the top of the file, and MUST follow semantic versioning:

- **MAJOR**: Backward-incompatible principle removal or redefinition (e.g., abandoning
  shared-schema multi-tenancy).
- **MINOR**: New principle or materially expanded guidance (e.g., adding a new module
  category's constraints).
- **PATCH**: Wording clarifications, typo fixes, non-semantic edits.

All specs, plans, and code reviews MUST verify compliance with this constitution.
Complexity or deviation from a principle MUST be explicitly justified in the relevant
plan; unjustified deviations block merge.

**Version**: 1.3.0 | **Ratified**: 2026-06-05 | **Last Amended**: 2026-06-05
