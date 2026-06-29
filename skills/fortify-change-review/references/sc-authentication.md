# Authentication, Session, and Credential Flow

## Risks

| # | Risk | Detection signal | Fortify Category | CWE | OWASP 2025 | Action |
|---|------|-----------------|-----------------|-----|------------|--------|
| R1 | Authentication bypass | Auth check absent, skipped on certain paths, or null/empty credential accepted; validation occurs client-side only; token compared with `==` instead of constant-time comparison | Access Control: Missing Authentication | CWE-306 | A07 Authentication Failures | Action 3 |
| R2 | Weak password reset token | Token generated with `Math.random()`, UUID, timestamp, or sequential value; no expiry or single-use enforcement | Password Management: Weak Password Policy | CWE-640 | A07 Authentication Failures | Actions 1, 2 |
| R3 | Form authentication bypass | Security constraint URL patterns are incomplete or use allow-by-default ordering; a protected resource is reachable via a path variant (trailing slash, extension, path parameter) not covered by the configured pattern | Access Control: Form Authentication Bypass | CWE-287 | A07 Authentication Failures | Action 4 |

## Scoped Exclusions

Do not report these patterns even if a detection signal above matches:

- *(R2)* **UUIDs as resource identifiers** — UUIDs used as record or resource IDs (not as authentication tokens, password reset links, or session identifiers) are assumed to be unguessable and do not require CSPRNG validation. Only flag UUID usage when the UUID is the sole factor protecting access to a resource (i.e., it functions as an authentication credential rather than a lookup key).

## Required Agent Actions

1. **Issue cryptographically random, unpredictable tokens** *(R2)* — password reset tokens, session IDs, and API keys must be generated using a CSPRNG (e.g., `crypto.randomBytes()`, `secrets.token_urlsafe()`, `SecureRandom`). Minimum 128 bits of entropy. Never use sequential IDs, UUIDs v1/v4 from non-secure sources, or timestamps as tokens.

2. **Enforce token expiry** *(R2)* — password reset tokens must expire (≤15–60 minutes is typical). Session tokens must have an absolute lifetime. Refresh tokens must be rotatable and revocable.

3. **Validate the full credential server-side** *(R1)* — do not short-circuit validation based on partial matches or encoding differences. Use constant-time comparison for token/HMAC validation to prevent timing attacks.

4. **Use deny-by-default URL pattern configuration** *(R3)* — protect resources by blocking all paths by default and explicitly permitting only public paths, rather than listing paths to protect. Verify that patterns account for trailing slashes, case sensitivity, and path parameter variations; no protected resource should be reachable via an uncovered path variant.

## Completion Evidence

- [ ] *(R2)* Tokens generated using a CSPRNG with sufficient entropy (≥128 bits)
- [ ] *(R2)* Password reset and one-time tokens have an enforced expiry and are single-use
- [ ] *(R1)* Auth checks present on all protected paths; token validation uses constant-time comparison
- [ ] *(R3)* Security constraint uses deny-by-default ordering; no protected resource reachable via an uncovered URL pattern variant
