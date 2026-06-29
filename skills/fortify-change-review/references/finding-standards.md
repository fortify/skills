# Finding Standards

Load this file when a potential finding has been identified during a check. It governs whether to report the finding, how to classify it, and whether to fix or flag it.

---

## Required Finding Elements

Before reporting any finding, confirm you can state all three elements concretely. If any cannot be stated without speculation, drop the finding.

1. **Location** — the specific file and line number (e.g., `UserRepo.java:41`). If the vulnerability spans multiple lines, cite the line where attacker-controlled input reaches the vulnerable operation. The location must correspond to code you actually read in this session — never fabricate a line number. If you cannot pinpoint the exact line, cite the function or method and note that the line is approximate.
2. **Attack Path** — one sentence: where attacker-controlled input enters (source), how it reaches the vulnerable operation (sink), and what the attacker achieves (impact). Example: "`sortBy` query parameter is concatenated directly into the SQL ORDER BY clause at `UserRepo.java:41`, enabling an attacker to inject arbitrary SQL expressions and extract or modify data."
3. **Recommendation** — a specific, actionable fix or the architectural change required. For Fixed findings, state what was changed. For Flagged findings, state what needs to change (e.g., "Validate `sortBy` against an allowlist of permitted column names before use in the ORDER BY clause").

A check that passes cleanly with no finding is a valid result. Report it explicitly under "Checks with no findings."

---

## Drop Criteria

Drop a finding if you can confirm any of the following. If uncertain, proceed — the finding stands.

**Defense-in-depth gap** — the finding identifies a missing hardening measure (e.g., absent rate limiting, missing security header) with no direct attacker-controlled path to a meaningful impact. Leave these to downstream Fortify SAST/SCA.

**Negligible impact** — a concrete mechanism exists but the practical impact is de minimis: the exposed data is non-sensitive, the operation is idempotent with no side effects, or the attack requires a chain of prerequisites a typical developer would consider implausible.

**Upstream guard** — an existing control demonstrably blocks this specific instance: ORM parameterization, framework-level auto-escaping (e.g., JSX, Thymeleaf default encoding), type-safe binding (e.g., Spring `@RequestParam`), or an auth/validation middleware that runs before this code path and covers this case.

**Test-only context** — the file's sole purpose is testing and is unreachable from production code or external input. Common indicators: `*Test.java`, `*.spec.ts`, `*.test.js`, `*_test.go`, `test_*.py`, or files under `__tests__/`, `spec/`, or `test/`.

**Trusted input source** — the value originates from an environment variable (`process.env.*`, `os.environ[...]`, `System.getenv()`), a CLI flag the operator supplies, or a developer-controlled config file. Attacker control of these inputs is out of scope.

---

## Suppression Comments

Before reporting any finding instance, check the line **immediately preceding** the flagged code for a Fortify suppression comment:

```
// FortifyRemove(ID="<guid>")
// FortifyRemove(Category="<Fortify category name>")
```

- `FortifyRemove(Category="...")` — suppresses the finding if the category name matches the Fortify category of this finding. Do not report that instance.
- `FortifyRemove(ID="...")` — suppresses the finding unconditionally regardless of category. Do not report that instance.
- Suppression for a *different* category does not suppress the current finding.
- No other comment style or annotation format counts as suppression.

---

## Fix vs. Flag

- **Fixed** — vulnerability is in code written in this session. Correct it inline before proceeding to Step 4. Add a negative exploit test.
- **Flagged** — high-confidence finding that requires an architectural change, touches code outside the current change, or cannot be safely corrected in this pass. Document for Fortify SAST/SCA follow-up.
