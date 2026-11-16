# ADR-010: Authentication with short-lived JWTs and rotating refresh tokens, authorization with RBAC plus domain policies

- **Status:** Accepted · **Date:** 2026-11-12 · **Deciders:** Masum Islam Badsha
- **Related:** [ADR-003 Multi-tenancy](ADR-003-multi-tenancy.md) · [ADR-005 Approval workflow storage](ADR-005-approval-workflow-storage.md) · [ADR-006 PO/invoice consistency](ADR-006-po-invoice-consistency.md) · [ADR-009 Redis](ADR-009-redis.md) · [API structure](../03-architecture/api-structure.md) · [Engineering practices](../07-engineering/engineering-practices.md)

## Context (the problem)

SupplyMesh has five roles from the brief: Organization Owner, Procurement Manager, Department Manager, Finance Officer and Employee. Roles alone do not describe the real rules:

- a Department Manager approves only for **their** department;
- an approver may approve only **up to** an amount (`member_roles.approval_limit_amount`, in base currency);
- segregation of duties: a requester cannot approve their own request; the PO creator cannot record or approve the payment for it; the invoice recorder cannot approve the invoice (`chk_inv_sod`);
- one person can be a member of several organizations with different roles in each.

Authentication must also protect high-value actions (payments, supplier bank details) and allow quick revocation when someone leaves.

## Decision drivers

- Revoking a member or a role must take effect within seconds, not at token expiry.
- Stateless `api` replicas (ADR-012) should not need a session lookup on every request.
- Refresh tokens are the long-lived credential; theft must be detectable.
- Owner and Finance Officer accounts can move money and change bank details, so they need MFA.
- Organizations add custom roles; permissions must be configurable per organization.
- Department scope, approval limits and SoD need data from the request itself, so they cannot be checked by a role name alone.

## Options considered

**Authentication**

| Option | How it would work here | Pros | Cons |
|---|---|---|---|
| Server-side sessions (cookie + session store) | Session id cookie; session row in Redis or PostgreSQL read on every request | Instant revocation; simple | Session lookup per request; sessions in Redis would make Redis a source of truth (ADR-009) |
| **Short JWT + rotating opaque refresh token** | 15-min RS256 access JWT; 14-day refresh token hashed in `user_sessions`; httpOnly cookie | Stateless request auth; revocation through short expiry + membership check; theft detection through rotation | Two token types to implement; access token valid until expiry unless the membership check rejects it |
| External IdP (Auth0, Cognito, Keycloak) | Hosted login; we consume OIDC tokens | SSO/SAML and MFA out of the box; less security code to own | Per-user cost; multi-organization membership and org switching still modelled by us; another vendor in the login path. Good fit later for enterprise SSO |

**Authorization**

| Option | How it would work here | Pros | Cons |
|---|---|---|---|
| Role checks in code (`if role === 'finance_officer'`) | Hard-coded role names | Simple | Custom roles impossible; rules scattered |
| **Permission-based RBAC + scoped roles + domain policies** | `permissions` catalog, `role_permissions`, `member_roles` with `scope_department_id` and `approval_limit_amount`; policies in services | Configurable per org; scope and limits in data; business rules close to the domain | Two layers to understand (permission guard + policy) |
| ABAC / policy engine (OPA, Cedar, CASL) | Policies as rules over attributes | Very expressive; central policy | Another language and runtime; overkill for five roles and a handful of SoD rules |

## Decision

**Authentication**

- Email + password hashed with **Argon2id**; optional TOTP MFA, **mandatory for Owner and Finance Officer** roles. SSO (SAML/OIDC) is future scope.
- **Access token:** JWT, RS256, 15 minutes. Claims: `sub` (user_id), `org_id`, `member_id`, `sid` (session id). **No permissions in the token.** It is delivered as an httpOnly, Secure, `SameSite=Lax` cookie on the shared origin (ADR-012), so JavaScript can never read it and the Next.js server can read it to render pages. The API accepts it from that cookie, or from an `Authorization: Bearer` header (used by the Next.js server and by future machine clients).
- **Refresh token:** opaque random value, 14 days, sent as an httpOnly, Secure, SameSite=Strict cookie. Only its hash is stored in `user_sessions.refresh_token_hash`. The cookie value is `<session id>.<secret>`. On refresh the API loads the `user_sessions` row by id and compares hashes. A match rotates the token: a new secret is issued and the row's hash replaced. A mismatch on a known, unrevoked session means an old token was replayed, so the session is revoked (`revoked_at`) and the user must log in again. The session row is the token family; no extra columns are needed.
- **CSRF.** Cookies are sent automatically, so every state-changing request (`POST`, `PUT`, `PATCH`, `DELETE`) must carry an `Origin` header equal to the web origin, or it is rejected with `403`. `Lax` is used for the access cookie, not `Strict`, so that a deep link from an approval email arrives logged in; `Lax` still never sends the cookie on a cross-site `POST`.
- **Rejected: access token kept in browser memory.** It keeps the token out of cookies, but the Next.js server could then not read it, so server-rendered pages and the login redirect in middleware would not work.
- **Switching organization** issues a new access token for the other membership.

**Authorization: guard chain on every request**

1. `JwtAuthGuard` verifies signature and expiry.
2. `TenantContextGuard` verifies an `active` membership for `org_id` and sets the tenant context (ADR-003).
3. `PermissionGuard` checks a code such as `@RequirePermission('purchase_request.approve')` against the member's permission set from Redis (`org:{orgId}:member:{memberId}:perms`, TTL 10 min, deleted on any role or `member_roles` change).
4. **Domain policies in the service**, inside the transaction: department scope (`scope_department_id`), `approval_limit_amount ≥ estimated_total_base`, and the SoD rules.

| Role (system, `is_system = true`) | Example permissions | Typical policy limits |
|---|---|---|
| Organization Owner | all except executing POs, invoices and payments (segregation of duties), incl. workflow and role configuration | MFA required |
| Procurement Manager | `supplier.verify`, `purchase_request.approve`, `purchase_order.issue` | approval limit set per member |
| Department Manager | `purchase_request.approve` | scoped to own department; approval limit |
| Finance Officer | `invoice.approve`, `payment.record` | MFA required; cannot pay POs they created |
| Employee | `purchase_request.create` | own requests only |

## Reasons for selection

1. **Quick revocation without per-request session reads.** Permissions are not in the token and are cached with explicit invalidation; deactivating a member fails `TenantContextGuard` on the next request; the access token itself expires in 15 minutes.
2. **Refresh-token theft is detectable** with the existing `user_sessions` table, and same-site cookies plus the `Origin` check on every state-changing request block CSRF (ADR-012).
3. **Custom roles and per-org permissions** come from `role_permissions`, as the brief requires (the Owner must "Configure roles and permissions").
4. **Approval limits and department scope are data**, so a Department Manager of IT with a $2,000 limit is one `member_roles` row, not a new role.
5. **SoD is enforced where the facts are.** Only the service knows who created the PO; `chk_inv_sod` adds a database backstop for invoices.

## Advantages

- Stateless request authentication; `api` scales horizontally.
- One permission vocabulary for API guards, UI rendering and tests.
- Role, permission and approval-limit changes are written to `audit_logs`; authorization denials go to the structured security log with the `request_id`.

## Disadvantages

- We own security-sensitive code (password hashing, MFA, token rotation) that an IdP would provide.
- Policies live in code; a rule change needs a deploy.
- A stolen access token remains usable for up to 15 minutes if the membership is still active.

## Trade-offs accepted

- We accept owning authentication code in exchange for native multi-organization memberships and no per-user IdP cost at MVP.
- We accept a 15-minute access-token window, limited by the membership check and MFA on money-moving roles.

## Consequences / follow-ups

- Login rate limiting and lockout with backoff; generic error messages to avoid account enumeration.
- Key rotation for the RS256 signing key (`kid` header; old key kept until tokens expire); private key in AWS KMS or Secrets Manager.
- A permission matrix test runs every endpoint against every system role.
- **Revisit when:** the first enterprise customer requires SAML/OIDC SSO (add an IdP integration next to password login), or policy rules grow beyond what can be reviewed in a single service class (evaluate Cedar or OPA).
