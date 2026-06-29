# Injection-Safe Data Access

## Risks

| # | Risk | Detection signal | Fortify Category | CWE | OWASP 2025 | Action |
|---|------|-----------------|-----------------|-----|------------|--------|
| R1 | SQL injection | String concatenation or `String.format()` in SQL query construction; `raw()`, `executeQuery()`, `createNativeQuery()`, or similar ORM escape hatches called with user input | SQL Injection | CWE-89 | A05 Injection | Actions 1, 2, 3 |
| R2 | NoSQL injection | User input in `$where`, `$expr`, or query operator fields; template literals in NoSQL query construction; `find({field: userInput})` without typed parameter binding | NoSQL Injection | CWE-943 | A05 Injection | Actions 2, 3 |
| R3 | LDAP injection | User input concatenated into LDAP filter strings without escaping | LDAP Injection | CWE-90 | A05 Injection | Actions 1, 3 |
| R4 | Expression language injection | User input passed into SpEL `#{}`, OGNL, MVEL, or EL `${}` expressions evaluated at runtime | Expression Language Injection | CWE-917 | A05 Injection | Action 4 |
| R5 | Server-side template injection | User input concatenated into Jinja2, Freemarker, Velocity, or Thymeleaf template source strings at runtime rather than bound to variable slots | Expression Language Injection | CWE-917, CWE-95 | A05 Injection | Action 5 |

## Scoped Exclusions

Do not report these patterns even if a detection signal above matches:

- *(R1)* **Standard ORM filtering methods** — built-in ORM methods that use parameterized binding internally (e.g., `.findById()`, `.findByEmail()`, `.where(field: value)`, `.filter(field=value)`, ActiveRecord `.find()`) are safe by default. Only flag raw query escape hatches: `raw()`, `fromRaw()`, `executeQuery()`, `createNativeQuery()`, or direct string concatenation into a query string.
- *(R2)* **Explicitly type-coerced NoSQL inputs** — if user-supplied input is explicitly cast to a primitive type before use as a NoSQL query value (e.g., `String(req.body.field)`, `int(field)`, or schema validation that enforces a scalar type), injection via operator objects (`{"$gt": ""}`) is mitigated. Do not flag this pattern.

## Required Agent Actions

1. **Use parameterized queries or prepared statements** *(R1, R3)* — never concatenate or interpolate untrusted values into SQL or LDAP query strings. Use `?`/`:name` placeholders for SQL and equivalent escaping libraries for LDAP.

2. **Use ORM/query-builder safe APIs; avoid raw expressions** *(R1, R2)* — prefer the ORM's built-in filtering methods over `raw()`, `fromRaw()`, `$where`, `$expr`, or `executeQuery()`. If raw is unavoidable, use parameter binding. For NoSQL databases (R2), the primary injection vector is when JSON-parsed input (e.g., `req.body`) flows directly into a query value, allowing an attacker to supply an operator object (`{"$gt": ""}`) instead of a scalar. If the value is explicitly cast to a primitive type before use (e.g., `String(req.body.field)`), the injection risk is mitigated. Flag cases where object-typed sources are used as query values without explicit type coercion.

3. **Allowlist dynamic identifiers** *(R1, R2, R3)* — sort fields, column names, table names, and operators cannot be parameterized and must be validated against an explicit allowlist of permitted values before use in a query.

4. **Never pass user input into expression language evaluation** *(R4)* — user input must not reach SpEL `#{}`, OGNL, MVEL, or EL `${}` expressions at runtime. If dynamic expressions are required, use a safe, restricted evaluator with an explicit allowlist of permitted operations.

5. **Pass user input to templates as data, never as template source** *(R5)* — bind user input only to variable slots in Jinja2, Freemarker, Velocity, Thymeleaf, and equivalent engines. Never construct a template string from user-supplied text and then evaluate it.

## Completion Evidence

- [ ] *(R1, R3)* All SQL and LDAP queries use parameterized statements or safe query APIs; no string concatenation in query construction
- [ ] *(R1, R2)* ORM raw/escape-hatch methods not called with unsanitized user input; dynamic identifiers (sort field, column, table) validated against an allowlist
- [ ] *(R4)* No user-controlled input reaches SpEL, OGNL, MVEL, or EL expression evaluation
- [ ] *(R5)* User input bound only to template variable slots; no user-supplied template source strings evaluated at runtime