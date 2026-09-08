# Roadmap of Specifications — AEspejo ERP SaaS

This document is the high-level backlog of modules to specify with Spec Kit.
It does not replace the specs: each row becomes its own `specs/NNN-<module>/spec.md`
via `/speckit-specify` when its turn comes, following the flow described in
`.specify/memory/constitution.md` (Development Workflow & Quality Gates).

## Proposed Order

| # | Module | Depends on | Status |
|---|--------|-----------|--------|
| 001 | Multi-Tenant SaaS Foundation (tenants, auth/RBAC, plans and subscriptions, module activation) | — | 🟢 In implementation — US1 and US2 complete (tenant onboarding, login, isolation verified); pending US3 (RBAC) and US4 (plans/modules) |
| 002 | Electronic Invoicing | 001 | ⬜ Pending |
| 003 | Inventory | 001 | ⬜ Pending |
| 004 | General Accounting | 001, 002, 003 (journal entry source) | ⬜ Pending |
| 005 | CRM / Sales | 001, 002 | ⬜ Pending |
| 006 | Medical Clinic Management (general) | 001, 002, 003 | ⬜ Pending |
| 007 | Dental Clinic Management | 001, 006 (extends general Clinic with odontogram, dental treatments) | ⬜ Pending |

> Order 002→005 can be freely reordered; the only fixed constraint is that 001 (Foundation)
> comes first because all business modules depend on the multi-tenancy, auth, and
> module activation it provides. 006/007 assume reusing Invoicing and Inventory
> as shared kernel (appointments → supply consumption → invoice), aligned with
> Principle IV (Modular Vertical Architecture) of the constitution.

## How to Continue

For the next module on the list:

```
/speckit-specify Specification of the <name> module for AEspejo ERP SaaS, tenant-aware, ...
/speckit-clarify
/speckit-plan
/speckit-tasks
/speckit-implement
```

Update the **Status** of the corresponding row (⬜ Pending → 🟡 Spec created →
🟢 Planned → ✅ Implemented) as each module advances through the workflow.
