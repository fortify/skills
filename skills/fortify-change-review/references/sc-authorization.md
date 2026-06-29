# Authorization and Object Access

## Risks

| # | Risk | Detection signal | Fortify Category | CWE | OWASP 2025 | Action |
|---|------|-----------------|-----------------|-----|------------|--------|
| R1 | Missing authorization check | Authorization check absent; write operations (PUT, DELETE, PATCH) lack an authorization check that is present on read operations | Access Control: Missing Authorization Check | CWE-862 | A01 Broken Access Control | Action 1 |
| R2 | Insecure direct object reference (IDOR) | Resource query uses client-supplied identity or resource ID parameters (such as `userId`, `tenantId`, `accountId`, or `resourceId`) without validating against the authenticated session; horizontal access possible by changing an ID in the URL or request body | Access Control | CWE-639 | A01 Broken Access Control | Action 2 |
| R3 | Mass assignment | Request body bound to a model or entity without an explicit field allowlist; privileged fields (`isAdmin`, `role`, `balance`, `id`, `createdAt`) settable from user input via `@ModelAttribute`, `update(params)`, `assign_attributes`, or equivalent | Mass Assignment: Request Parameters Bound into Persisted Objects | CWE-915 | A01 Broken Access Control | Action 3 |

## Scoped Exclusions

Do not report these patterns even if a detection signal above matches:

- *(R1, R2, R3)* **Client-side JavaScript and TypeScript** — missing authorization checks, unvalidated resource IDs, or unprotected model fields in client-side code are not findings. Client-side code is untrusted by design; it is the server's responsibility to enforce authorization. Only flag server-side code that receives requests and acts on them.

## Required Agent Actions

1. **Verify and enforce authorization before data access or mutation** *(R1)* — the check must execute on the server before the resource is retrieved or modified. Confirm the authenticated principal has permission for this specific resource or action, not just that they are logged in. If the principal lacks access, return 403 or 404 immediately — do not fetch the resource first.

2. **Enforce ownership and tenant boundaries** *(R2)* — scope resource queries to the authenticated principal's identity. Do not rely on client-supplied ID parameters (such as `userId`, `tenantId`, `accountId`, or `resourceId`) as the authorization gate; validate them against the session or derive them from the session directly.

3. **Restrict mass assignment with an explicit allowlist** *(R3)* — when binding request parameters to a model or entity, declare exactly which fields users may set. Block fields like `id`, `isAdmin`, `role`, `balance`, `createdAt`, or any field not intentionally user-settable. Only apply this fix when the model contains privileged or sensitive fields that should not be user-settable; if every field in the model is legitimately user-modifiable, an explicit allowlist is not required.

## Completion Evidence

- [ ] *(R1)* Server-side authorization check executes before data is accessed or mutated
- [ ] *(R1)* Both read and write operations on the resource are protected by an authorization check
- [ ] *(R2)* Resource queries are scoped to the authenticated principal's identity; client-supplied ID parameters are validated against the session
- [ ] *(R3)* Model/entity binding uses an explicit allowlist of permitted user-settable fields; applies only when the model contains privileged or sensitive fields
