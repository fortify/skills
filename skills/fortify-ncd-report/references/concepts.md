# NCD Concepts and Definitions

Load this when you need precise definitions of the Named Contributing Developer
count, the reporting domain, execution models, roles, or when explaining why a
count looks higher than expected.

## Named Contributing Developer (NCD)

Per Fortify licensing:

> A Named Contributing Developer (NCD) is any individual that has: **(1) committed
> code to the applications to be assessed during the 90 days prior to assessment**,
> or **(2) the most recent individual who has made changes to the application code
> if no code commits have been made in the past 90 days.**

In practice this is two criteria:

- **Criterion 1 — active contributors.** Anyone with at least one commit in the
  trailing 90-day window is counted.
- **Criterion 2 — last committer on dormant repos.** If a repository has had **no
  commits in the past 90 days**, the person with the most recent commit is *still*
  counted, even if that commit is much older. This commonly catches:
  - contributors on stale feature branches,
  - maintainers of archived or dormant projects,
  - occasional contributors to long-stable repositories.

**Why counts look high.** Criterion 2 is the usual reason a count is higher than
expected. When questioning a result, check whether:
1. the reporting domain includes repositories that should be excluded (archived
   projects, examples, test harnesses);
2. contributors should be marked `IGNORED` for repos that shouldn't count toward the
   license;
3. stale branches are pulling in dormant last-committers (Criterion 2).

## Reporting domain

**Reporting domain** = all repositories covered by **one** NCD license agreement.

- Default: the entire organization (every repository scanned with Fortify).
- Split pools: if NCD licenses are purchased per business unit, each unit defines its
  own reporting domain.
- Each reporting domain produces exactly **one** final report — either directly
  (single-run) or by merging team reports (federated).

When merging, every source report (and every repository in it) must belong to the
**same** reporting domain. Never mix an org-wide report with a department report.

## Execution models

- **Single-run model** — one `create` config processes all repositories in the
  domain in a single run. Prefer when repository metadata is clean and centrally
  managed, one automation identity can read the full SCM scope, and a single
  repeatable scheduled run is wanted.
- **Federated model** — teams or departments generate separate source reports that
  are later merged into one domain result. Prefer when team/department boundaries
  are distinct and access or ownership varies.

### Execution model decision rules

Prefer **federated** when:
- different teams own different SCM scopes,
- token access differs by team,
- business review happens per team before a domain-level rollup.

Prefer **single-run** when:
- repository metadata is clean and centrally managed,
- one automation identity can read the full SCM scope,
- a single repeatable scheduled run is desired.

Determine the execution model **first**, then assign roles.

## Roles

- **Producer** — generates a team/department source report and hands it off to the
  consolidator. Producer completion does **not** require running `merge`.
- **Consolidator** — collects all source reports for the domain, merges them, and
  optionally performs post-merge contributor cleanup.
- **Both** — one actor performs both responsibilities (typical in single-run mode);
  keep producer output and consolidation steps logically separate.

### Role inference cheat sheet

| User says | Model | Role |
|-----------|-------|------|
| "organization-wide report", "one report for all repos", "company report" | Single-run | Both |
| "team report", "my team's report", "department report" | Federated | Producer |
| "merge reports", "consolidate team reports", "combine reports" | Federated | Consolidator |
| "generate a report, then merge" | Federated | Both |

Always summarize your inferred model and role and confirm with the user before
proceeding.

## Report output inventory

A generated or merged report (zip or directory) contains:

- `summary.txt` — author, commit, and repository totals
- `contributors.csv` — the contributor list
- `checksums.sha256` — integrity checksums (validated by `update-contributor-status`)
- `report-config.yaml` — the effective config, including `endDate`
- `details/repositories.csv`
- `details/commits-by-branch.csv`
- `details/commits-by-repository.csv`
- `details/contributors-by-repository.csv`

Use `details/repositories.csv` to spot dormant repos (old last-commit dates) and
`details/contributors-by-repository.csv` to find contributors who appear only on
those repos.

## Contributor record fields

`list-contributors` exposes per-contributor fields. Editable for amendments:

- `duplicateOf` — set to another existing `authorId` to mark a duplicate
- `overrideStatus` — `contributing`, `duplicate`, or `ignored` (forces status)
- `overrideStatusConfidence` — `0.0`–`1.0`, used with `--min-confidence` on apply
- `overrideStatusNotes` — human-readable reason for the override

**Never modify `authorId`** or other system fields. `authorId` is the stable key
that ties an update row back to a contributor.
