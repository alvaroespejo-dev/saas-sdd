# Data Model — Multi-Tenant SaaS Foundation (001)

**Date**: 2026-06-05 | **Spec**: [spec.md](./spec.md) | **Research**: [research.md](./research.md)

Logical model of the Foundation. Conventions applied to all entities:

- **All identifiers (entities, fields, tables, indexes) are in English**
  (Principle VIII of the constitution). The explanatory prose in this document continues in
  English; technical names are translated.
- Primary key `Guid` (sequential, to avoid index fragmentation in SQL Server).
- `CreatedAt` / `CreatedBy` / `UpdatedAt` / `UpdatedBy` in mutable entities.
- **Tenant-owned** = implements `ITenantOwned` → carries `TenantId` and is subject to Global
  Query Filter (D-002). **Global** = platform table, not filtered by tenant.

---

## Entities

### PlatformUser *(global)*

SaaS operator (FR-017). **Does not** implement `ITenantOwned` and therefore **does not** carry
`TenantId` nor is subject to Global Query Filter — it is one of the cross-tenant exceptions
that Principle I obliges to declare explicitly.

| Campo | Tipo | Reglas |
|---|---|---|
| `Id` | Guid | PK |
| `Email` | string(256) | **Único global** (a diferencia de `User`, único sólo por tenant) |
| `PasswordHash` | string | Gestionado por ASP.NET Core Identity |
| `FullName` | string(200) | Requerido |
| `Status` | enum | `Active` \| `Disabled` |

- Authenticates via a different endpoint from tenants, **without** `tenantSlug` (FR-017).
- Their sessions emit a token with platform claim and **without** `tenant_id`; that token
  MUST be rejected by any tenant-owned data endpoint (FR-018).
- All actions generate `AuditLogEntry` with `ActorUserId = null` and
  `ActorPlatformUserId` informed.

---

### Tenant *(global)*

Client organization (company or clinic).

| Campo | Tipo | Reglas |
|---|---|---|
| `Id` | Guid | PK |
| `Name` | string(200) | Requerido |
| `Slug` | string(63) | **Único global**, minúsculas, `[a-z0-9-]`, usado como subdominio (FR-002) |
| `Status` | enum | `Active` \| `Suspended` (FR-014) |
| `PlanId` | Guid | FK → Plan (plan vigente) |
| `DefaultLanguage` | string(5) | `es` \| `en` \| `fr`; idioma corporativo, usado cuando el usuario no ha elegido uno y como respaldo de traducciones de catálogo (FR-021, FR-022) |
| `CreatedAt` | datetime2 | Requerido |

- Unique index on `Slug` — uniqueness is guaranteed in database, not just in
  application, to close the race condition described in Edge Cases.
- State transitions: `Active ⇄ Suspended`. Both generate audit events (FR-013).

---

### User *(tenant-owned)*

Person who accesses the system. Belongs to **exactly one** Tenant (Clarifications 2026-06-05).

| Campo | Tipo | Reglas |
|---|---|---|
| `Id` | Guid | PK |
| `TenantId` | Guid | FK → Tenant, requerido |
| `Email` | string(256) | **Único dentro del tenant** (no globalmente: el mismo correo puede existir en dos tenants como cuentas independientes) |
| `PasswordHash` | string | Gestionado por ASP.NET Core Identity (D-003) |
| `FullName` | string(200) | Requerido |
| `Status` | enum | `Active` \| `Disabled` |
| `PreferredLanguage` | string(5)? | `es` \| `en` \| `fr`; `null` ⇒ hereda el idioma por defecto del tenant (FR-021) |

- Composite unique index `(TenantId, Email)`.
- **Invariant (FR-008)**: cannot deactivate a user nor revoke their Administrator role
  if they are the last active `Active` Administrator of their tenant. Validated in the
  domain service within the same transaction.
- When deactivating the user or changing their roles → revoke their refresh tokens (FR-016).

---

### Role *(tenant-owned)*

| Campo | Tipo | Reglas |
|---|---|---|
| `Id` | Guid | PK |
| `TenantId` | Guid | FK → Tenant |
| `Name` | string(100) | Único dentro del tenant |
| `IsSystem` | bool | `true` para roles predefinidos (p. ej. Administrador); no editables ni eliminables |

- Each tenant receives an `Administrator` role (`IsSystem = true`) when onboarded.
- **UserRole**: junction table `(UserId, RoleId)`, tenant-owned. A user can have
  multiple roles within their tenant (FR-006).
- **RolePermission**: junction table `(RoleId, PermissionCode)`. Permissions are
  granular codes (e.g., `invoicing.invoice.create`) declared by each module; Role is the
  grouping entity configurable per tenant.

---

### Plan *(global)*

| Campo | Tipo | Reglas |
|---|---|---|
| `Id` | Guid | PK |
| `Name` | string(100) | Requerido, único |
| `IsActive` | bool | Un plan inactivo no puede asignarse a tenants nuevos |

- **PlanModule**: junction table `(PlanId, ModuleId)` — defines which modules the plan includes (FR-009).
- No quantitative limits in this version (Clarifications 2026-06-05). Adding them later
  means adding columns to `Plan` without altering these relationships.

---

### Module *(global)*

Business module catalog. Fed by data, not code (D-004), so that specs 002-007 can
be incorporated without deploying the Foundation.

| Campo | Tipo | Reglas |
|---|---|---|
| `Id` | Guid | PK |
| `Code` | string(50) | Único; p. ej. `invoicing`, `inventory`, `accounting`, `crm`, `clinic`, `dental-clinic` |
| `Name` | string(150) | Etiqueta para UI |
| `IsActive` | bool | Permite retirar un módulo del catálogo comercial |

---

### TenantModule *(tenant-owned)*

Effective state of a module for a tenant. Derived from the plan, but **materialized**
to support manual deactivation and check state without recalculating.

| Campo | Tipo | Reglas |
|---|---|---|
| `TenantId` | Guid | PK compuesta |
| `ModuleId` | Guid | PK compuesta |
| `IsEnabled` | bool | `false` ⇒ reads allowed on existing data, writes blocked (FR-012a) |
| `ChangedAt` | datetime2 | Last transition |

- When changing a tenant's plan (FR-012), rows are recalculated and **entitlements cache
  is invalidated** for that tenant (SC-004: reflected in < 1 s).
- Disabling **never** deletes module data (FR-012a).

---

### RefreshToken *(tenant-owned)*

| Campo | Tipo | Reglas |
|---|---|---|
| `Id` | Guid | PK |
| `TenantId` | Guid | FK → Tenant |
| `UserId` | Guid | FK → User |
| `TokenHash` | string | Stored **hashed**, never in plaintext |
| `ExpiresAt` | datetime2 | Required |
| `RevokedAt` | datetime2? | `null` while valid |

- Revoked in bulk when user is deactivated or roles modified (FR-016).

---

### AuditLogEntry *(tenant-owned, append-only)*

| Campo | Tipo | Reglas |
|---|---|---|
| `Id` | Guid | PK |
| `TenantId` | Guid | Requerido |
| `ActorUserId` | Guid? | Usuario de tenant que ejecutó la acción; `null` si fue un operador de plataforma |
| `ActorPlatformUserId` | Guid? | FK → PlatformUser; informed only in cross-tenant actions (FR-018). Exactly one of the two actors MUST be informed |
| `Action` | string(100) | e.g., `tenant.created`, `plan.changed`, `user.role.assigned` |
| `EntityType` / `EntityId` | string / Guid? | Target of the action |
| `OldValues` / `NewValues` | json | Only relevant fields, **no secrets or password hashes** |
| `OccurredAt` | datetime2 | UTC |

- Generated by `SaveChanges` interceptor (D-005). No UPDATE/DELETE operations
  exposed; 7-year retention (D-007).
- Minimum required events (FR-013): tenant creation, plan change, user role creation/removal/change,
  tenant suspension/reactivation.

---

## Data Translation Convention (FR-022, Principle VIII)

Three categories of stored text, each with **only one** correct mechanism:

| Category | Example in This Model | Mechanism |
|---|---|---|
| System catalog (fixed) | `Module.NameKey`, `Role.Name` of system roles | i18n key in the row; text lives in frontend resources |
| Tenant-editable catalog | Product categories, payment methods, treatment types (modules 002-007) | **Translation table** per entity |
| Captured data | `Tenant.Name`, `User.FullName`, invoice notes | **Never translated**: displayed as written |

**Mandatory pattern for tenant catalogs** — every editable catalog entity
`<Entity>` carries a child table `<Entity>Translation`:

| Field | Type | Rules |
|---|---|---|
| `TenantId` | Guid | Composite PK; the table is tenant-owned |
| `<Entity>Id` | Guid | Composite PK, FK → parent entity |
| `LanguageCode` | string(5) | Composite PK; `es` \| `en` \| `fr` |
| `Name` (and other translatable fields) | string | Text in that language |

Resolution on read: user's active language → `Tenant.DefaultLanguage` → first
available translation. **Never** returns empty due to missing translation (FR-022).

> A tenant-editable catalog **cannot** use i18n keys: the tenant cannot
> add entries to our resource files. Conversely, a system catalog
> **doesn't** use a translation table, or each deployment would need data migrations to
> bring its own labels.

This spec (the Foundation) doesn't yet contain any tenant-editable catalog; the
pattern is fixed here so modules 002-007 apply it without reinventing it.

## Relationship Diagram

```text
PlatformUser (global)          Plan ──< PlanModule >── Module
 │  administra ▼                     │                        │
 └───────────────────────────►  (plan vigente)                │
                                     ▼                        ▼
                                Tenant ──────────────< TenantModule >
                                 │
                                 ├──< User ──< UserRole >── Role ──< RolePermission
                                 ├──< RefreshToken
                                 └──< AuditLogEntry
```

Everything hanging from `Tenant` is tenant-owned and is automatically filtered by the
Global Query Filter.

**Declared global exceptions** (Principle I requires marking them explicitly):
`Plan` and `Module` are shared commercial catalogs, and `PlatformUser` is the
SaaS operator identity. None carry `TenantId`. `PlatformUser` is also the
only authorized path for cross-tenant access, and that's why all its activity is audited.
