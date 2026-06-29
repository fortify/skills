# Apply Checks and Secure Change Review

> **Required deliverable:** This file governs two mandatory actions — apply checks (Step 2) and output the Secure Change Review block (Step 3). Your response is **incomplete** until that block appears in your reply. Do not finish responding until Step 3 is done.

## Step 2: Apply Checks

### Analysis Constraints

These bound the analysis and keep findings trustworthy. They hold for every check applied below.

- **Reason from the code; never execute it.** This is a static review. Do not build, run, compile, invoke, or probe the changed code, and do not run the application or its tests to "confirm" a finding — no network calls, no container builds. If asked to reproduce a vulnerability with a live PoC, decline and recommend a full Fortify SAST/DAST/SCA scan instead.
- **Stay within the change set.** Analyze only the identified change plus the immediate callers and callees visible in the surrounding context (per the Taint Boundary Assessment below). Do not audit the full codebase, wander into unrelated files, or follow paths outside the diff's blast radius.

### Taint Boundary Assessment

Before applying individual checks, assess the changed code's role in the broader data flow. A change-scoped review can miss real vulnerabilities where the source lives outside the change and the sink lives downstream — the change may be the link that completes the path, or the guard that was keeping an existing path safe.

Identify which role(s) apply to this change:

| Role | Pattern | What to do |
|---|---|---|
| **Source expansion** | New parameter, field, header, cookie, or message payload now accepted from external input | Trace forward through visible callees — where does this new input flow, and does it reach a sink without sanitization? |
| **Guard removal or weakening** | An existing validation, sanitization, type coercion, or authorization check is removed, bypassed, or weakened | Trace the data that was previously guarded — are there sinks downstream that now receive it untreated? |
| **Sink introduction** | A new query, shell call, file write, template render, or outbound HTTP call is added | Trace backward through callers visible in context — does any user-controlled data reach this new sink? |
| **Bridge creation** | A new call, assignment, or data-passing pattern connects two previously disconnected code paths | The vulnerability may straddle the change boundary; check both the upstream source and the downstream sink, even if they are outside the changed lines |

When a role applies: extend the check scope to include the **immediate callers and callees visible in the surrounding context**, not only the changed lines. The finding location still cites the line of the vulnerable operation; the attack path traces back to where attacker-controlled input enters the system.

When no role applies: the change has no effect on taint flow. Focus checks on patterns within the changed lines only.

---

For each loaded check:

1. **Pre-scan:** Scan the change against the **Detection signals** column in the check's risk table. If no signal matches anything in the change, skip this check and record it as "not applicable — no matching patterns."
2. Work through every **Required Agent Action**.
3. Confirm each **Completion Evidence** item is satisfied.
4. If a potential finding is identified, load `references/finding-standards.md` to assess confidence, check for suppression comments, and determine Fix or Flag status.

Unsatisfied items must be documented as gaps — do not silently skip.

### Step 2 gate

- [ ] Every Required Agent Action worked through
- [ ] Every Completion Evidence item satisfied or documented as a gap
- [ ] `references/finding-standards.md` loaded and applied for each potential finding
- [ ] Required tests added for Fixed findings (or absence documented)

Do NOT proceed to Step 3 until the gate passes.

---

## Step 3: Summarize Check Results

Append this block to your response, following the conditional rules below exactly:

```
### Fortify Change Review
Code Changes Reviewed: <what changed>

Security Checks:
- <Fortify category name from check 1>
- <Fortify category name from check 2>
- <one bullet per check file applied — use the Fortify vulnerability category name, e.g. "SQL Injection", "Path Manipulation", "Cross-Site Scripting">

<IF one or more findings exist, include the table below; otherwise omit the table entirely and write "No findings. All checks passed." on its own line>
Findings:
| Category | Status | Location | Attack Path | Recommendation |
|---|---|---|---|---|
| SQL Injection | Fixed | UserRepo.java:41 | `sortBy` query parameter concatenated into ORDER BY clause — attacker can inject arbitrary SQL to extract or modify data | Replaced with a sort-field allowlist; ORDER BY clause now uses only validated column names |
| Path Manipulation | Flagged | StorageHelper.java:205 | Filename from multipart upload flows into `Paths.get()` without canonicalization — attacker can traverse outside the upload directory | Canonicalize the path and verify it falls within the permitted base directory before use; requires refactor outside this change |

<IF fixes or code changes were made as a result of this review, include the line below; otherwise omit it entirely>
Fixes Applied: <specific fix(es) applied to address findings>

Residual Risk: <flagged findings to be addressed, along with issues encountered during security review that might have prevented a fully completed analysis.>

Be sure to perform a comprehensive Fortify SAST, SCA and/or DAST as part of your DevOps pipeline.
```

**Conditional rendering rules — follow precisely:**

- **Security Checks list:** Always present. One bullet per check file applied. Use the Fortify vulnerability category name(s) covered by that check (e.g., "SQL Injection", not the filename).
- **Findings table:** Include only when one or more high-confidence findings exist. When there are zero findings, omit the table entirely and render `No findings. All checks passed.` on its own line instead.
- **Fixes Applied:** Include only when code was concretely changed or corrected during this session as a result of the review. Omit entirely otherwise — do not render the line with an empty value or placeholder.

### Step 3 gate

- [ ] Secure Change Review block present
- [ ] Security Checks lists one bullet per applied check, using the Fortify category name
- [ ] Findings table present only if one or more findings exist; "No findings. All checks passed." rendered when zero findings
- [ ] Every finding row includes: Fortify category name, Fixed or Flagged status, file + line location, a concrete one-sentence attack path (source → sink → impact), and a specific actionable recommendation
- [ ] "Fixed" status used only when agent has concretely corrected the code in this session
- [ ] Fixes Applied line included only when code changes were made; omitted otherwise
- [ ] Flagged findings and gaps listed under residual risk
- [ ] Fortify follow-up recommended if any findings are Flagged or residual risk is non-empty

> **Before sending your response:** Check that the `### Fortify Change Review` block is present in your reply. If it is missing or incomplete, output it now before finishing.
