---
name: fortify-ncd-report
description: "**WORKFLOW SKILL** — Generate Fortify Number of Contributing Developers (NCD) reports using `fcli license ncd-report`. USE FOR: 'generate NCD report', 'ncd-report', 'number of contributing developers', Fortify license reconciliation/reporting, team NCD report merge/consolidation, list contributors (`lsc`), update contributor status (`ucs`), and prompts like 'workspace contains all team projects, generate NCD report'. NOT FOR: running SAST/DAST/SCA scans, triaging FoD/SSC vulnerabilities, or generic security code review."
license: MIT
metadata:
  version: "1.0.0"
  tested-with:
    fcli: "3.21"
argument-hint: "task (generate / merge / review), SCM platform, Fortify platform, role (producer or consolidator), current vs historical"
---

# Fortify NCD Reports

A **Number of Contributing Developers (NCD)** report counts the developers who
contribute to Fortify-scanned code, for license reconciliation. This skill drives
the `fcli license ncd-report` command family: discover repositories, author a
config, run current or historical reports, merge team reports, and review or amend
contributor data (including AI-assisted deduplication).

> **Stay within the chosen task.** Do not jump ahead to discovery, merge, deduplication, or Fortify inventory comparison before the current step establishes they are needed. Eager side-quests waste the user's attention and lead them off the workflow.

## Non-negotiable rules

- `fcli` executable must be accessible, for example on PATH. If it is missing or version below 3.21, activate [references/fcli-install.md](references/fcli-install.md).
- Always load relevant reference files for the current task, and follow them end to end. Do not skip steps or improvise. Always apply non-negotiable rules and completion gates listed in the reference files.
- Do not generate any files or directories in arbitrary locations. Always use a temp directory and inform the user where to find them. Do not write to the current working directory unless explicitly requested by the user.
- Always ask user to confirm auto-discovered or inferred values before proceeding. Never assume the user's intent.
- Infer model/role/platform/scope from the prompt and environment first, then
  explicitly confirm before proceeding.

## Activation and disambiguation

Trigger this skill when the prompt includes NCD intent, for example:
- generate NCD report
- `fcli license ncd-report`
- contributors for Fortify licensing / contributor counting / license reconciliation
- merge team NCD reports
- list contributors (`lsc`) or update contributor status (`ucs`)
- validate contributors / validate contributing authors / validate NCD contributor accuracy
- check whether contributing authors should be duplicate or ignored

Do not trigger this skill for:
- full FoD/SSC scan execution (use `fortify-fod` or `fortify-ssc`)
- fixing vulnerabilities or reviewing code changes (`fortify-remediate` or
  `fortify-change-review`)
- creating new FoD/SSC applications (`fortify-create-app`)

## Key concepts

Read [references/concepts.md](references/concepts.md) for full definitions and edge
cases. In brief:

- **NCD** — any developer who either (1) committed in the trailing **90 days**, or
  (2) made the most recent commit to a repo that has had **no commits in 90 days**
  (dormant repos still count their last committer). Criterion 2 is the usual cause
  of "the count is higher than I expected".
- **Reporting domain** — all repositories covered by **one** NCD license agreement
  (usually org-wide; per-department if licenses are split). Each domain yields
  exactly **one** final report.
- **Execution model** — *single-run* (one config covers the whole domain) or
  *federated* (teams generate separate reports that are later merged).
- **Role** — *producer* (generates a source report), *consolidator* (merges source
  reports), or *both*.

## Prerequisites

- SCM access token(s) for GitHub, GitLab, and/or Azure DevOps (externalized via
  `#env(...)`, never embedded).
- *Optional:* an active FoD or SSC session if repositories will be compared against
  Fortify inventory. Activate the `fortify-fod` or `fortify-ssc` skill for deeper
  portfolio work.

## Key commands

| Command | Purpose |
|---------|---------|
| `fcli license ncd-report create-config -y -c NcdReportConfig.yml -o yaml` | Generate a config scaffold |
| `fcli license ncd-report create -y -c NcdReportConfig.yml -z ncd-report.zip [--end-date yyyy-MM-dd]` | Run a current or historical report |
| `fcli license ncd-report merge -r a.zip,b.zip -z full-report.zip -y` | Merge federated source reports |
| `fcli license ncd-report list-contributors -r ncd-report.zip -o json --to-file contributors.json` | Export contributors (alias `lsc`) |
| `fcli license ncd-report update-contributor-status -r ncd-report.zip -c updates.json [--min-confidence 0.90]` | Apply duplicate/ignore/override amendments (alias `ucs`) |

Defaults: `--end-date` defaults to today (current 90-day window); `--min-confidence`
defaults to `0.8`. Prefer `-z` (zip) output over `-d` (directory) unless the user
asks otherwise.

## Choose your task

Identify the user's intent, confirm it, then load the matching reference and follow
it end to end (each contains ordered steps with completion gates).

| User intent | Task | Load |
|-------------|------|------|
| Create a new report (single-run, or a producer's team report) | **Generate** | [references/generate-report.md](references/generate-report.md) |
| Combine several team/department reports into one domain result | **Merge** | [references/merge-reports.md](references/merge-reports.md) |
| Export, explain, validate, or correct contributors in an existing report | **Review & amend** | [references/review-amend-report.md](references/review-amend-report.md) |

Validation routing rule: requests to "validate" contributors or contributor status must route to **Review & amend** and run the amend workflow checks in Step 4 (including producing and applying updates when needed).

If intent is ambiguous, ask the fewest questions to pick a row:
1. Generate a new report, merge existing ones, or review one you already have?
2. (If generate) Whole organization in one run, or one team's report among several?

## Reference files

Load only what the current task needs:

| File | When to load |
|------|--------------|
| [references/concepts.md](references/concepts.md) | NCD definition, reporting domain, execution models, roles, dormant-repo edge cases, report file inventory |
| [references/discovery-and-config.md](references/discovery-and-config.md) | Discover which repos belong in scope and author/customize `NcdReportConfig.yml` (sources, include expressions, credentials, contributor expressions) |
| [references/generate-report.md](references/generate-report.md) | End-to-end workflow to produce a report |
| [references/merge-reports.md](references/merge-reports.md) | Consolidator workflow to merge source reports |
| [references/review-amend-report.md](references/review-amend-report.md) | List contributors, explain unexpected counts, amend/deduplicate |

Assets used by the review/amend task:
[assets/lsc-to-ucs-review-template.md](assets/lsc-to-ucs-review-template.md) (AI review
prompt + output schema) and
[assets/lsc-to-ucs-worked-example.md](assets/lsc-to-ucs-worked-example.md) (concrete
example).

## Cross-skill dependencies

- `fcli-common` — fcli installation, session setup, shared fcli/query patterns.
- `fortify-fod` / `fortify-ssc` — deeper FoD/SSC portfolio analysis when comparing
  repositories against Fortify inventory.
