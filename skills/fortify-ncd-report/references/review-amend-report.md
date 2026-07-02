# Task: Review and Amend an NCD Report

Use this workflow to **inspect or modify** an existing report — list contributors,
explain an unexpected count, or amend contributor status (mark duplicates, ignore
accounts, override status). For the NCD count rules and field definitions, see
[concepts.md](concepts.md).

## Step 1: Determine the review goal

If source reports still need merging in a federated model, merge first and perform
`list-contributors` / `update-contributor-status` once on the merged report. This
catches cross-team duplicates and avoids repeated review effort.

Confirm what the user wants:

| User says | Goal | Go to |
|-----------|------|-------|
| "show / export the contributors" | **List** | Step 2 |
| "explain count", "explain unexpected count", "why does it show X contributors?", "the count seems high" | **Explain** | Step 3 |
| "mark these as duplicates", "these should be ignored", "fix this" | **Amend** | Step 4 |
| "validate contributors", "validate contributing authors", "check if contributors should be duplicate/ignored" | **Amend** | Step 4 |

Validation rule: treat contributor-validation requests as amend workflow requests. Always execute Step 4a and Step 4d; execute Step 4c only if proposed changes are found.

### Step 1 gate
- [ ] Review goal confirmed (list, explain, or amend)

## Step 2: List contributors

Export the contributor list in the format that suits the user (`json` for AI/programmatic
review, `csv` for spreadsheets, `yaml` for manual reading):

```bash
fcli license ncd-report list-contributors -r ncd-report.zip -o json --to-file contributors.json
```

The export contains every contributor with fields including `authorId`,
`lastCommitDate`, `status` (`ACTIVE`/`IGNORED`), `duplicateOf`, and `overrideStatus`.
See [concepts.md](concepts.md) for the editable fields. **Never modify `authorId`.**

### Step 2 gate
- [ ] Output format chosen and exported

## Step 3: Explain an unexpected count

Use this step when the user asks to **explain count** or **explain unexpected count**.

A higher-than-expected count can be caused by any of the following:
- Criterion 2 of the NCD definition (see [concepts.md](concepts.md)): contributors can still count if they contributed to repositories that were ever scanned by Fortify, even if those repositories are now dormant.
- Duplicate authors were not detected as duplicates by current deduplication settings.
- Bot/service/non-human accounts were not marked `IGNORED`.
- Repositories are in scope that should not be in the reporting domain.

1. Export contributors and build candidate lists for:
    - (A) potential duplicates
    - (B) potential users that should be ignored

    Re-use the same review approach and assets as Step 4b:
    - [../assets/lsc-to-ucs-review-template.md](../assets/lsc-to-ucs-review-template.md)
    - [../assets/lsc-to-ucs-worked-example.md](../assets/lsc-to-ucs-worked-example.md)

2. Analyze dormant repositories:
    - Read `details/repositories.csv` and identify dormant repositories (default heuristic: no commits in 90+ days).
    - Read `details/contributors-by-repository.csv` and collect (C) contributors that appear only on dormant repositories.
    - Explain that Criterion 2 (see [concepts.md](concepts.md)) may still include those contributors if the repositories were ever scanned with Fortify.

3. Produce the explain output with three separate sections:
    - Potential duplicates: list users, with brief evidence.
    - Potential to be ignored: list users, with brief evidence.
    - Dormant-only contributors: list users plus the dormant repositories they are tied to.

4. Include cleanup suggestions:
    - For potential duplicates and potential ignored users, recommend applying updates through Step 4 (Step 4a to export, Step 4b to prepare minimal updates, Step 4c to apply, Step 4d to spot-check).
    - For dormant-only contributors, ask whether those repositories have ever been scanned by Fortify.
    - If repositories were never scanned and this is not a merged upstream scope, recommend tightening `repositoryIncludeExpression` for future runs (see [discovery-and-config.md](discovery-and-config.md)).
    - If immediate correction is needed in the current report, recommend applying `IGNORED` overrides through Step 4.

### Step 3 gate
- [ ] Explain request handled (`explain count` and `explain unexpected count`)
- [ ] Output includes users grouped by duplicate/ignored/dormant
- [ ] Cleanup suggestions provided (Step 4 and/or config scope changes)

## Step 4: Amend contributor status

### Step 4a: Export for amendment

```bash
fcli license ncd-report list-contributors -r ncd-report.zip -o json --to-file contributors.json
```

### Step 4b: Produce a minimal update file

Edit by hand or with AI assistance. **Emit only the rows that should change** — not the
full export. For each changed row set the editable fields (`duplicateOf`, `overrideStatus`, `overrideStatusConfidence`, `overrideStatusNotes`) and keep `authorId` exactly as exported. `overrideStatus` must be one of `contributing`, `duplicate`, or `ignored`; `duplicateOf` must point to another existing `authorId`.

For AI-assisted review, use [../assets/lsc-to-ucs-review-template.md](../assets/lsc-to-ucs-review-template.md)
(strict prompt + output schema). For a concrete walkthrough, see
[../assets/lsc-to-ucs-worked-example.md](../assets/lsc-to-ucs-worked-example.md).

### Step 4c: Apply the amendments

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

### Step 4d: Spot-check

Re-export and confirm the intended rows changed and no broad misclassification was
introduced:

```bash
fcli license ncd-report list-contributors -r ncd-report.zip -o json --to-file contributors-after.json
```

### Step 4 gate
- [ ] Contributors exported for review
- [ ] Only changed rows emitted, `authorId` preserved exactly
- [ ] Amendments applied
- [ ] Spot-check confirms the intended changes

## Completion

Work is complete when the necessary corrections are applied, the spot-check confirms
the changes are as intended, and no broad misclassification was introduced. If this was
a merged report and you are the consolidator, your work is done; otherwise return to
[generate-report.md](generate-report.md) or [merge-reports.md](merge-reports.md) if a
regenerate/re-merge is needed.
