# Feature Specification: Multi-Tenant SaaS Foundation

**Feature Branch**: `001-fundacion-multitenant`

**Created**: 2026-06-05

**Status**: Draft

**Input**: User description: "Multi-tenant SaaS foundation: tenant management, RBAC authentication and authorization, plans and subscriptions, module activation per tenant"

## Clarifications

### Session 2026-06-05

- Q: Can the same user belong to multiple tenants, or does each user belong to a single tenant? → A: A user belongs to a single tenant (simple model, 1:N Tenant→User).
- Q: When a tenant is `Suspended`, do its users retain read-only access to existing data, or is access completely blocked? → A: Read-only — they can view/export existing data, but no writes in any module.
- Q: When a business module is deactivated for a tenant that already has data created in that module, what happens to that data? → A: It remains visible in read-only; only creation/editing of new module records is blocked.
- Q: Do subscription plans have quantitative limits (users, storage, etc.) in this first version, or only determine which modules are enabled? → A: Only module catalog for now; quantitative limits are out of scope for this version.
- Q: How does the platform operator who onboards tenants authenticate? (detected in `/speckit-analyze`, finding C1) → A: With its own platform identity, global and separate from tenant users, logging in without specifying a tenant.

- Q: What language are the code, database, and UI written in? → A: Everything in English (identifiers, file names, tables, fields, enums, and comments), with i18n for visible text. Ratified as Principle VIII of the constitution.
- Q: What languages should the application support and how are database catalogs translated? → A: Spanish (default), English, and French, with selector in the UI. Hybrid strategy: i18n key for system catalogs, translation table for catalogs that the tenant edits, and captured data is never translated.
- Q: Where is the language chosen by each person stored? → A: In their user account (DB), with the browser as fallback while not logged in.

**Glossary**: "the Foundation" and "Platform Core" designate the same thing — the cross-cutting core described by this spec. Technical artifacts (`plan.md`, code) use *Platform Core*.

**Canonical Names**: this spec describes the business in Spanish, but technical artifacts use English names (Principle VIII): Tenant, User, Role, UserRole, RolePermission, Plan, PlanModule, Module, TenantModule, RefreshToken, PlatformUser, AuditLogEntry.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Onboarding a New Tenant (Client Organization) (Priority: P1)

A platform administrator (or a self-registration flow) creates a new tenant
(client company/clinic) with a subscription plan, ready for its first
administrator user to log in and start configuring the organization.

**Why this priority**: Without the ability to create isolated tenants there is no SaaS: it is the
absolute prerequisite for everything else (Principle I of the constitution).

**Independent Test**: Can be tested by creating a tenant via API/UI onboarding and
verifying it persists with its associated plan, without needing any
business module (Invoicing, Inventory, etc.) to exist yet.

**Acceptance Scenarios**:

1. **Given** that no tenant exists with the requested identifier/subdomain,
   **When** onboarding is completed with organization name, plan, and initial
   administrator user data, **Then** the system creates the tenant in `Active` state, creates the
   administrator user associated with that tenant, and assigns the selected plan.
2. **Given** that a tenant already exists with the same identifier/subdomain,
   **When** attempting to create another tenant with that same identifier, **Then** the system
   rejects the operation with a clear "identifier already in use" error.
3. **Given** a newly created tenant, **When** its list of
   enabled modules is queried, **Then** the system returns exactly the modules included in the contracted plan
   (no more, no less).

---

### User Story 2 - Login and Per-Tenant Data Isolation (Priority: P1)

A user authenticates with their credentials and accesses only their own
tenant's data; they can never see, list, or modify another tenant's data, even if knowing or
guessing other record identifiers.

**Why this priority**: It is Principle I (Multi-Tenant Isolation First) of the
constitution, non-negotiable; without this no other module can be considered secure.

**Independent Test**: With two test tenants already created (via User Story 1), a user
from Tenant A attempts to access a resource belonging to Tenant B via direct API and the
system must deny it, verifiably without depending on any business module.

**Acceptance Scenarios**:

1. **Given** an authenticated valid user from Tenant A, **When** requesting a resource that
   belongs to Tenant B (e.g., by changing an ID in the URL), **Then** the system
   responds as if the resource doesn't exist (404) or with access denied (403), never with
   Tenant B's data.
2. **Given** invalid or expired credentials, **When** the user attempts to authenticate
   or use an expired token, **Then** the system rejects access and doesn't expose which part of
   the credential was wrong (user vs. password).
3. **Given** an authenticated user, **When** performing any read or
   write operation, **Then** every created/modified record is automatically associated with the
   authenticated user's `TenantId`, without the client being able to send it manually.

---

### User Story 3 - Role-Based Access Control (RBAC) Within a Tenant (Priority: P2)

A tenant administrator defines what roles their organization's users have
(e.g., Administrator, Accountant, Receptionist, Read-only), and the system
restricts available actions by role, consistently across all enabled
modules for that tenant.

**Why this priority**: Necessary to operate with real teams within a tenant, but
depends on tenants and authentication existing (User Stories 1 and 2) first.

**Independent Test**: With a tenant and two users of different roles already created, verify
that a user with a restricted role receives 403 on an action reserved for Administrator,
while the Administrator executes it successfully — without needing real business modules,
using a test endpoint/action protected by role.

**Acceptance Scenarios**:

1. **Given** a user with "Read-only" role, **When** attempting to execute a write
   action (create/edit/delete), **Then** the system rejects it with 403.
2. **Given** a user with "Administrator" role of a tenant, **When** assigning or revoking a
   role to another user in their same tenant, **Then** the change applies immediately and
   is recorded in the audit log (Principle V).
3. **Given** an Administrator user from Tenant A, **When** attempting to assign roles to a
   user from Tenant B, **Then** the system rejects it (no cross-tenant authority exists).

---

### User Story 4 - Subscription Plan Management and Module Activation (Priority: P2)

A platform administrator (or the tenant itself, depending on the business model)
changes a tenant's subscription plan, which automatically enables or disables
access to business modules (Invoicing, Inventory,
Accounting, CRM/Sales, Clinic, Dental Clinic) corresponding to that plan.

**Why this priority**: Necessary to monetize and for business modules (specs
002+) to have a mechanism to depend on, but doesn't block being able to build and test the
Foundation itself.

**Independent Test**: By changing a test tenant's plan between two plans with
different included module catalogs, and verifying that the list of enabled modules
queryable by API changes accordingly, without any business module being implemented
yet.

**Acceptance Scenarios**:

1. **Given** a tenant on a plan that doesn't include the "Accounting" module,
   **When** a user from that tenant attempts to access an Accounting module
   functionality, **Then** the system blocks it with a message of "module not included in
   your plan", not a generic error.
2. **Given** an active tenant, **When** its plan is updated to one that adds a new
   module, **Then** the module becomes available to that tenant without requiring
   additional manual intervention on existing data.
3. **Given** a tenant whose subscription expires or is canceled, **When** it transitions to
   `Suspended` state, **Then** users of that tenant can authenticate and consult/export
   their existing data in read-only mode, but no write operation in any
   module is executed until the subscription is reactivated.

---

### Edge Cases

- If a business module is deactivated for a tenant that already has data created in that
  module (e.g., issued invoices), existing data MUST remain visible in read-only mode;
  the system MUST only block creation/editing of new records in that module (see Clarifications, session 2026-06-05).
- What happens if two tenant onboarding requests with the same identifier/subdomain arrive simultaneously? The system must guarantee uniqueness without a race condition.
- How does a user recover a forgotten password without allowing enumeration of
  existing accounts (not revealing whether an email is registered)?
- A user who needs to access multiple tenants (e.g., an external accountant serving
  multiple clinics) MUST use a separate account/credential per tenant, since the model
  is one tenant per user (see Clarifications, session 2026-06-05).
- What happens if the last Administrator user of a tenant is deactivated or removes their
  own role? The system must prevent a tenant from being without any Administrator.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: The system MUST allow creating a new tenant with: organization name, unique identifier (subdomain or code), initial subscription plan, and data for the first Administrator user.
- **FR-002**: The system MUST prevent the creation of two tenants with the same unique identifier.
- **FR-003**: The system MUST authenticate users via credentials (user/email + password) and issue a session/token that unambiguously identifies the user and their `TenantId`.
- **FR-004**: The system MUST resolve the `TenantId` of each authenticated request from the session/token, and MUST ignore/reject any `TenantId` explicitly sent by the client in the body or request parameters.
- **FR-005**: The system MUST apply a tenant isolation filter to all tenant-owned data queries, so no endpoint can return or modify data from a tenant other than the authenticated user's.
- **FR-006**: The system MUST support per-tenant configurable roles (at minimum: Administrator, and additional roles defined by the tenant) and MUST restrict actions based on the authenticated user's role.
- **FR-007**: The system MUST allow an Administrator-role user of a tenant to invite/create, edit, and deactivate users and roles within their own tenant, and MUST prevent administering users from another tenant.
- **FR-008**: The system MUST prevent a tenant from being without any active Administrator-role user.
- **FR-009**: The system MUST maintain a subscription plan catalog, each with a defined set of included business modules.
- **FR-010**: The system MUST expose, for the authenticated tenant, the list of business modules currently enabled according to their current plan.
- **FR-011**: The system MUST block all **write** operations on a module not enabled for the tenant, returning an explicit reason ("module not included in your plan") distinguishable from a role denial. **Read** operations on that module's already-existing data are governed by FR-012a and NOT blocked.
- **FR-012**: The system MUST allow changing a tenant's plan and reflect the enabled modules change immediately (without requiring manual data migration).
- **FR-012a**: When a module transitions from enabled to disabled for a tenant (by plan change or manual deactivation), the system MUST keep that module's already-existing data accessible in read-only mode, and MUST only block creation/editing of new records while the module remains disabled.
- **FR-013**: The system MUST record in an immutable audit log the events of: tenant creation, plan change, user role creation/removal/change, and tenant suspension/reactivation — including who, when, and the affected tenant.
- **FR-014**: The system MUST support an `Active` tenant state and a `Suspended` state. In `Suspended` state, the system MUST allow authentication and read/export operations on existing data, and MUST reject all write operations in any module until the tenant returns to `Active`.
- **FR-015**: The system MUST provide a password recovery flow that doesn't reveal whether an account/email exists in the system.
- **FR-016**: The system MUST invalidate a user's sessions when their role is revoked or the user is deactivated, without waiting for their next login. Session revocation (inability to renew) MUST be immediate, and effective loss of access MUST occur within a maximum of 15 minutes from the change.
- **FR-017**: The system MUST support a **platform operator** identity, independent of tenant users and not belonging to any tenant, capable of authenticating without specifying a tenant. Only this identity can execute the cross-tenant operations of FR-001, FR-012, and FR-014.
- **FR-018**: The system MUST reject any attempt by a tenant user to execute operations reserved for the platform operator, and MUST audit all cross-tenant operations performed by a platform operator (Principle I).
- **FR-019**: The system MUST resolve all user-visible text through translation resources with support for **Spanish (default), English, and French**, and MUST NOT include visible strings written directly in code or returned by the API as final text. Error messages are identified by their code, and displayed text is resolved on the client (Principle VIII).
- **FR-020**: The interface MUST offer a language selector accessible from any screen, and the change MUST apply immediately without reloading the application or losing current work.
- **FR-021**: The system MUST remember the language chosen by each user by associating it with their account, so it applies when logging in from any device. While not logged in, MUST use the browser's latest preference and, if not available, the browser's language if supported; if not, the default language.
- **FR-022**: Catalogs that the tenant creates and edits (categories, payment methods, treatment types, etc.) MUST be translatable to each supported language via data, without requiring deployment. If translation is missing in the active language, the system MUST display the text in the tenant's default language instead of an empty value.

### Key Entities

- **Platform Operator**: **Global** identity (does not belong to any tenant) that
  administers the SaaS: onboards tenants, changes plans, and suspends/reactivates tenants. It is the
  only identity authorized to operate cross-tenant, and all its activity is audited (FR-018).
- **Tenant**: Client organization (company, clinic, etc.). Key attributes: name,
  unique identifier/subdomain, state (Active/Suspended), current subscription
  plan, onboarding date.
- **User**: Person who accesses the system. Belongs to exactly one Tenant (1:N Tenant→User
  relationship). Key attributes: credentials, email, state
  (active/disabled), role(s) assigned within their tenant.
- **Role**: Named set of permissions (Administrator, and roles additional roles defined
  per tenant). A user has one or more roles within a tenant.
- **Subscription Plan**: Commercial definition that determines which Business Modules are
  included for a tenant. In this first version it does not include quantitative limits
  (users, storage, etc.); those limits are explicitly out of scope and
  can be added as a subsequent incremental story.
- **Business Module**: Catalog entry (Invoicing, Inventory, Accounting,
  CRM/Sales, Clinic, Dental Clinic, …) that may or may not be enabled for a tenant
  based on their plan. This spec only models the catalog and its activation; the content of each
  module is specified in its own specs (see `ROADMAP.md`).
- **Audit Record**: Immutable event with tenant, actor, action, affected entity,
  timestamp, and relevant before/after state.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: A new tenant can onboard and have its first Administrator user
  successfully authenticate in less than 5 minutes of flow (onboarding).
- **SC-002**: 0 cross-tenant data leak incidents detected in the isolation test
  suite (100% of tenant-owned data endpoints have at least one test that
  verifies that a user from another tenant cannot access them).
- **SC-003**: 100% of role-restricted actions return 403 for users without
  the required role, verified by automated tests for each defined role.
- **SC-004**: A plan change is reflected in the tenant's list of enabled modules
  in less than 1 second from when the change is confirmed (no stale cache).
- **SC-005**: 100% of audit events defined in FR-013 are recorded and
  queryable, verified by automated tests.

## Assumptions

- The first tenant onboarding channel is managed (by a platform operator or
  commercial team); public self-registration (self-service signup) can be added
  later as an additional story, does not block this spec.
- Authentication is based on own credentials (username/password) managed by
  the Foundation; integration with external SSO/OAuth (Google, Microsoft) is out of
  scope for this first version and will be evaluated as a later improvement.
- A user belongs to a single tenant (confirmed in Clarifications, session
  2026-06-05); access to multiple tenants requires a separate account per tenant.
- Subscription plans only determine the catalog of enabled modules; quantitative
  limits per plan (user cap, storage, number of documents) are out of scope for this version (Clarifications, session 2026-06-05).
- The initial Business Module catalog corresponds to those listed in `ROADMAP.md`
  (Invoicing, Inventory, Accounting, CRM/Sales, Clinic, Dental Clinic); new
  modules can be added to the catalog without requiring changes to this spec.
- Subscription plans and their relationship with payment/billing means of the SaaS
  platform itself (charging tenants) are treated as out of scope for this spec —
  this spec only covers which modules each plan enables, not the charging itself.
