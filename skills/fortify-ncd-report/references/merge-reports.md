# Task: Merge NCD Reports

Use this workflow when you are a **consolidator** in a federated model: you have
several team/department source reports and need one reporting-domain result. To create
a new report instead, use [generate-report.md](generate-report.md). For definitions of
domain, model, and role, see [concepts.md](concepts.md).

The `merge` command reuses contributor expressions and performs cross-report
deduplication — but review the merged output rather than assuming the counts are
automatically correct.

## Non-negotiable rules

- Apply non-negotiable rules from [../SKILL.md](../SKILL.md)

## Overall process

Use this high-level flow when explaining Merge:

1. Collect source reports and verify domain, end-date alignment, and completeness.
2. Merge the source reports into one domain result.
3. Review merged summary counts and deduplication outcome.

## Step 1: Collect source reports and verify requirements

**Identify all source reports.** Ask for the exact paths, or discover them:

```bash
find . -name "*ncd*.zip"
find . -name "*ncd*" -type d
```

**Verify the reporting domain.** Confirm every source report belongs to the **same**
domain (all org-wide, or all department-X — never a mix). Ask explicitly if there's
any doubt. See [concepts.md](concepts.md).

**Verify the reporting period.** Confirm all source reports use the **same** end date.
Prefer `fcli license ncd-report get-summary -r <report>` for each source report and check `reportEndDate`.
If dates differ, warn that merged totals may mix reporting windows, and ask whether to merge anyway or rerun producers with aligned dates.

**Verify completeness.** Confirm all expected source reports were received. If some are
missing, note explicitly that merged totals are partial.

### Step 1 gate
- [ ] All source report paths identified
- [ ] All reports belong to the same reporting domain
- [ ] All reports use the same end date, or the mismatch is explicitly acknowledged
- [ ] Missing reports (if any) explicitly noted

## Step 2: Run the merge

Prefer zip output:

```bash
fcli license ncd-report merge -r team-a-report.zip,team-b-report.zip,team-c-report.zip -z full-report.zip -y
```

Use `-d full-report` for directory output only if the user explicitly wants it.

## Step 3: Review the merged result

- Prefer `fcli license ncd-report get-summary -r full-report.zip` and verify total author,
  commit, and repository counts, including dormant counts.
- Verify cross-report deduplication looks sensible — the merged set should not show
  duplicate team members. Compare against the known domain size if possible, and use
  `fcli license ncd-report list-contributors -r full-report.zip -o csv` for spot checks.

### Step 3 gate
- [ ] Merge succeeded
- [ ] Merged summary reviewed (including dormant counts)
- [ ] Cross-report deduplication looks correct

## Completion

Work is complete when:
- the goal for the reporting domain is met;
- all expected source reports were received (or gaps are explicitly acknowledged);
- the merge succeeded and the merged summary was reviewed.

Consolidator completion requires the merge checks above. Producer completion is handled
in [generate-report.md](generate-report.md) and does not require running `merge`.

**Next action.** If contributor data still needs refinement (cross-team duplicates,
bots, dormant-repo contributors), continue with
[review-amend-report.md](review-amend-report.md) — running deduplication once on the
**merged** report is more effective than per-team review.
