---

description: "Task list — Multi-Tenant SaaS Foundation (001)"
---

# Tasks: Multi-Tenant SaaS Foundation

**Input**: Design documents from `/specs/001-fundacion-multitenant/`

**Prerequisites**: [plan.md](./plan.md), [spec.md](./spec.md), [research.md](./research.md),
[data-model.md](./data-model.md), [contracts/](./contracts/)

**Tests**: **MANDATORY**. Principle III of the constitution (Test-First) is non-negotiable,
so test tasks are written and must **fail** before corresponding implementation.

**Organization**: Tasks grouped by user story to allow independent implementation and
validation.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can be executed in parallel (separate files, no pending dependencies)
- **[Story]**: User story it belongs to (US1…US4)
- Each task includes its exact file path

## Path Conventions

**Language (Principle VIII)**: all code, database, and UI identifiers go in
**English** (classes, methods, files, tables, fields, enums, and comments). Visible text
is resolved via i18n with resources `es` (default) and `en` (FR-019). This task list and
specs continue in English.

Web application structure per `plan.md`:

- Backend: `backend/src/{Platform.Api,Platform.Core,Platform.Infrastructure,SharedKernel}/`
- Tests: `backend/tests/{Platform.Contract.Tests,Platform.Integration.Tests,Platform.Unit.Tests}/`
- Frontend: `frontend/src/app/{core,shell,features}/`

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Solution initialization and foundational tooling

- [x] T001 Crear solución .NET 10 y estructura de proyectos (`Platform.Api`, `Platform.Core`, `Platform.Infrastructure`, `SharedKernel`) en `backend/` con las referencias del monolito modular (D-006)
- [x] T002 Crear proyectos de test (`Platform.Contract.Tests`, `Platform.Integration.Tests`, `Platform.Unit.Tests`) en `backend/tests/` con xUnit + FluentAssertions
- [x] T003 [P] Inicializar proyecto Angular con TypeScript en modo estricto en `frontend/`
- [x] T004 [P] Configurar analizadores y formato en `backend/.editorconfig` y `Directory.Build.props` (nullable enabled, warnings as errors)
- [x] T005 [P] Configurar ESLint + Prettier en `frontend/eslint.config.js` y `frontend/.prettierrc.json` (el archivo generado del contrato queda excluido de ambos)
- [x] T006 [P] Configurar `dotnet user-secrets` y lectura de configuración/Key Vault en `backend/src/Platform.Api/Program.cs` (ninguna cadena de conexión en el repositorio)
- [x] T007 [P] Configurar SQL Server real para tests de integración con cadena configurable (LocalDB en local, `TEST_SQL_CONNECTION` con Testcontainers en CI) en `backend/tests/Platform.Integration.Tests/Fixtures/SqlServerFixture.cs` — *ajustado el 2026-06-05: el daemon de Docker no estaba disponible, se acordó LocalDB en local + Testcontainers en CI*
- [x] T008 [P] Configurar pipeline CI (build + test backend y frontend) en `.github/workflows/ci.yml`

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Multi-tenant isolation, auth, and auditing infrastructure that ALL stories depend on

**⚠️ CRITICAL**: No user story can begin until this phase is complete. Principle I
is materialized here; if this fails, everything else is insecure.

> **Dependency correction (detected during implementation, 2026-06-05)**: T021 (initial
> migration) requires entities to exist, which were assigned to phases 3-6
> (T028-T030, T041, T053, T066). Model entities are therefore created in this phase,
> along with their EF configurations. Phase 3-6 tasks mentioning them are
> reduced to adding domain behavior, not creating the class.

- [x] T009 [P] Definir `ITenantOwned` e `ITenantContext` en `backend/src/SharedKernel/Multitenancy/`
- [x] T010 [P] Definir tipos de error tipificados (`UNAUTHENTICATED`, `FORBIDDEN_ROLE`, `FORBIDDEN_SCOPE`, `MODULE_NOT_IN_PLAN`, `TENANT_SUSPENDED`, `NOT_FOUND`, `SLUG_ALREADY_IN_USE`, `LAST_ADMIN`) en `backend/src/SharedKernel/Errors/PlatformError.cs` según la tabla de decisión de `contracts/README.md`
- [x] T011 Crear `PlatformDbContext` en `backend/src/Platform.Infrastructure/Persistence/PlatformDbContext.cs`
- [x] T012 **Test-first**: escribir test que verifique que TODA entidad `ITenantOwned` registrada en el modelo tiene Global Query Filter aplicado, en `backend/tests/Platform.Unit.Tests/Multitenancy/QueryFilterConventionTests.cs` (debe fallar antes de T013)
- [x] T013 Aplicar Global Query Filter **por convención** a todas las entidades `ITenantOwned` en `PlatformDbContext.OnModelCreating` (FR-005, D-002)
- [x] T014 **Test-first**: escribir test que verifique que `SaveChanges` asigna el `TenantId` del contexto y **rechaza** entidades con `TenantId` ajeno, en `backend/tests/Platform.Unit.Tests/Multitenancy/TenantAssignmentTests.cs`
- [x] T015 Implementar interceptor de asignación/validación de `TenantId` en `SaveChanges` en `backend/src/Platform.Infrastructure/Interceptors/TenantAssignmentInterceptor.cs` (cierra el flanco de escritura, D-002)
- [x] T016 Implementar resolución de `TenantId` **exclusivamente desde el claim** del token en `backend/src/Platform.Api/Middleware/TenantContextMiddleware.cs` (FR-004; ignora/rechaza cualquier tenantId del cliente)
- [x] T017 Configurar hasheo de contraseñas (PasswordHasher de ASP.NET Core Identity) y emisión/validación de JWT (access ~15 min) en `backend/src/Platform.Infrastructure/Identity/` (D-003) — *refinamiento registrado: se reutiliza sólo el hasher de Identity, no su user store, porque éste exige nombre de usuario único global y nuestro modelo permite el mismo correo en tenants distintos*
- [x] T017a [P] Crear entidad global `PlatformUser` (sin `TenantId`, email único global) en `backend/src/Platform.Core/Platform/PlatformUser.cs` (FR-017)
- [x] T017b **Test-first**: escribir test que verifique que un token de tenant en un endpoint de plataforma devuelve `403 FORBIDDEN_SCOPE` y que un token de plataforma es rechazado en endpoints tenant-owned, en `backend/tests/Platform.Integration.Tests/Foundational_IdentityScopeTests.cs` (FR-018)
- [x] T017c Implementar `POST /auth/platform/login` (sin `tenantSlug`) y emisión de token de plataforma sin claim `tenant_id` en `backend/src/Platform.Api/Controllers/PlatformAuthController.cs` (FR-017)
- [x] T017d Implementar política de autorización `PlatformScope` y su verificación en el pipeline en `backend/src/Platform.Api/Authorization/PlatformScopePolicy.cs` (FR-018)
- [x] T018a [P] Crear entidad `AuditLogEntry` (append-only, con `ActorUserId` y `ActorPlatformUserId`) en `backend/src/Platform.Core/Auditing/AuditLogEntry.cs` (FR-013)
- [x] T018 [P] Implementar interceptor de auditoría append-only en `backend/src/Platform.Infrastructure/Interceptors/AuditInterceptor.cs`, cubriendo explícitamente `Tenant`, `User`, `UserRole` y cambios de plan/estado (D-005, FR-013; excluye secretos y hashes de contraseña)
- [x] T019 [P] Configurar logging estructurado con `TenantId` + correlation id y health checks en `backend/src/Platform.Api/Observability/` (Principio VI)
- [x] T020 [P] Implementar middleware de manejo de errores que serialice el formato `{code, message, traceId}` en `backend/src/Platform.Api/Middleware/ErrorHandlingMiddleware.cs`
- [x] T021 Crear migración inicial de EF Core con todas las entidades de `data-model.md` en `backend/src/Platform.Infrastructure/Migrations/`
- [x] T022 [P] Crear datos semilla (catálogo de módulos del `ROADMAP.md` + planes de ejemplo) en `backend/src/Platform.Infrastructure/Persistence/SeedData.cs`
- [x] T023 [P] Configurar cliente HTTP Angular generado desde `contracts/platform-api.openapi.yaml` en `frontend/src/app/core/api/` (Principio II)
- [x] T023a [P] Configurar i18n en Angular con recursos `es` (por defecto), `en` y `fr` en `frontend/public/i18n/` y el servicio de idioma en `frontend/src/app/core/i18n/language.service.ts` (FR-019, Principio VIII)
- [x] T023b [P] Crear el catálogo de claves de traducción para los códigos de error de `contracts/README.md` (uno por `code`) en `frontend/public/i18n/{es,en}.json` (FR-019)
- [x] T023c [P] Test que verifique que todo `PlatformErrorCode` tiene clave de traducción en los tres idiomas y que la estructura de claves coincide entre archivos, en `frontend/src/app/core/i18n/error-messages.spec.ts` (FR-019)
- [x] T023d [P] Implementar el selector de idioma accesible desde cualquier pantalla en `frontend/src/app/shell/language-selector/language-selector.ts` y montarlo en el shell (`frontend/src/app/app.html`) (FR-020)
- [x] T023e [P] Definir `SupportedLanguages` (es/en/fr) y la precedencia usuario → tenant → sistema en `backend/src/SharedKernel/Localization/SupportedLanguages.cs`, con test en `backend/tests/Platform.Unit.Tests/Localization/LanguageResolutionTests.cs` (FR-021)
- [x] T023f [P] Añadir `User.PreferredLanguage` y `Tenant.DefaultLanguage` con su migración en `backend/src/Platform.Infrastructure/Persistence/Migrations/` (FR-021)
- [x] T023g [P] Definir el patrón de traducción de catálogos del tenant (`IEntityTranslation`, `ITranslatable<T>`, resolución con respaldo) en `backend/src/SharedKernel/Localization/ITranslatable.cs` para que lo reutilicen los módulos 002-007 (FR-022)
- [x] T023h Implementar `PUT /users/me/language` en `backend/src/Platform.Api/Controllers/UsersController.cs` (FR-021) — el contrato y el cliente Angular ya lo contemplan

**Checkpoint**: Isolation, auth, auditing, and operational errors → user stories can begin

---

## Phase 3: User Story 1 — Tenant Onboarding (Priority: P1) 🎯 MVP

**Goal**: A platform operator onboards a tenant with its plan and first Administrator user.

**Independent Test**: `POST /platform/tenants` creates tenant in `Active` state with its admin; repeating the `slug` returns `409 SLUG_ALREADY_IN_USE`; tenant modules match plan modules exactly.

### Tests for User Story 1 ⚠️ (write first, must FAIL)

- [x] T024 [P] [US1] Contract test for `POST /platform/tenants` and `POST /auth/platform/login` against OpenAPI in `backend/tests/Platform.Contract.Tests/TenantsContractTests.cs` (FR-001, FR-017)
- [x] T025 [P] [US1] Integration test for tenant onboarding (201 + admin created + Active state) in `backend/tests/Platform.Integration.Tests/US1_TenantOnboardingTests.cs`
- [x] T026 [P] [US1] Integration test for duplicate `slug` → `409 SLUG_ALREADY_IN_USE` in `backend/tests/Platform.Integration.Tests/US1_UniqueSlugTests.cs`
- [x] T027 [P] [US1] Unit test for `slug` uniqueness rule at index level (race condition) in `backend/tests/Platform.Unit.Tests/Tenants/SlugUniquenessTests.cs`

### Implementation for User Story 1

- [x] T028 [P] [US1] Create `Tenant` entity (with `Status`, `Slug`, `PlanId`) in `backend/src/Platform.Core/Tenants/Tenant.cs`
- [x] T029 [P] [US1] Create `Plan`, `Module`, and `PlanModule` entities (global catalogs, no `TenantId`) in `backend/src/Platform.Core/Plans/`
- [x] T030 [P] [US1] Create `User` entity in `backend/src/Platform.Core/Users/User.cs`
- [x] T031 [US1] Set up unique index on `Tenant.Slug` (FR-002, closes race condition) and composite unique `(TenantId, Email)` in `backend/src/Platform.Infrastructure/Persistence/Configurations/` (depends on T028, T030)
- [x] T032 [US1] Implement `TenantProvisioningService` (creates tenant + Administrator role + admin user + `TenantModule` rows in one transaction) in `backend/src/Platform.Core/Tenants/TenantProvisioningService.cs` (FR-001, FR-009)
- [x] T033 [US1] Implement `POST /api/v1/platform/tenants` endpoint protected by `PlatformScope` policy (T017d) in `backend/src/Platform.Api/Controllers/PlatformTenantsController.cs` (FR-001, FR-018)
- [x] T034 [US1] Emit `tenant.created` audit events with `ActorPlatformUserId` (FR-013, FR-018) from provisioning service
- [x] T035 [P] [US1] Tenant onboarding screen (platform operator) in `frontend/src/app/features/tenants/tenant-onboarding.ts`

**Checkpoint**: US1 functional and independently demonstrable (MVP)

---

## Phase 4: User Story 2 — Login and Data Isolation (Priority: P1)

**Goal**: A user authenticates and accesses **only** their tenant's data.

**Independent Test**: With two seeded tenants, a user from A requests a resource from B and receives `404` — never the data nor a `403` revealing existence.

> This is the critical security story (Principle I). Its tests are the exit criterion for
> the complete feature (SC-002).

### Tests for User Story 2 ⚠️ (write first, must FAIL)

- [x] T036 [P] [US2] Contract test for `POST /auth/login`, `/auth/refresh`, `/auth/me` in `backend/tests/Platform.Contract.Tests/AuthContractTests.cs`
- [x] T037 [P] [US2] **Cross-tenant leak test**: user from A requests resource from B → `404 NOT_FOUND`, in `backend/tests/Platform.Integration.Tests/US2_CrossTenantIsolationTests.cs`
- [x] T038 [P] [US2] Integration test: `tenantId` sent in body is ignored; record created in session's tenant (FR-004) in `backend/tests/Platform.Integration.Tests/US2_UntrustedTenantIdTests.cs`
- [x] T039 [P] [US2] Integration test: invalid credentials and expired token → `401` without revealing which part failed, in `backend/tests/Platform.Integration.Tests/US2_FailedLoginTests.cs`
- [x] T040 [P] [US2] Integration test for `POST /auth/forgot-password` → always `202`, whether account exists (FR-015), in `backend/tests/Platform.Integration.Tests/US2_ForgotPasswordTests.cs`

### Implementation for User Story 2

- [x] T041 [P] [US2] Create `RefreshToken` entity (stored **hashed**) in `backend/src/Platform.Core/Auth/RefreshToken.cs`
- [x] T042 [US2] Implement `AuthService` (login, access+refresh issuance with `tenant_id` claim, rotation and revocation) in `backend/src/Platform.Core/Auth/AuthService.cs` (FR-003)
- [x] T043 [US2] Implement `POST /auth/login`, `/auth/refresh`, `/auth/logout` endpoints in `backend/src/Platform.Api/Controllers/AuthController.cs`
- [x] T044 [US2] Implement `GET /auth/me` returning tenant, roles, and enabled modules in `backend/src/Platform.Api/Controllers/AuthController.cs`
- [x] T045 [US2] Implement `POST /auth/forgot-password` with anti-enumeration uniform response (FR-015) in `backend/src/Platform.Api/Controllers/AuthController.cs`
- [x] T046 [P] [US2] Angular HTTP interceptor that attaches access token and renews with refresh in `frontend/src/app/core/auth/auth.interceptor.ts`
- [x] T047 [P] [US2] Login screen and authenticated route guard in `frontend/src/app/features/auth/login.ts` and `frontend/src/app/core/auth/auth.guard.ts`, with home screen listing enabled modules

**Checkpoint**: US1 + US2 work independently; isolation is tested

---

## Phase 5: User Story 3 — RBAC Within Tenant (Priority: P2)

**Goal**: Tenant Administrator manages users and roles; actions restricted by role.

**Independent Test**: A "Read-only" user receives `403 FORBIDDEN_ROLE` on a write that Administrator executes with `201`.

### Tests for User Story 3 ⚠️ (write first, must FAIL)

- [x] T048 [P] [US3] Contract test for `/users` and `/roles` in `backend/tests/Platform.Contract.Tests/UsersRolesContractTests.cs`
- [x] T049 [P] [US3] Integration test **parameterized per defined role** (SC-003): each role without required permission → `403 FORBIDDEN_ROLE`; Administrator → `201`, in `backend/tests/Platform.Integration.Tests/US3_RbacTests.cs` (FR-006)
- [x] T050 [P] [US3] Integration test: Admin of A cannot manage users of B (FR-007) in `backend/tests/Platform.Integration.Tests/US3_NoCrossTenantAuthorityTests.cs`
- [x] T051 [P] [US3] Unit test for "last Administrator" invariant → `409 LAST_ADMIN` (FR-008) in `backend/tests/Platform.Unit.Tests/Users/LastAdministratorTests.cs`
- [x] T052 [P] [US3] Integration test: when role is revoked, refresh tokens revoked and access lost in ≤15 min (FR-016) in `backend/tests/Platform.Integration.Tests/US3_SessionRevocationTests.cs`

### Implementation for User Story 3

- [x] T053 [P] [US3] Create `Role`, `UserRole`, and `RolePermission` entities in `backend/src/Platform.Core/Roles/`
- [x] T054 [US3] Implement `UserService` with last Administrator invariant (same transaction) in `backend/src/Platform.Core/Users/UserService.cs`
- [x] T055 [US3] Implement `RoleService` (create role, assign permissions; `IsSystem` roles not editable) in `backend/src/Platform.Core/Roles/RoleService.cs`
- [x] T056 [US3] Implement CRUD endpoints for `/api/v1/users` in `backend/src/Platform.Api/Controllers/UsersController.cs`
- [x] T057 [US3] Implement `PUT /users/{id}/roles` with refresh token revocation (FR-016) in `backend/src/Platform.Api/Controllers/UsersController.cs`
- [x] T058 [US3] Implement `/api/v1/roles` endpoints in `backend/src/Platform.Api/Controllers/RolesController.cs`
- [x] T059 [US3] Implement permission-based authorization policy in `backend/src/Platform.Api/Authorization/PermissionRequirement.cs`
- [x] T060 [P] [US3] User and role management screens in `frontend/src/app/features/users/` and `frontend/src/app/features/roles/`

**Checkpoint**: US1 + US2 + US3 operationally independent

---

## Phase 6: User Story 4 — Plans, Modules, and Suspension (Priority: P2)

**Goal**: Changing a tenant's plan enables/disables modules; suspended tenants become read-only.

**Independent Test**: Changing to a plan without Accounting makes `GET /modules` reflect it in <1 s and writes return `403 MODULE_NOT_IN_PLAN`, while previous data remains readable.

### Tests for User Story 4 ⚠️ (write first, must FAIL)

- [x] T061 [P] [US4] Contract test for `/platform/tenants/{id}/plan`, `/platform/tenants/{id}/status`, and `/modules` in `backend/tests/Platform.Contract.Tests/PlansModulesContractTests.cs`
- [x] T062 [P] [US4] Integration test: module outside plan → `403 MODULE_NOT_IN_PLAN` (distinguishable from `FORBIDDEN_ROLE`, FR-011) in `backend/tests/Platform.Integration.Tests/US4_EntitlementsTests.cs`
- [x] T063 [P] [US4] Integration test: disabled module allows **read** of previous data and rejects write with `403 MODULE_NOT_IN_PLAN`, per decision table in `contracts/README.md` (FR-012, FR-012a) in `backend/tests/Platform.Integration.Tests/US4_ModuleReadOnlyTests.cs`
- [x] T064 [P] [US4] Integration test: `Suspended` tenant reads but doesn't write → `403 TENANT_SUSPENDED` (FR-014) in `backend/tests/Platform.Integration.Tests/US4_SuspendedTenantTests.cs`
- [x] T065 [P] [US4] Integration test: plan change reflected in `/modules` in <1 s, no stale cache (SC-004) in `backend/tests/Platform.Integration.Tests/US4_CacheEntitlementsTests.cs`

### Implementation for User Story 4

- [x] T066 [P] [US4] Create `TenantModule` entity in `backend/src/Platform.Core/Modules/TenantModule.cs`
- [x] T067 [US4] Implement `EntitlementService` with per-tenant cache and invalidation on plan change (D-004) in `backend/src/Platform.Core/Modules/EntitlementService.cs`
- [x] T068 [US4] Implement `[RequiresModule("code")]` attribute and filter that **only blocks writes** on disabled modules (`403 MODULE_NOT_IN_PLAN`), allowing reads, in `backend/src/Platform.Api/Authorization/RequiresModuleAttribute.cs` (FR-011, FR-012a)
- [x] T069 [US4] Implement tenant state verification (`TENANT_SUSPENDED` on writes) in same pipeline, in `backend/src/Platform.Api/Authorization/TenantStateFilter.cs`
- [x] T070 [US4] Implement `PUT /platform/tenants/{id}/plan` with `TenantModule` recalculation in `backend/src/Platform.Api/Controllers/PlatformTenantsController.cs`
- [x] T071 [US4] Implement `PUT /platform/tenants/{id}/status` (suspend/reactivate) with auditing in `backend/src/Platform.Api/Controllers/PlatformTenantsController.cs`
- [x] T072 [US4] Implement `GET /api/v1/modules` exposing tenant's enabled modules in session in `backend/src/Platform.Api/Controllers/ModulesController.cs` (FR-010)
- [x] T073 [P] [US4] Module guard and navigation filtered by enabled modules (lazy loading) in `frontend/src/app/core/guards/module.guard.ts` and `frontend/src/app/shell/`

**Checkpoint**: All 4 user stories work independently

---

## Phase 7: Polish & Cross-Cutting Concerns

**Purpose**: Quality, security, and exit criteria closure

- [x] T074 **Isolation coverage audit (SC-002)**: automatic sweep over route table in `backend/tests/Platform.Integration.Tests/IsolationCoverageTests.cs` — *implemented as reflection over controllers instead of a hand-maintained list, because a hand list becomes stale the moment a module adds an endpoint*
- [x] T075 [P] Integration test verifying 5 required audit events from FR-013 (SC-005) in `backend/tests/Platform.Integration.Tests/AuditLogTests.cs`
- [x] T076 [P] Verify audit records don't contain secrets or password hashes, in `backend/tests/Platform.Unit.Tests/Auditing/NoSecretsLoggedTests.cs`
- [ ] T077 [P] Add indexes and pagination per p95 < 300 ms target (D-007) in `backend/src/Platform.Infrastructure/Persistence/Configurations/` and record load measurement in `backend/tests/Platform.Integration.Tests/Performance/`
- [x] T078 [P] Configurable rate limiting (`RateLimiting:CredentialAttemptsPerMinute`, default 30/min per IP) on `/auth/login`, `/auth/platform/login`, and `/auth/forgot-password`, with dedicated test in `backend/tests/Platform.Integration.Tests/RateLimitingTests.cs` — *limit is configurable because partitioning by IP means an office behind NAT shares quota; a value designed for one user would block the entire company*. Pending: HTTP security headers and password policy
- [x] T079 [P] Verify no secrets in repository and configuration comes from Key Vault/user-secrets
- [ ] T080 Run complete validation of [quickstart.md](./quickstart.md) (scenarios V1–V5), **timing tenant onboarding + first login to verify SC-001 (<5 min)** and recording result in `specs/001-fundacion-multitenant/quickstart.md`
- [ ] T081 Update `ROADMAP.md` (001 → ✅ Implemented) and leave module catalog ready for spec 002

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: no dependencies
- **Foundational (Phase 2)**: depends on Phase 1 — **BLOCKS all user stories**
- **User Stories (Phases 3-6)**: all depend on Phase 2
- **Polish (Phase 7)**: depends on which stories you want to close

### User Story Dependencies

- **US1 (P1)**: no dependencies on other stories after Phase 2
- **US2 (P1)**: independent in implementation; its tests need existing tenants,
  provided by seed data (T022) or by US1
- **US3 (P2)**: independent; its tests use seeded users
- **US4 (P2)**: independent; uses module catalog seeded in T022

> No story depends on **code** from another: they only share Phase 2 and seed
> data, which is what keeps them separately testable (Principle VII).

### Within Each User Story

- Tests are written and **must fail** before implementing (Principle III)
- Entities → services → endpoints → UI

### Parallel Opportunities

- T003–T008 (setup) in parallel
- T009, T010 in parallel; T018, T019, T020, T022, T023 in parallel after T011
- All tests marked `[P]` within a story, in parallel
- With sufficient team, US1–US4 in parallel after Phase 2

---

## Parallel Example: User Story 2

```bash
# US2 tests in parallel (write first, must fail):
Task: "Contract test for auth in backend/tests/Platform.Contract.Tests/AuthContractTests.cs"
Task: "Cross-tenant leak test in backend/tests/Platform.Integration.Tests/US2_CrossTenantIsolationTests.cs"
Task: "Untrusted tenantId test in backend/tests/Platform.Integration.Tests/US2_UntrustedTenantIdTests.cs"
Task: "Failed login test in backend/tests/Platform.Integration.Tests/US2_FailedLoginTests.cs"
Task: "Forgot-password test in backend/tests/Platform.Integration.Tests/US2_ForgotPasswordTests.cs"
```

---

## Implementation Strategy

### MVP (US1 + US2)

1. Phase 1: Setup
2. Phase 2: Foundational — **critical**, isolation lives here
3. Phase 3 (US1) + Phase 4 (US2)
4. **STOP AND VALIDATE**: scenarios V1 and V2 from `quickstart.md`
5. Possible demo: tenant onboarding + login with proven isolation

> The MVP includes **two** stories because both are P1: a SaaS with tenant onboarding but
> unverified isolation is not demonstrable with real data.

### Incremental Delivery

1. Setup + Foundational → base ready
2. + US1 → tenant onboarding → demo
3. + US2 → isolated login → **deployable MVP**
4. + US3 → teams and permissions per tenant
5. + US4 → plans and module activation → **unblocks specs 002-007**

---

## Notes

- `[P]` = separate files, no pending dependencies
- Verify each test fails before implementing
- Commit per task or logical group
- T074 is non-negotiable exit criterion: without isolation coverage, the feature is
  not considered complete even if the rest passes (SC-002)
