# Task: Generate an NCD Report

Use this workflow to **create a new report** — either a single-run org-wide report,
or a producer's team/department report in a federated model. To merge existing
reports instead, use [merge-reports.md](merge-reports.md). For definitions of model,
role, domain, and the NCD count, see [concepts.md](concepts.md).

> Confirm model and role before gathering details. Don't start repo discovery until
> scope and reporting period are agreed.

## Non-negotiable rules

- For new config creation, **must** run:
  `fcli license ncd-report create-config -y -c NcdReportConfig.yml -o yaml`.
  Do **not** hand-scaffold a fresh `NcdReportConfig.yml` from memory.
- If an existing config is being adjusted, edit that file in place; do not replace it with a brand-new hand-written scaffold.

## Step 1: Confirm execution model and role

Infer the execution model (single-run vs federated) and your role (producer,
consolidator, or both) from the user's request and environment, then confirm.

- Use the role inference cheat sheet in [concepts.md](concepts.md).
- Check the environment:
  - `find . -name "NcdReportConfig.yml"` — existing config implies a rerun.
  - `fcli fod session ls --query "expired=='No'"` / `fcli ssc session ls --query "expired=='No'"`
    — whether a Fortify platform is available for comparison.

If ambiguous, ask in order:
1. A report for the entire organization, or for a team/department? (→ model)
2. Will other teams also generate reports to be merged, or are you the only producer?
   (→ role)

### Step 1 gate
- [ ] Execution model confirmed (single-run or federated)
- [ ] Role confirmed (producer, consolidator, or both)

## Step 2: Establish scope and reporting period

**Reporting domain.** Confirm which repositories this report covers (org-wide, or a
specific department). See [concepts.md](concepts.md).

**Reporting period.** Confirm the 90-day window's end date:
- For a **federated producer**, recommend the **last completed quarter**
  (e.g. `2026-03-31`) so reports merge with aligned dates.
- **Last completed month** for faster iteration.
- **Current cycle** (today) for a snapshot now.
- A **specific historical date** for a particular 90-day window.

Historical reports use `--end-date yyyy-MM-dd` as the inclusive end of the window. In
federated mode, all producers must use the **same** end date.

**Fortify platform (if comparing).** If the user named FoD or SSC, use it. Otherwise
check active sessions (above): only FoD active → FoD; only SSC → SSC; both → ask;
neither → offer to skip Fortify comparison.

**Existing config.** If the user already has an `NcdReportConfig.yml`, ask whether it
can be reused as-is (skip to Step 4) or needs adjustment (use it as the starting point
in Step 3).

### Step 2 gate
- [ ] Reporting domain confirmed
- [ ] Reporting period (end date) confirmed
- [ ] Fortify platform identified or comparison intentionally skipped
- [ ] Existing config status decided (reuse as-is / adjust / create new)

## Step 3: Discover repositories and author the config

Skip if reusing a config as-is. Otherwise load
[discovery-and-config.md](discovery-and-config.md) and follow it to discover the
in-scope repositories and produce or adjust `NcdReportConfig.yml`. Return here once
the discovery + config gate in that file passes.

For any new config path, do not hand-author `NcdReportConfig.yml` first. The workflow
must execute `create-config` and then edit the generated file.

## Step 4: Run the report

### Step 4 prerequisites

Before running `fcli license ncd-report create`, confirm:

- Config is complete for each configured SCM source (selectors, include logic,
  and contributor expressions).
- Repository selection is intentional: include only repos in the reporting domain
  that should count for Fortify licensing, unless the user explicitly requests a
  broader scope.
- Tokens referenced by `#env("...")` in `NcdReportConfig.yml` are defined in the
  environment.
- Authentication path is explicit: use authenticated SCM access by default; use
  unauthenticated access only if the user explicitly asks for it.

### Step 4 gate
- [ ] Config completeness verified
- [ ] Repository scope validated with user
- [ ] Referenced token environment variables are set
- [ ] Authenticated access path confirmed (or explicit user-approved exception)

Prefer zip output. Run the report:

```bash
# Current cycle
fcli license ncd-report create -y -c NcdReportConfig.yml -z ncd-report.zip

# Historical (example: Q1 2026 ending 2026-03-31)
fcli license ncd-report create -y -c NcdReportConfig.yml -z ncd-report-2026Q1.zip --end-date 2026-03-31
```

Use `-d <dir>` instead of `-z <file>.zip` only if the user explicitly wants directory
output.

## Step 5: Validate the output

Confirm the report contains the expected files (see the inventory in
[concepts.md](concepts.md)): `summary.txt`, `contributors.csv`, `checksums.sha256`,
`report-config.yaml`, and the four `details/*.csv` files.

Review and validate:
- Check author, commit, and repository counts in `summary.txt` for obvious scope
  mistakes.
- Extract `details/repositories.csv` and compare the listed repos against your known
  list.
  - Missing repos → check `report.log` for auth failures, rate limits, or access
    errors.
  - Unexpected repos → tighten `repositoryIncludeExpression` and rerun.
- In federated mode, confirm the report uses the agreed end date.

If validation exposes problems, return to Step 2 or Step 3, adjust, and regenerate.

### Step 5 gate
- [ ] Report generated successfully
- [ ] Expected files present
- [ ] Summary reviewed for obvious scope/count problems
- [ ] Discovered repositories validated against the known list

## Completion

Work is complete when:
- model and role are documented;
- repo inclusion logic is reproducible and credentials are externalized;
- report files and summary are internally consistent;
- **single-run** → the report is final;
- **federated producer** → the report is handed off to the consolidator (who proceeds
  with [merge-reports.md](merge-reports.md)).

**Next action.** If you want to inspect or correct contributors, continue with
[review-amend-report.md](review-amend-report.md).
