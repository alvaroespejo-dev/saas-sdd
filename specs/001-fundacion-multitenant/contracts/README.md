# API Contracts — Multi-Tenant SaaS Foundation (001)

Contract-first (Principle II): these contracts are defined **before** implementing the
endpoints and are the source of truth for contract tests and the generated Angular
client.

- [`platform-api.openapi.yaml`](./platform-api.openapi.yaml) — Foundation API.

## Cross-Cutting Conventions

**Language (Principle VIII)**: routes, DTO properties, parameters, and error codes go
**in English**. The error `message` field is supportive text for diagnosis, not
the string displayed to the user: the UI resolves the visible message from the `code`
via i18n (es/en), so the same error is read in each user's language.

**Base path**: `/api/v1`

**Authentication**: `Authorization: Bearer <access_token>` (JWT, ~15 min). The `tenant_id`
travels as a claim inside the token and **is never** accepted from the client (D-002/FR-004).
Any `tenantId` present in a body is ignored or rejected with `400`.

**Error format** (uniform across all endpoints):

```json
{
  "code": "MODULE_NOT_IN_PLAN",
  "message": "The Accounting module is not included in your plan.",
  "traceId": "00-4bf92f...-01"
}
```

**Platform error codes** — must be distinguishable from each other (FR-011, FR-014):

| HTTP | `code` | Meaning |
|---|---|---|
| 401 | `UNAUTHENTICATED` | No token, invalid or expired token |
| 403 | `FORBIDDEN_ROLE` | Authenticated, but role doesn't allow action (FR-006) |
| 403 | `FORBIDDEN_SCOPE` | Wrong identity type: tenant token on platform endpoint, or vice versa (FR-018) |
| 403 | `MODULE_NOT_IN_PLAN` | **Write** on a module not enabled for the tenant (FR-011) |
| 403 | `TENANT_SUSPENDED` | Tenant suspended: **write** blocked (FR-014) |
| 404 | `NOT_FOUND` | Doesn't exist **or belongs to another tenant** (never distinguished: FR-005) |
| 409 | `SLUG_ALREADY_IN_USE` | Tenant identifier already taken (FR-002) |
| 409 | `LAST_ADMIN` | Operation would leave tenant without active Administrator (FR-008) |
| 400 | `PASSWORD_TOO_WEAK` | Password doesn't meet tenant policy (FR-023) |
| 400 | `PASSWORD_RESET_TOKEN_INVALID` | Recovery link unknown, expired, or already used (FR-015) |

### Decision Rule: Module Enabled vs. Disabled

To avoid code overlap, the rule is **single** and depends only on the
type of operation:

| Module State for Tenant | Read Operation | Write Operation |
|---|---|---|
| Enabled | Allowed | Allowed (subject to role) |
| Disabled (outside plan or deactivated) | **Allowed** — returns existing data, which may be empty (FR-012a) | `403 MODULE_NOT_IN_PLAN` (FR-011) |

There is no separate code for "module in read-only": read access to historical data
simply **is allowed**, and only writes are rejected. The UI decides what to
show based on `GET /modules` (`enabled`), not based on error codes.

**Isolation rule in contracts**: no tenant-owned data endpoint accepts a
tenant identifier as a parameter. Requesting a resource from another tenant **always**
returns `404 NOT_FOUND`, never `403`, to avoid leaking resource existence (FR-005,
User Story 2).
