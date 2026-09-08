# AEspejo ERP SaaS — Spec-Driven Development (SDD)

![Status](https://img.shields.io/badge/Status-In%20Development-blue)
![Language](https://img.shields.io/badge/Language-English-green)
![Date](https://img.shields.io/badge/Date-2026--06--05-informational)

## 📋 Overview

This repository contains the **complete technical specification** and **implementation plan** for **AEspejo ERP SaaS**, a multi-tenant enterprise application built with **.NET 10** and **Angular**, following the **Spec-Driven Development (SDD)** methodology with **Claude Code** tools.

The project is organized into **independent specifications** (features), each fully documented with:
- Technical research (ADRs - Architectural Decision Records)
- Functional specification and requirements
- Detailed implementation plan
- Data model
- API contracts (OpenAPI)
- Executable task lists

---

## 🎯 What is Spec-Driven Development (SDD)?

**Spec-Driven Development** is a methodology where:

1. **Specification precedes code**: Every requirement, architectural decision, and pattern is documented before writing code.
2. **Contracts are the source of truth**: APIs are defined via OpenAPI/Swagger before implementation.
3. **Tests are mandatory**: They are written and executed first, fail, then functionality is implemented (Test-First).
4. **Full traceability**: Every requirement (FR-xxx), success criterion (SC-xxx), and task (Txxx) is linked throughout documentation.
5. **Vertical modularity**: Features are designed as independent, testable, deployable modules.

### Benefits of SDD

✅ **Clarity**: All stakeholders understand what will be built before spending resources.
✅ **Fewer bugs**: Requirements are validated before coding; scope changes are explicit.
✅ **Maintainability**: Documentation is living and linked to code.
✅ **Scalability**: New modules follow the same pattern without reinvention.
✅ **Auditability**: Every decision has its rationale and traceability.

---

## 📁 Project Structure

```
AEspejo.SaaS-SDD/
├── README.md                           # This file
├── CONSTITUTION.md                     # Project core principles
├── ROADMAP.md                          # Feature roadmap (1-7)
│
├── specs/                              # Specifications per feature
│   ├── 001-fundacion-multitenant/      # Feature 1: Multi-Tenant SaaS Foundation
│   │   ├── spec.md                     # Functional specification (requirements FR-001 to FR-022)
│   │   ├── plan.md                     # Implementation plan + Constitution Check
│   │   ├── research.md                 # Technical decisions (D-001 to D-007)
│   │   ├── data-model.md               # Entities, relationships, invariants
│   │   ├── quickstart.md               # End-to-end validation guide (V1-V5)
│   │   ├── tasks.md                    # Executable tasks per phase (T001-T081)
│   │   └── contracts/                  # API contracts
│   │       ├── README.md               # API conventions and error codes
│   │       └── platform-api.openapi.yaml # OpenAPI contract (Contract-First)
│   │
│   └── 002-invoicing/ (...)            # Future features
│       └── [similar structure]
│
├── .specify/                           # Claude Code tool configuration
│   ├── memory/                         # Persistent memory (project facts)
│   │   └── constitution.md             # Constitution versions and amendments
│   ├── templates/                      # Templates for /speckit-* skills
│   ├── workflows/                      # Executable workflows (speckit)
│   └── integrations/                   # Integration manifests
│
├── backend/                            # .NET implementation (not yet created)
│   ├── src/
│   │   ├── Platform.Api/               # Endpoints, middleware, DI
│   │   ├── Platform.Core/              # Domain: Tenant, User, Role, Plan
│   │   ├── Platform.Infrastructure/    # DbContext, migrations, Identity
│   │   └── SharedKernel/               # Shared abstractions (ITenantOwned, etc)
│   └── tests/
│       ├── Platform.Contract.Tests/    # Contract tests (OpenAPI)
│       ├── Platform.Integration.Tests/ # Integration tests (real DB)
│       └── Platform.Unit.Tests/        # Unit tests (domain rules)
│
└── frontend/                           # Angular implementation (not yet created)
    ├── src/app/
    │   ├── core/                       # HTTP interceptors, auth, guards
    │   ├── shell/                      # Layout, navigation
    │   └── features/                   # Feature modules (auth, users, roles, etc)
    └── src/tests/                      # Component & integration tests
```

---

## 🏗️ Feature 001: Multi-Tenant SaaS Foundation

### Description
Build the **Platform Core** of the SaaS ERP: tenant onboarding and lifecycle, authentication, per-tenant RBAC, subscription plans, and module activation.

### Key Documents

| Document | Purpose | Link |
|----------|---------|------|
| **spec.md** | Functional requirements, user stories, success criteria | [Read](./specs/001-fundacion-multitenant/spec.md) |
| **plan.md** | Technical strategy, architectural decisions, Constitution Check | [Read](./specs/001-fundacion-multitenant/plan.md) |
| **research.md** | ADRs: 7 fundamental technical decisions (D-001 to D-007) | [Read](./specs/001-fundacion-multitenant/research.md) |
| **data-model.md** | Entities, relationships, conventions (9 tables + rules) | [Read](./specs/001-fundacion-multitenant/data-model.md) |
| **tasks.md** | 81 tasks organized in 7 phases, with dependencies | [Read](./specs/001-fundacion-multitenant/tasks.md) |
| **quickstart.md** | Validation guide: 5 end-to-end scenarios (V1-V5) | [Read](./specs/001-fundacion-multitenant/quickstart.md) |
| **contracts/README.md** | API conventions, error codes, isolation rules | [Read](./specs/001-fundacion-multitenant/contracts/README.md) |

### Architectural Decisions (D-001 to D-007)

1. **D-001**: Angular (LTS) framework + TypeScript strict mode
2. **D-002**: Global Query Filters + server-side TenantId resolution
3. **D-003**: ASP.NET Core Identity + JWT (15 min) + revocable refresh tokens
4. **D-004**: Module catalog in database (not code); authorization in pipeline
5. **D-005**: EF Core interceptor for append-only audit logging
6. **D-006**: Modular monolith (.NET projects per module boundary)
7. **D-007**: Performance targets (p95 < 300ms, ~500 tenants, 7-year audit retention)

### Functional Requirements (FR-001 to FR-022)

**Identity & Authentication**:
- FR-001: Create tenant with plan and admin
- FR-002: Unique slug (prevents race condition)
- FR-003: Authentication with credentials
- FR-004: TenantId only from server-side claim
- FR-017: Platform operator identity (global, cross-tenant)

**Isolation (Principle I)**:
- FR-005: Global Query Filter on all queries
- FR-006: Configurable roles per tenant
- FR-007: Admin only manages their tenant
- FR-008: Invariant: last admin cannot be removed

**Modules & Plans**:
- FR-009: Plans catalog
- FR-010: List tenant's enabled modules
- FR-011: Block writes to disabled module (403 MODULE_NOT_IN_PLAN)
- FR-012: Plan change reflects modules in <1s (SC-004)
- FR-012a: Read allowed, write blocked on disabled module

**Operational**:
- FR-013: Append-only audit (tenant created, plan changed, role changed, suspended)
- FR-014: Suspended state (read yes, write no)
- FR-015: Password recovery (no enumeration)
- FR-016: Session revocation in ≤15 min on role change
- FR-018: Audit all platform operator operations

**i18n (Principle VIII)**:
- FR-019: i18n (es default, en, fr) via resource files
- FR-020: Language selector on any screen
- FR-021: Remember user language + fallback
- FR-022: Tenant-editable catalogs → translation tables

### Success Criteria (SC-001 to SC-005)

| Criterion | Target | Verification |
|-----------|--------|--------------|
| SC-001 | Tenant onboarding + 1st login < 5 min | Manual timing in quickstart.md |
| SC-002 | 0 cross-tenant leaks; 100% endpoints tested | Leak test per endpoint |
| SC-003 | 100% role-restricted actions → 403 FORBIDDEN_ROLE | Parameterized tests |
| SC-004 | Plan change reflected in <1s (no stale cache) | Integration test with timing |
| SC-005 | 100% audit events recorded and queryable | AuditLogTests.cs |

### Implementation Phases

| Phase | Name | Purpose | Est. Duration |
|-------|------|---------|----------------|
| 1 | Setup | .NET solution, test projects, Angular, CI | 2-3 days |
| 2 | Foundational | Isolation, auth, audit (CRITICAL) | 3-5 days |
| 3 | US1 | Tenant onboarding | 1-2 days |
| 4 | US2 | Login and isolation | 2-3 days |
| 5 | US3 | RBAC | 2 days |
| 6 | US4 | Plans and modules | 2 days |
| 7 | Polish | Validation, indexes, rate limiting | 2-3 days |
| **Total** | | | **14-20 days** |

---

## 🛠️ Work Completed with Claude Code

### Structured Documentation

**Principles applied**:
1. **Contract-First** (Principle II): OpenAPI defined before implementation
2. **Test-First** (Principle III): Tests written and fail before code
3. **Multi-Tenant Isolation First** (Principle I): Isolation is central, non-negotiable
4. **Modular Architecture** (Principle IV): Module boundaries verifiable at compile-time
5. **Security & Compliance by Design** (Principle V): Audit and auth from the start
6. **English-First Code** (Principle VIII): All identifiers in English; i18n for UI only

**Complete traceability**:
- Each User Story (US1-US4) linked to requirements (FR-xxx)
- Each requirement linked to success criteria (SC-xxx)
- Each task linked to user story (T001-T081)
- Each architectural decision (D-001-D-007) justified in research.md

---

## 📚 How to Use This Documentation

### For Product Managers / Stakeholders

1. Read **spec.md** → Understand requirements and user stories
2. Consult **quickstart.md** → Validation scenarios
3. Review **plan.md** → Constitution Check (ensure alignment)

### For Architects

1. Study **research.md** → Decisions and alternatives
2. Analyze **data-model.md** → Entities and invariants
3. Review **contracts/README.md** → APIs and conventions
4. Consult **plan.md** → Constitution and dependencies

### For Developers

1. Start with **tasks.md** → Tasks for your phase
2. Read **plan.md** → Technical context and project structure
3. Review **contracts/platform-api.openapi.yaml** → API to implement
4. Consult **data-model.md** → Entities and rules
5. Run tests from **quickstart.md** → End-to-end validation

### For QA / Testers

1. Read **spec.md** → Requirements to verify
2. Consult **quickstart.md** → Scenarios V1-V5
3. Review **tasks.md** → Tests per user story
4. Run automated suite (dotnet test, npm test)

---

## 🚀 Next Steps

### Immediate Phase

1. **Create Git repository** with this structure
2. **Initialize .NET solution** (backend/src and backend/tests)
3. **Scaffold Angular** (frontend/)
4. **Configure CI/CD** (.github/workflows/)
5. **Execute Phase 1 (Setup)** → tasks T001-T008

### Future Features

- **002-Invoicing**: Documents, lines, taxes, currencies
- **003-Inventory**: Products, stock, movements
- **004-Accounting**: Ledger entries, trial balance, financial reports
- **005-CRM/Sales**: Opportunities, quotes, orders
- **006-Medical Clinic**: Patients, consultations, medical history
- **007-Dental Clinic**: Patients, treatments, odontograms

Each feature will follow the **same SDD pattern**:
- Complete specification (spec.md)
- Documented decisions (research.md)
- Data model (data-model.md)
- Implementation plan (plan.md)
- Executable tasks (tasks.md)
- End-to-end validation (quickstart.md)
- API contract (contracts/...)

---

## 🔐 Security Principles

### Multi-Tenant Isolation (Principle I)

**Non-negotiable**: Every query, log, job, cache entry must be scoped by TenantId.

- Global Query Filter on **all** tenant-owned queries
- TenantId resolved **only** from server-side claim (never from client)
- Audit log of **all** cross-tenant operations
- 100% coverage of cross-tenant leak tests (SC-002)

### Test-First Security (Principle III)

- Cross-tenant leak test **before** implementing
- Role-based access control tests **before** endpoints
- "Last admin" invariant **before** removing admin

### No Secrets in Repo

- Connection string: `dotnet user-secrets set`
- Secrets: Azure Key Vault in production
- Passwords hashed with ASP.NET Core Identity

---

## 📖 Related Documents

- **CONSTITUTION.md** — Core principles (v1.3.0, ratified 2026-06-05)
- **ROADMAP.md** — Roadmap: features 001-007, estimated timeline
- **.specify/memory/constitution.md** — Historical versions and amendments

---

## 🤝 Contributions

### How to Contribute

1. **Specification changes**: Edit spec.md directly; record amendments in plan.md
2. **New tasks**: Add to tasks.md; update dependencies
3. **Architecture changes**: Create new ADR in research.md; update constitution.md
4. **Bugs found**: Document in issues + update affected specs

### Governance

- **Principles**: Non-negotiable (Principle I, II, III, IV, V)
- **Constitution amendments**: Require consensus; record in constitution.md
- **Scope changes**: Require stakeholder review; impact timeline

---

## 📞 Contact

**Project**: AEspejo ERP SaaS
**Methodology**: Spec-Driven Development (SDD)
**Tools**: Claude Code, .NET 10, Angular (LTS), SQL Server
**Start Date**: 2026-06-05

**Owners**:
- Product Owner: [To be defined]
- Tech Lead: [To be defined]
- QA Lead: [To be defined]

---

## 📄 License

[To be defined per project policy]

---

**Last updated**: 2026-06-05 | **Documentation version**: 1.0.0

---

## Quick Links

📋 **Specifications**
- [Functional Specification (spec.md)](./specs/001-fundacion-multitenant/spec.md)
- [Implementation Plan (plan.md)](./specs/001-fundacion-multitenant/plan.md)
- [Technical Decisions (research.md)](./specs/001-fundacion-multitenant/research.md)

🗂️ **Technical Detail**
- [Data Model (data-model.md)](./specs/001-fundacion-multitenant/data-model.md)
- [Tasks (tasks.md)](./specs/001-fundacion-multitenant/tasks.md)
- [API Contract (platform-api.openapi.yaml)](./specs/001-fundacion-multitenant/contracts/platform-api.openapi.yaml)

✅ **Validation**
- [Validation Guide (quickstart.md)](./specs/001-fundacion-multitenant/quickstart.md)
- [API Conventions (contracts/README.md)](./specs/001-fundacion-multitenant/contracts/README.md)

📜 **Governance**
- [Project Constitution (CONSTITUTION.md)](./CONSTITUTION.md)
- [Roadmap (ROADMAP.md)](./ROADMAP.md)
