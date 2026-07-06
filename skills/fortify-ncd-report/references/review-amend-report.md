# Task: Review and Amend an NCD Report

Use this workflow to **inspect or modify** an existing report — list contributors,
explain an unexpected count, or amend contributor status (mark duplicates, ignore
accounts, override status). For the NCD count rules and field definitions, see
[concepts.md](concepts.md).

## Non-negotiable rules

- Apply non-negotiable rules from [../SKILL.md](../SKILL.md)

## Overall process

Use this high-level flow when explaining Review & amend:

1. Identify report input(s): single merged report, multiple source reports, or wildcard pattern.
2. Confirm whether the goal is list, explain, or amend.
3. For explain/amend goals, run shared analysis to identify duplicates,
    ignored-account candidates, and dormant contributors, including a mandatory
    full-list LLM heuristic pass over contributor identities.
4. Present findings and confirm next action.
5. If amending, export contributors, produce minimal updates, apply changes, and
    spot-check.

## Step 1: Select report input and route if needed

Ask the user to provide report input first. Do not begin broad filesystem hunts before asking.

Accepted input forms:
- Single report path (zip or directory report bundle): continue in this workflow.
- Multiple report paths or wildcard pattern: treat as federated source input and route to [merge-reports.md](merge-reports.md) first.

Optional quick assist (one-time only):
- Offer one quick local discovery command to propose likely report files (for example `find . -maxdepth 3 -name "*ncd*.zip" | sort`).
- Run it only if the user asks for discovery help.
- Show findings and ask user to confirm selected report(s) before continuing.
- If no candidates are found, stop discovery and ask user to provide path(s) directly.

Routing rule:
- If user provides or confirms multiple reports (explicit list or wildcard expansion), execute merge workflow first, then resume this workflow with the merged report.

### Step 1 gate
- [ ] Report input provided or explicitly requested from user
- [ ] Any auto-discovery was limited to one quick pass and user-confirmed
- [ ] If multiple reports/wildcard were provided, merge workflow was executed first

## Step 2: Determine the review goal

If source reports still need merging in a federated model, merge first and perform
`list-contributors` / `update-contributor-status` once on the merged report. This
catches cross-team duplicates and avoids repeated review effort.

Confirm what the user wants:

| User says | Goal | Go to |
|-----------|------|-------|
| "show / export the contributors" | **List** | Step 3 |
| "explain count", "explain unexpected count", "why does it show X contributors?", "the count seems high" | **Explain** | Step 4 |
| "mark these as duplicates", "these should be ignored", "fix this" | **Amend** | Step 4 |
| "validate contributors", "validate contributing authors", "check if contributors should be duplicate/ignored" | **Amend** | Step 4 |

Validation rule: treat contributor-validation requests as amend workflow requests. Always execute shared analysis in Step 4 first, then execute Step 6a and Step 6d; execute Step 6c only if proposed changes are found.

### Step 2 gate
- [ ] Review goal confirmed (list, explain, or amend)

## Step 3: List contributors

Export the contributor list in the format that suits the user (`json` for AI/programmatic
review, `csv` for spreadsheets, `yaml` for manual reading):

```bash
fcli license ncd-report list-contributors -r ncd-report.zip -o json --to-file contributors.json
```

The export contains every contributor with fields including `authorId`,
`lastCommitDate`, `status` (`ACTIVE`/`IGNORED`), `duplicateOf`, and `overrideStatus`.
See [concepts.md](concepts.md) for the editable fields. **Never modify `authorId`.**

### Step 3 gate
- [ ] Output format chosen and exported

## Step 4: Analyze report contributors (shared for Explain and Amend)

Use this step for both **Explain** and **Amend** goals.

A higher-than-expected count can be caused by any of the following:
- Criterion 2 of the NCD definition (see [concepts.md](concepts.md)): contributors can still count if they contributed to repositories that were ever scanned by Fortify, even if those repositories are now dormant.
- Duplicate authors were not detected as duplicates by current deduplication settings.
- Bot/service/non-human accounts were not marked `IGNORED`.
- Repositories are in scope that should not be in the reporting domain.

1. Run analysis preflight validation before generating any conclusions:
     - Verify command-level report readability by running:
         - `fcli license ncd-report get-summary -r ncd-report.zip`
         - `fcli license ncd-report list-contributors -r ncd-report.zip -o csv`
         - `fcli license ncd-report list-repositories -r ncd-report.zip -o csv`
     - If manual CSV parsing is needed, validate parsing correctness with at least one independent check (for example, compare parsed contributor row counts against `fcli ... list-contributors -o csv`, and verify sampled identities can be found verbatim in source files).
     - If validation fails, stop and report **inconclusive** analysis results rather than reporting zero candidates.

2. Export contributors and build candidate lists for:
    - (A) potential duplicates
    - (B) potential users that should be ignored

    Mandatory LLM identity pass (automatic, no extra user prompt):
    - Export contributing users (only) to JSON for dedupe/ignore analysis:
      ```bash
      fcli license ncd-report list-contributors -r ncd-report.zip -q "contributionStatus=='contributing'" -o json --to-file contributors-contributing.json
      ```
    - Pass the full contributing-identity set from `contributors-contributing.json` (at minimum `authorId`, `authorName`, `authorEmail`, and contribution status fields if present) to the LLM.
    - Ask the LLM to produce heuristic candidate groups for:
        - potential duplicate contributors
        - potential bot/service/admin/non-human contributors to ignore
    - Require confidence and short evidence per candidate/group.
    - Run this LLM pass automatically for Explain and Amend goals; do not wait for an extra user request.

    Convergence rule (mandatory):
    - Produce one comprehensive first-pass result that includes all duplicate and ignore candidates split by confidence tiers (high/medium/low), with counts and evidence.
    - Do not run repeated "harder" duplicate sweeps by default after this first pass.
    - Run an additional stricter pass only if:
        - zero-result safeguard requires it, or
        - the user explicitly asks for a stricter/deeper pass.
    - If a stricter pass is run, report only delta findings versus first pass (new/removed/re-scored candidates), not a full re-narration.

    Minimum heuristic coverage:
    - (A) include exact/normalized-email checks and near-name checks with supporting evidence.
    - (B) include bot/service/admin/technical-account patterns with supporting evidence.
    - Combine LLM findings with deterministic checks; if they disagree materially, mark affected category as inconclusive and explain.

    Re-use the same review approach and assets as Step 6b:
    - [../assets/lsc-to-ucs-review-template.md](../assets/lsc-to-ucs-review-template.md)
    - [../assets/lsc-to-ucs-worked-example.md](../assets/lsc-to-ucs-worked-example.md)

3. Analyze dormant status:
     - Query dormant repositories directly from report output:
         - `fcli license ncd-report list-repositories -r ncd-report.zip -q "dormant=='true'" -o json`
     - Query dormant contributors directly from report output:
         - `fcli license ncd-report list-contributors -r ncd-report.zip -q "dormant=='true'" -o json`
     - Treat dormant values as follows:
         - `true`: contributor or repository is classified dormant by report logic.
         - `false`: at least one non-dormant contribution/repository is in scope.
         - `unknown`: insufficient data in report (common on older/legacy reports).
     - Explain that dormant contributors still count toward NCD by default per Criterion 2 (see [concepts.md](concepts.md)); do not recommend ignoring them unless there is evidence that the related repositories were never scanned by Fortify and/or associated FoD/SSC applications were deleted.

4. Run a mandatory zero-result safeguard:
    - If any of (A), (B), or (C) returns zero, run an independent second-pass check before finalizing (for example, alternate parsing method or stricter/broader heuristic set).
    - If first-pass and second-pass disagree, report the category as **inconclusive** and include the discrepancy.
    - Only report a final zero when both passes agree and parser validation succeeded.

5. Persist analysis artifacts:
    - Save candidate sets A/B/C and counts so they can be reused in Step 5 (Explain output) and Step 6 (Amend actions) without recomputing.

### Step 4 gate
- [ ] Preflight validation completed (assets present, parse validated)
- [ ] Mandatory zero-result safeguard executed where applicable
- [ ] Candidate sets A/B/C produced with evidence and counts

## Step 5: Present analysis and choose next action

1. If goal is **Explain**, produce output with three separate sections:
    - Potential duplicates: list users, with brief evidence.
    - Potential to be ignored: list users, with brief evidence.
    - Dormant contributors: list users, related dormant repositories (if available), and whether they still count by default.
    - Include per-section counts and a short note on how the count was derived.

2. Include cleanup suggestions:
    - For potential duplicates and potential ignored users, recommend applying updates through Step 6 (Step 6a to export, Step 6b to prepare minimal updates, Step 6c to apply, Step 6d to spot-check).
    - For dormant contributors, explicitly state they still count toward NCD license definition by default.
    - Ask whether dormant repositories were ever scanned by Fortify and whether associated FoD/SSC applications still exist.
    - Only if repositories were never scanned and/or associated applications were deleted, recommend tightening `repositoryIncludeExpression` for future runs (see [discovery-and-config.md](discovery-and-config.md)).
    - Only if that evidence is available and immediate correction is needed in the current report, recommend applying `IGNORED` overrides through Step 6.

3. Decision rule (mandatory):
    - If analysis finds any non-empty candidate set (A, B, or C), explicitly ask whether to proceed to Step 6 to amend now.
    - If analysis is fully empty and validated, ask whether to stop or run a stricter custom heuristic pass.
    - If user goal is **Amend**, do not ask a second time whether to run duplicate/bot analysis or whether to include both categories. Proceed directly to Step 6 using Step 4 findings, unless the user explicitly narrowed scope.
    - If duplicates are present only at medium confidence, present them as manual-review candidates in the same first-pass report instead of silently omitting them.

### Step 5 gate
- [ ] Analysis output shared (full explain output for Explain goal, concise scope summary for Amend goal)
- [ ] If A/B/C has candidates, user was explicitly offered Step 6 amend action
- [ ] Next action confirmed (amend now, stricter analysis, or stop)

## Step 6: Amend contributor status

### Step 6a: Export for amendment

```bash
fcli license ncd-report list-contributors -r ncd-report.zip -o json --to-file contributors.json
```

### Step 6b: Produce a minimal update file

Edit by hand or with AI assistance. **Emit only the rows that should change** — not the
full export. For each changed row set the editable fields (`duplicateOf`, `overrideStatus`, `overrideStatusConfidence`, `overrideStatusNotes`) and keep `authorId` exactly as exported. `overrideStatus` must be one of `contributing`, `duplicate`, or `ignored`; `duplicateOf` must point to another existing `authorId`.

Default amendment scope rule:
- If the user selected Amend and did not explicitly narrow scope, include both duplicate and ignored candidates from Step 4 in the proposed minimal update file.

Confidence handling rule:
- By default, include only high-confidence candidates in `contributors-reviewed.json` for direct apply, and present medium-confidence candidates as a separate manual-review list.
- If the user asks for aggressive coverage, allow medium-confidence candidates in the update file but keep confidence/notes explicit.

For dormant-related `ignored` overrides, include evidence in `overrideStatusNotes` that the relevant repositories were never scanned by Fortify and/or associated FoD/SSC applications were deleted.

For AI-assisted review, use [../assets/lsc-to-ucs-review-template.md](../assets/lsc-to-ucs-review-template.md)
(strict prompt + output schema). For a concrete walkthrough, see
[../assets/lsc-to-ucs-worked-example.md](../assets/lsc-to-ucs-worked-example.md).

### Step 6b-confirm: Review and confirm amendments

Before applying, display the amendments planned:

1. Count amendments in the file:
   ```bash
   jq '. | length' contributors-reviewed.json
   ```

2. By amendment type (duplicate, ignored, contributing):
   ```bash
   jq 'group_by(.overrideStatus) | map({status: .[0].overrideStatus, count: length})' contributors-reviewed.json
   ```

3. Display decision logic:
   - If amendment count ≤ 10: Display all amendments in readable table/list format with contributor names, emails, amendment type, and brief evidence/notes.
   - If amendment count > 10: Display only the count by type; offer user a choice:
     - "Review in detail" — export full amendment list for manual inspection before confirming.
     - "Show preview" — display first N (≈5–10) amendments as sample.
     - "Apply as-is" — skip detail review and proceed directly to Step 6c.

4. Mandatory user confirmation: Present an interactive choice list:
   - "Yes, apply these amendments"
   - "No, let me review the file first" (returns to hand-edit mode)
   - "Cancel and stop"

Do not proceed to Step 6c until the user confirms "Yes, apply these amendments".

### Step 6b-confirm gate
- [ ] Amendment count and breakdown displayed
- [ ] User explicitly confirmed amendments before proceeding to Step 6c

### Step 6c: Apply the amendments

```bash
fcli license ncd-report update-contributor-status -r ncd-report.zip -c contributors-reviewed.json
```

`--min-confidence` (default `0.8`) applies only rows whose `overrideStatusConfidence`
meets the threshold — useful for AI-generated updates:

```bash
fcli license ncd-report update-contributor-status -r ncd-report.zip -c contributors-reviewed.json --min-confidence 0.90
```

The command validates report checksums before applying, so amend the report in place
rather than hand-editing files inside it.

### Step 6d: Spot-check

Re-export and confirm the intended rows changed and no broad misclassification was
introduced:

```bash
fcli license ncd-report list-contributors -r ncd-report.zip -o json --to-file contributors-after.json
```

### Step 6 gate
- [ ] Contributors exported for review
- [ ] Only changed rows emitted, `authorId` preserved exactly
- [ ] Amendment count reviewed and user confirmed before applying
- [ ] Amendments applied
- [ ] Spot-check confirms the intended changes

## Completion

Work is complete when the necessary corrections are applied, the spot-check confirms
the changes are as intended, and no broad misclassification was introduced. If this was
a merged report and you are the consolidator, your work is done; otherwise return to
[generate-report.md](generate-report.md) or [merge-reports.md](merge-reports.md) if a
regenerate/re-merge is needed.
