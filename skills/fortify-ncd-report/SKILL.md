---
name: fortify-ncd-report
description: "Handle Fortify NCD (number of contributing developers) tasks: generate reports, merge partial (team) reports, analyze and amend existing reports, answer questions about the NCD model and reporting workflow."
license: MIT
metadata:
  version: "1.0.0"
  tested-with:
    fcli: "3.21"
argument-hint: "task (generate / merge / review), SCM platform, Fortify platform, role (producer or consolidator), report date"
---

# Fortify NCD Reports

A **Number of Contributing Developers (NCD)** report counts the developers who
contribute to Fortify-scanned code, for license reconciliation. This skill drives
the `fcli license ncd-report` command family: explain reporting process, author a config file (providing assistance with repo discovery), run current or historical reports, merge team reports, and review or amend contributor data (including AI-assisted deduplication).

## Non-negotiable rules

- **Stay within the documented workflow.** Do not proactively offer side-quests or capabilities that aren't part of the step you're in. If a step in this skill calls for it, do it; if not, don't surface it as an option. Eager suggestions waste the user's attention and lead them off the remediation path.
- Always load relevant reference files for the current task, and follow them end to end. Do not skip steps or improvise. Always apply non-negotiable rules and completion gates listed in the reference files.
- Do not generate any files or directories in arbitrary locations; always ask user for working directory if current task needs to output files/directories, providing choice between current directory, temp directory, or custom location.
- Always ask user to confirm auto-discovered or inferred values before proceeding. Never assume the user's intent.
- Infer any required task inputs from the prompt and environment first, then explicitly confirm before proceeding.
- When doing manual CSV parsing, always validate that CSV parsing approach is accurate, for example by matching CSV parsing output of `contributors.csv` against `fcli license ncd-report list-contributors -o csv` output, or by comparing CSV parsing output against manual text searches against the same CSV file. Do not assume that CSV parsing is correct without validation.

## Tasks

For reference in `Mandatory Workflow` section below only; do not pre-load or execute any of these tasks until the workflow calls for it.

| User intent | Task | Load |
|-------------|------|------|
| Create a new report (single-run, or a producer's team report) | **Generate** | [references/generate-report.md](references/generate-report.md) |
| Combine several team/department reports into one domain result | **Merge** | [references/merge-reports.md](references/merge-reports.md) |
| Export, explain, validate, or correct contributors in an existing report | **Review & amend** | [references/review-amend-report.md](references/review-amend-report.md) |

## Mandatory Workflow

Complete each step before proceeding. Do not skip steps.

### Step 1 — Identify your task

Identify whether user's intent unambiguously matches one of the tasks listed in the Tasks table above. **DO NOT** attempt to clarify user's intent in this step; just check for a clear match.
- If no clear match, continue to step 2a.
- If clear match, continue to step 3.

### Step 2 — Clarify user's intent

#### Step 2a — Offer to explain NCD reporting process
If user's intent is not specified, ambiguous, or if there's any indication that user isn't sure about what task to perform, ask them whether they'd like to see an explanation of NCD reporting process first:
- If confirmed, continue to step 2b
- If denied, continue to step 2c

#### Step 2b — Explain NCD reporting process
Load [references/concepts.md](references/concepts.md) and provide user with concise, clear, non-technical explanation of NCD model, reporting domain, execution models, roles, dormant-repo edge cases, and report file inventory. Then, continue to step 2c.

#### Step 2c — Ask user to clarify intent
Ask user fewest amount of questions to clarify their intent, and provide a list of the three tasks (generate, merge, review/amend) with brief descriptions. If user still cannot clarify, offer to explain NCD reporting process again, in more detail if necessary (step 2b).

Once intent is clear, continue to step 3.

#### Step 2 gate
- [ ] User intent clarified and task identified (generate, merge, or review/validate/explain/amend)

### Step 3 — Ensure fcli is available
Check that `fcli` is installed and available in the current environment, and `fcli -V` returns version 3.21.0 or above. If not, use the instructions in [references/fcli-install.md](references/fcli-install.md) to install or upgrade `fcli`.

Continue to step 4.

#### Step 3 gate
- [ ] fcli v3.21.0 or above is installed and available

### Step 4 — Execute requested task
Load the reference file for the requested task and follow it end to end. Do not skip steps or improvise. Always apply non-negotiable rules and completion gates listed in the reference files.

#### Step 4 gate
- [ ] Task completed successfully, all steps in reference file executed

## Cross-skill dependencies

- `fortify-fod` / `fortify-ssc` — deeper FoD/SSC portfolio analysis when comparing
  repositories against Fortify inventory.
