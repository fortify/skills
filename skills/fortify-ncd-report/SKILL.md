---
name: fortify-ncd-report
description: "Handle Fortify NCD (number of contributing developers) tasks: generate reports, merge partial (team) reports, analyze and amend existing reports, answer questions about the NCD model and reporting workflow."
license: MIT
metadata:
  version: "1.0.0"
  tested-with:
    fcli: "3.23"
argument-hint: "task (explain / prepare-config / generate / merge / review), report path(s) or wildcard(s), SCM platform, Fortify platform, report date"
---

# Fortify NCD Reports

A **Number of Contributing Developers (NCD)** report counts the developers who
contribute to Fortify-scanned code, for license reconciliation. This skill drives
the `fcli license ncd-report` command family: explain reporting process, author a config file (providing assistance with repo discovery), run current or historical reports, merge team reports, and review or amend contributor data (including AI-assisted deduplication).

## Non-negotiable rules

- **Stay within the documented workflow.** Do not proactively offer side-quests or capabilities that aren't part of the step you're in. If a step in this skill calls for it, do it; if not, don't surface it as an option. Eager suggestions waste the user's attention and lead them off the remediation path.
- Always load relevant reference files for the current task, and follow them end to end. Do not skip steps or improvise. Always apply non-negotiable rules and completion gates listed in the reference files.
- Do not generate any files or directories in arbitrary locations; always ask user for working directory if current task needs to output files/directories, providing choice between any logical directories derived from chat (like next to source reports when merging), current directory, temp directory, or custom location.
- Always ask user to confirm auto-discovered or inferred values before proceeding. Never assume the user's intent.
- Infer any required task inputs from the prompt and environment first, then explicitly confirm before proceeding.
- When doing manual CSV parsing, always validate that CSV parsing approach is accurate, for example by matching CSV parsing output of `contributors.csv` against `fcli license ncd-report list-contributors -o csv` output, or by comparing CSV parsing output against manual text searches against the same CSV file. Do not assume that CSV parsing is correct without validation.
- If user input is required and a distinct set of valid options exists, present an interactive choice list (clickable options in question/answer style, not a plain numbered list) and ask the user to select one. If applicable, include a "Custom" option for free-form input and/or a "Something else" option to break out of the list. Fall back to a numbered list only if the runtime does not support interactive choice prompts.
- If a workflow step requires discovered candidates, findings, or inventory to be explained before selection, show that information in normal chat output before presenting any interactive choice prompt.
- When the same candidate set is shown in chat output and later used in a follow-up interactive choice prompt, reuse the same natural unique labels from the underlying data where practical, for example report paths, repository URLs, or author ids. Introduce synthetic numbering only if no clear natural label exists or the runtime cannot present the natural labels clearly.
- When workflow calls for presenting a choice list for a given inventory, do **not** output the inventory in normal narrative output if interactive choice prompts are available and used to display that same inventory.
- Keep question dialog content strictly to the question and selectable options; put recommendations, handoff guidance, and explanatory notes in separate normal chat output.
- Use an artifact-first output strategy for large datasets (for example long repository/contributor lists):
  - Persist full details to a review artifact file (prefer markdown; memory file if available, otherwise local file in user-approved working directory).
  - In chat, provide a concise summary and a bounded preview only (for example first 10-25 rows per relevant group), plus the artifact location.
  - Do not stream full large lists in chat unless the current step explicitly requires direct user selection over those exact items.
  - For user-selection steps, keep chat concise but ensure the user can unambiguously select items (interactive options preferred; numbered fallback if needed).

## Tasks

For reference in `Mandatory Workflow` section below only; do not pre-load or execute any of these tasks until the workflow calls for it.

| User intent | Task | Load | Requires fcli |
|-------------|------|------|---------------|
| Explain NCD reporting process, concepts, or details | **Explain** | [references/explain.md](references/explain.md) | No |
| Create a new `NcdReportConfig.yml` or update an existing one | **Prepare config** | [references/prepare-config.md](references/prepare-config.md) | Yes |
| Run a report from an existing `NcdReportConfig.yml` | **Generate report** | [references/generate-report.md](references/generate-report.md) | Yes |
| Combine several team/department reports into one domain result | **Merge** | [references/merge-reports.md](references/merge-reports.md) | Yes |
| Export, explain, validate, or correct contributors in an existing report | **Review & amend** | [references/review-amend-report.md](references/review-amend-report.md) | Yes |

## Mandatory Workflow

Complete each step before proceeding. Do not skip steps.

### Step 1 — Identify your task

Identify whether user's intent unambiguously matches one of the tasks listed in the Tasks table above. **DO NOT** attempt to clarify user's intent in this step; just check for a clear match.
- If no clear match, continue to step 2.
- If clear match, continue to step 3.

### Step 2 — Choose task

Inform user that it's recommended to run the explain task first if they haven't done so before, to explain reporting workflow and execution models. Then ask the user to choose a task by providing a choice list containing each of the tasks listed in the Tasks table above together with short descriptions. 

After a task is explicitly selected, treat it as the active task and keep it locked for this run. Do not ask for task selection again unless the user explicitly asks to switch tasks or the current task workflow explicitly redirects back to task selection. Once a task has been chosen by the user, continue to step 3.

#### Step 2 gate
- [ ] User's intent clarified and task identified (explain, prepare-config, generate, merge, or review/amend)

### Step 3 - Ensure fcli is available if required
Note the task identified in Step 1 or Step 2. Do not re-prompt for task selection after fcli availability has been confirmed (even if user deviated from the expected workflow); resume directly at Step 4 with that same task.

Check whether the task requires `fcli` (see Tasks table above):
- If yes, continue to step 3a.
- If no, skip to step 4.

#### Step 3a — Ensure fcli is available
Check that `fcli` is installed and available in the current environment, and `fcli -V` returns version 3.23.0 or above, unless user explicitly pointed you to an fcli executable to use. If not, use the instructions in [references/fcli-install.md](references/fcli-install.md) to install or upgrade `fcli`.

Once fcli availability has been confirmed, continue to step 4 using the task identified in Step 1 or Step 2; never re-prompt for task selection.

##### Step 3a gate
- [ ] fcli v3.23.0 or above, or user-requested fcli version, is installed and available

### Step 4 — Execute requested task
Load the reference file for the requested task and follow it end to end. Do not skip steps or improvise. Always apply non-negotiable rules and completion gates listed in the reference files.

#### Step 4 gate
- [ ] Task completed successfully, all steps in reference file executed
- [ ] If Explain was executed, control returned to Step 2 and next task selection was requested

### Step 5 — Confirm completion and next steps
Ask the user whether they want to:
- perform another task (return to Step 1 if explicit intent given, or Step 2 to show task selection dialog if intent is unclear),
- exit the skill (end session)

If the active task reference already instructs a next-action choice list,
do not present a second separate list. Present a single consolidated choice
list with unique numbering.