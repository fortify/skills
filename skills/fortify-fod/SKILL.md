---
name: fortify-fod
description: "Perform tasks against Fortify on Demand (FoD): query applications/releases; query & triage existing security issues in FoD; start & monitor full SAST/DAST/SCA/open source scans of the codebase; create releases; import FPR/SARIF/CycloneDX artifacts; policy and portfolio analysis. NOT for lightweight AI review of local code changes/diffs (use fortify-change-review)."
license: MIT
metadata:
  version: "1.2.0"
  tested-with:
    fcli: "3.18"
    fod: "26.1"
---

---

## Disambiguate "scan" / "review" first

These terms are overloaded. If ambiguous ("run a Fortify scan/review", "run a security review"), confirm user intent first:

- **scan** → 1) full SAST/DAST/SCA scan of the codebase (this skill), or 2) lightweight AI review of local code changes (`fortify-change-review`).
- **review** → 1) triage issues already found by FoD (this skill), or 2) lightweight AI review of local code changes (`fortify-change-review`).

## Verify Active FoD Session (skip if already confirmed this conversation)

```bash
fcli fod session ls --query "expired=='No'"
```

**No active FoD session found:** Instruct the user to run `fcli fod session login` and confirm successful login before proceeding.

**fcli not found:** You interact with FoD using `fcli`. It must already be on the PATH. If it is not installed, load `references/fcli-install.md`.

## FoD Logical Structure and Nomenclature

- **Applications** — top-level organizational unit; one per software project
- **Microservices** — optional grouping within an application; each microservice has its own releases. Most applications don't use microservices.
- **Releases** — versions/branches within an application or microservice (e.g., `main`, `v2.1`); each holds independent scan results and issues. A release can be seeded by copying state from another.
- **Scans** — SAST (static), DAST (dynamic), SCA (open source / OSS), or MAST (mobile) analyses attached to a release
- **Issues** — vulnerabilities found by scans; always scoped to a release

Teams typically structure FoD with one app per repo and releases per branch or version (e.g., `main`, `v25.1`). 

## Key Concepts

### Issue Status (system-assigned)
`New` (first seen), `Existing` (recurring), `Reopened` (was fixed), `Fixed` (not in latest scan, may be displayed as Fix Validated). Use `--query "status=='New'"` etc.

### Auditor Status (security decision — controls visibility)
Auditor Status is the **authoritative control for suppression**. An unreviewed issue has status `Pending Review`.
- **Suppressing** (hidden from default views and metrics): `Not an Issue`, `False Positive`
- **Non-suppressing** (visible and policy-relevant): `Remediation Required`, `Exploitable`
- Suppression persists across scans (tracked via `instanceId`)
- Values are tenant-customizable; see `references/fcli-fod-output-values.md` for full details

### Developer Status
Engineering workflow state (e.g., `Open`, `In Progress`, `Fixed`, `Deferred`). Tenant-customizable. **Does not affect suppression or policy.**

### Visibility and `--include`
By default `fcli fod issue list` returns **visible (non-suppressed) issues only**. This is what users expect unless they ask otherwise.
- `--include=visible,suppressed` — add suppressed issues
- `--include=visible,fixed` — add fixed issues
- `--include=visible,suppressed,fixed` — all issues

### Severity
`severity` is an integer: `4`=Critical, `3`=High, `2`=Medium, `1`=Low. `severityString` is the human-readable equivalent.

### Issue Counting
FoD does not have a dedicated `issue count` command. For severity totals, use precomputed release-level aggregate fields (`critical`, `high`, `medium`, `low`, `issueCount`) from `fcli fod release get` — this is fast and doesn't require listing issues. For filtered counts or category breakdowns, use `fcli fod issue list` with `-o json` and count the results. See the Issue Investigation use case for patterns.

### Open Source / SCA Components
FoD tracks open source component data from SCA scans (Debricked, Sonatype, or imported CycloneDX SBOMs). Release-level fields (`openSourceCritical`, `openSourceHigh`, etc.) provide aggregate vulnerability counts. For component-level detail — CVE impact triage, license risk, SBOM export — use the OSS Component Analysis use case.

### Fortify Aviator (AI-Assisted Audit)
FoD's Aviator feature provides AI-assisted issue review and remediation guidance. Issues with `aviatorRemediationGuidanceAvailable==true` have AI-generated fix recommendations. See `references/fcli-fod-output-values.md` for Aviator field definitions.

### Stable Issue Identity
Always use `instanceId` (UUID) as the canonical key for correlating the same issue across scans or releases. Do NOT use `id` (numeric) for deduplication. Note: SSC calls the equivalent field `issueInstanceId`.

### `--rel`: The Only Release Identifier for Issue and Scan Commands
`fcli fod issue list`, `fcli fod issue get`, `fcli fod issue update`, and all scan commands use **`--rel` as the sole release identifier**. There is no separate `--app` or `--release` flag on these commands — do not invent one.

`--rel` accepts a bare numeric release ID **or** a name string:
- Numeric ID (preferred when known): `--rel=1450004`
- Name string: `--rel="MyApp:main"` or `--rel="MyApp:svc:main"`

**When the user supplies a numeric release ID, use it directly as `--rel=<id>` and skip all name-resolution steps.** A numeric release ID is globally unique in FoD — no app context is required alongside it.

## Reference Files

Load these **only when needed** for the specific task at hand:

| File | When to load |
|------|--------------|
| `references/resolving-release.md` | Whenever you need to resolve an app or release name to a `--rel` identifier and it isn't already explicit |
| `references/fcli-fod-output-values.md` | Full field/enum reference for apps, releases, scans, issues; status/severity tables; Aviator semantics; available fcli modules |
| `references/fcli-fod-output-samples.md` | Concrete JSON output examples for every object type (app, release, scan, issue) |
| `references/fcli-query-output.md` | Detailed SpEL query syntax, null-safety patterns, output formats, `--store` variable chaining, server-side filtering, date utility functions |
| `references/output-formats.md` | Extended format options (`json-properties`, `--fetch=1`) and JSON processing scripting patterns |
| `references/fcli-install.md` | Full fcli installation and upgrade procedures |
| `references/mutating-operations.md` | Full safety rules for delete, create/update, access control, and REST mutations |
| `references/fod-openapi-spec.json` | **Only** when constructing `fcli fod rest` API calls — large file (~18K lines); load targeted sections only |

## Use Case Files

Read the corresponding file before proceeding with any use case. Do not generate commands from memory — these files contain required command patterns, rate-limit warnings, and safety steps specific to each workflow. If a request spans multiple use cases, load all relevant files before proceeding.

| Use Case | When to use | File |
|----------|-------------|------|
| **Portfolio Overview** | Broad view across many apps or releases — listing, counting, grouping by criticality or SDLC status, identifying failing policy at scale | `references/use-case-portfolio-overview.md` |
| **Application Context** | Detail about a specific app or release; resolving app/release names and `--rel` identifiers; comparing releases within the same app or between two named apps/releases | `references/use-case-application-context.md` |
| **Creating a Release** | Creating a new release within an existing FoD app for a branch, version, or environment | `references/use-case-create-release.md` |
| **Scan Status** | Whether scans have run, their results, in-progress or failed scans, last scan dates | `references/use-case-scan-status.md` |
| **Start Scan** | Submit a new SAST, SCA (open source), SAST+SCA, or DAST scan for a release; configure scan settings; package code; wait for results | `references/use-case-start-scan.md` |
| **Issue Investigation** | Vulnerability counts and category breakdowns for a specific app or release; filtering to specific issues; reading recommendations; triaging and bulk-updating audit status | `references/use-case-issue-investigation.md` |
| **OSS Component Analysis** | Open source component inventory and risk — finding all apps/releases using a specific component (CVE impact triage), investigating license types, reviewing OSS usage portfolio-wide or per application | `references/use-case-oss-component-analysis.md` |
| **Import Scan Results** | Importing security scan data from on-premises Fortify SAST/DAST (FPR files), third-party SAST/secret/IaC tools (SARIF), or third-party SCA/Container tools (CycloneDX SBOM); consolidating results into FoD as an ASPM single pane of glass | `references/use-case-import-scan-results.md` |
| **Advanced / REST API** | Operations not covered by named fcli commands; direct FoD REST API calls; general fallback | `references/use-case-advanced.md` |

---

## Never bypass fcli for FoD API access

All FoD API access must go through fcli — including when fcli seems slow or limited. **Never** read fcli's session or state files (`~/.fortify/fcli/state`, `$FCLI_STATE_DIR`), extract tokens or credentials from disk, or call FoD APIs directly with `curl`, `wget`, `requests`, `Invoke-WebRequest`, or any other HTTP client. The supported escape hatch for endpoints not covered by a named command is `fcli fod rest call`. If fcli is genuinely unworkable for the task, tell the user — do not work around it.

---

## Safety Rules for Mutating Operations

> **These rules are mandatory.** Before executing any `create`, `update`, `delete`, or REST mutation command, load and follow `references/mutating-operations.md`. Read-only commands (`list`, `get`, `ls`) are always safe without confirmation.
---

## Output Formats

**Prefer `-o json` for programmatic work.** Parse JSON and summarize rather than relying on table output, which omits fields and truncates values.

For extended format options and JSON processing patterns, see `references/output-formats.md`. For `--query` SpEL syntax and `--store` chaining, see `references/fcli-query-output.md`.

For available fcli fod modules and sub-commands, run `fcli fod -h` or see `references/fcli-fod-output-values.md`.
