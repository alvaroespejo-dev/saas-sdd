# Quickstart — Validation of the Multi-Tenant SaaS Foundation (001)

Guide to set up the Foundation and **verify end-to-end** that it fulfills what the
spec promises. It does not contain implementation code: that belongs to `tasks.md` and
the `/speckit-implement` phase.

## Prerequisites

- .NET SDK 10 (LTS)
- Node.js 22+ and Angular CLI
- SQL Server (local, Docker, or LocalDB) accessible
- Connection string in user configuration, **never** in the repository:
  `dotnet user-secrets set "ConnectionStrings:Default" "<...>"`

## Getting Started

```bash
# 1. Database: apply migrations and seed data
#    (seed: roadmap module catalog + example plans)
dotnet ef database update --project backend/src/Platform.Infrastructure

# 2. API
dotnet run --project backend/src/Platform.Api        # https://localhost:5001

# 3. SPA
cd frontend && npm install && npm start              # http://localhost:4200
```

## Validation Scenarios

Each scenario maps to a User Story in the spec. They are considered passed when the
corresponding automated test passes **and** the manual flow matches.

### V1 — Tenant Onboarding (User Story 1)

1. Authenticate as platform operator: `POST /api/v1/auth/platform/login` (without
   `tenantSlug`) → platform token (FR-017).
2. `POST /api/v1/platform/tenants` with a free `slug`.
3. **Expected**: `201`; tenant is in `Active` state with its Administrator user.
4. Repeat the same call with identical `slug` → **`409 SLUG_ALREADY_IN_USE`** (FR-002).
5. Repeat the call with a **tenant user** token → **`403 FORBIDDEN_SCOPE`** (FR-018).
6. `GET /api/v1/modules` with that admin's session → returns exactly the modules of
   the contracted plan, no more (User Story 1, scenario 3).

> **Measure SC-001**: time steps 1–3 plus the tenant administrator's first login;
> total must be under 5 minutes.

### V2 — Cross-Tenant Isolation (User Story 2) — *the critical scenario*

1. Create **two** tenants (A and B), each with a user.
2. Authenticate as user A and note the `id` of a user B.
3. `GET /api/v1/users/{idOfUserB}` with token A.
4. **Expected**: **`404 NOT_FOUND`** — never `403` nor B's data (FR-005).
5. Try sending a foreign `tenantId` in the body of a `POST /api/v1/users`.
6. **Expected**: field is ignored; record is created in the session's tenant (FR-004).

> This scenario verifies Principle I. If it fails, nothing else can be considered good.

### V3 — RBAC (User Story 3)

1. Create a "Read-only" role with no write permissions and assign it to a user.
2. With that user: `POST /api/v1/users` → **`403 FORBIDDEN_ROLE`**.
3. With the Administrator: same call → `201`.
4. Try removing the Administrator role from the **only** active admin → **`409 LAST_ADMIN`** (FR-008).
5. Revoke a role from a user with an open session: their refresh stops working immediately
   and the access token stops working when it expires (≤15 min) (FR-016).

### V4 — Plans, Modules, and Suspension (User Story 4)

1. `PUT /api/v1/platform/tenants/{id}/plan` changing to a plan **without** Accounting.
2. `GET /api/v1/modules` → Accounting appears with `enabled: false` in less than 1 s (SC-004).
3. Call an Accounting **write** endpoint → **`403 MODULE_NOT_IN_PLAN`**,
   distinguishable from `FORBIDDEN_ROLE` (FR-011).
4. **Read** Accounting data created before the change → **remains accessible**; read
   is never blocked by a disabled module (FR-012a, decision table in
   `contracts/README.md`).
5. `PUT /platform/tenants/{id}/status` to `Suspended`: reads OK, any write
   → **`403 TENANT_SUSPENDED`** (FR-014).

### V5 — Auditing (FR-013 / SC-005)

After running V1–V4, check the audit table: there should be records of tenant creation,
plan change, role assignment, and suspension, each with actor, tenant, and
timestamp. Also verify that they **do not** contain password hashes or secrets.

## Automated Suite

```bash
dotnet test                    # unit + integration + contract
cd frontend && npm test        # SPA unit tests
```

**Feature exit criterion**: 100% of tenant-owned data endpoints have at least one
cross-tenant leak test (SC-002); without that, the feature is not considered
complete even if the rest passes.
