# Task: Generate an NCD Report

Use this workflow to **run `fcli license ncd-report create`** from an existing,
complete `NcdReportConfig.yml`. To create or update the config first, use
[prepare-config.md](prepare-config.md). To merge existing reports instead, use
[merge-reports.md](merge-reports.md). For definitions of model, role, domain, and
the NCD count, see [concepts.md](concepts.md).

## Non-negotiable rules

- Apply non-negotiable rules from [../SKILL.md](../SKILL.md)
- For `fcli license ncd-report create`, `fcli license ncd-report validate-sources`, and direct SCM REST calls, execute through mandatory [run-cmd-with-scm-auth.md](run-cmd-with-scm-auth.md) workflow.

## Mandatory Workflow

Complete each step before proceeding. Do not skip steps.

### Step 1: Locate and confirm config

Ask or auto-detect the config to use:
- Run `find . -name "NcdReportConfig.yml"` to locate candidate files.
- If exactly one is found, present the path and ask the user to confirm or specify a different path.
- If multiple are found, ask the user to select from the list or specify a different path.
- If none are found, ask the user to provide the config path, or offer to switch to
  the **Prepare config** task first.

#### Step 1 gate
- [ ] Config file path confirmed

### Step 2: Confirm reporting period end date

Confirm the 90-day window's end date. Offer:
- **Last completed quarter** (e.g. `2026-03-31`) — recommended for federated
  producers so reports merge with aligned dates.
- **Last completed month** — for faster iteration.
- **Current cycle** (today) — for a snapshot now.
- **Specific historical date** — user supplies `yyyy-MM-dd`.

Historical reports use `--end-date yyyy-MM-dd` as the inclusive end of the window.
In **federated mode**, all producers should use the **same** end date.

#### Step 2 gate
- [ ] Reporting period (end date) confirmed

### Step 3: Confirm report output location

Confirm where to write the report output. Offer:
- `<config file dir>/ncd-report-<enddate>.zip`.
- `<cwd>/ncd-report-<enddate>.zip`.
- **Alternative output location**.

For **Alternative output location**, resolve as follows:
- If the provided path ends with `.zip`, use it as-is for zip output (`-z`).
- If the provided path is an existing directory, check contents:
  - If existing report dir (`summary.txt`, `contributors.csv`, and `checksums.sha256` all present), use as-is for directory output (`-d`) with auto-replace (`-y`).
  - Otherwise, use `<dir>/ncd-report-<enddate>.zip`
- If the provided path does not exist, use it as-is for directory output (`-d`).

#### Step 3 gate
- [ ] Output location confirmed

### Step 4: Pre-flight checks

Before running `fcli license ncd-report create`, confirm:

- Config is complete for each configured SCM source (selectors, include logic,
  and contributor expressions).
- Repository selection is intentional: include only repos in the reporting domain
  that should count for Fortify licensing, unless the user explicitly requests a
  broader scope.
- Authentication path is explicit: use authenticated SCM access by default; use
  unauthenticated access only if the user explicitly asks for it.

#### Step 4 gate
- [ ] Config completeness verified
- [ ] Repository scope validated with user
- [ ] Authenticated access path confirmed (or explicit user-approved exception)

### Step 5: Run the report

Prefer zip output. Run the report using config file, end date, and output location identified in previous steps:

```bash
# Current cycle
fcli license ncd-report create -y -c NcdReportConfig.yml -z ncd-report.zip

# Historical (example: Q1 2026 ending 2026-03-31)
fcli license ncd-report create -y -c NcdReportConfig.yml -z ncd-report-2026Q1.zip --end-date 2026-03-31
```

Use `-d <dir>` instead of `-z <file>.zip` only if the user explicitly wants directory
output.

#### Step 5 gate
- [ ] Report generation completed without errors

### Step 6: Validate the output

Confirm the report contains the expected files: `summary.txt`, `contributors.csv`, `checksums.sha256`, and `details/*.csv`.

Review and validate:
- Prefer command-level summary review:
  - `fcli license ncd-report get-summary -r ncd-report.zip`
  - Verify report end date and count sections, including dormant counts in `repositoryCounts` and `authorCounts`.
- Use repository list output for scope validation:
  - `fcli license ncd-report list-repositories -r ncd-report.zip -o csv`
  - Compare listed repositories against the known list.
  - Missing repos → check `report.log` for auth failures, rate limits, or access
    errors.
  - Unexpected repos → tighten `repositoryIncludeExpression` in the config and rerun
    (use **Prepare config** task to update the config).

If validation exposes problems, fix the config using the **Prepare config** task, then
re-run this workflow.

#### Step 6 gate
- [ ] Report generated successfully
- [ ] Expected files present
- [ ] Summary reviewed for obvious scope/count problems (including dormant counts)
- [ ] Discovered repositories validated against the known list

### Completion

Work is complete when:
- repo inclusion logic is reproducible and credentials are externalized;
- report files and summary are internally consistent;
- **single-run** → the report is final;
- **federated producer** → the report is handed off to the consolidator (who proceeds
  with [merge-reports.md](merge-reports.md)).

**Next action.** If you want to inspect or correct contributors, continue with
[review-amend-report.md](review-amend-report.md).
