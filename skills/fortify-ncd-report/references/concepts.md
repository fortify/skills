# NCD Concepts and Definitions

## Skill Overview

The `fortify-ncd-report` skill provides assistance on various Fortify NCD reporting tasks:
- Configuring and generating an NCD report
- Merging team/department reports into a federated report
- Reviewing and amending NCD reports

Generating an NCD report requires access to the SCM system(s) that host the repositories that are being scanned with Fortify. For each of these repositories, the commit history is used to identify the contributing developers. The skill can assist with drafting the configuration file used for generating the report, including repository discovery and setting up contributor deduplication and ignore rules.

Once report(s) have been generated, they can be merged into a federated report (see 'Execution models' below) and amended if needed, for example for further (manual or AI-assisted) cleanup of duplicate contributors.

## Named Contributing Developer (NCD)

Per Fortify licensing:

> A Named Contributing Developer (NCD) is any individual that has: **(1) committed code to the applications to be assessed during the 90 days prior to assessment**, or **(2) the most recent individual who has made changes to the application code if no code commits have been made in the past 90 days.**

Note that criterion 2 is often overlooked; if a repository has had **no commits in the past 90 days**, the person with the most recent commit is *still* counted, even if that commit is much older.

## Reporting domain

**Reporting domain** = all repositories covered by **one** NCD license agreement.

- Default: the entire organization (every repository scanned with Fortify).
- Split pools: if NCD licenses are purchased per business unit, each unit defines its own reporting domain.
- Each reporting domain produces exactly **one** final report — either directly (single-run) or by merging team reports (federated).

When merging, every source report (and every repository in it) must belong to the **same** reporting domain / NCD license agreement.

## Execution models

- **Single-run model** — one `create` config processes all repositories in the reporting domain in a single run. Prefer when repository metadata is clean and centrally managed, one automation identity can read the full SCM scope, and a single
  repeatable scheduled run is wanted.
- **Federated model** — teams or departments generate separate source reports that are later merged into one domain result. Prefer when team/department boundaries are distinct and access or ownership varies.

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

- **Producer** — generates a team/department source report and hands it off to the consolidator. Producer completion does **not** require running `merge`.
- **Consolidator** — collects all source reports for the domain, merges them, and optionally performs post-merge contributor cleanup.
- **Both** — one actor performs both responsibilities (typical in single-run mode); keep producer output and consolidation steps logically separate.

## High-level workflow patterns

### Single-run model (one domain-level run)

1. Generate report config and define SCM repositories to load commit data from.
2. Generate the report by loading commit data from SCM.
3. Review and amend the generated report (duplicates, ignored bot/service users, confidence-based overrides) as needed.

### Federated model (team reports then consolidation)

1. Each producer generates report config for their team/department SCM scope.
2. Each producer generates a source report by loading commit data from SCM.
3. Consolidator merges team/department source reports into one domain result.
4. Review and amend the merged report (cross-team duplicates, ignored bot/service users, confidence-based overrides) as needed.

## Report Repository Selection

SCM repositories are the single source of truth for NCD report generation, hence selecting the right repositories to be included in a report is vital for proper reporting.

Repository selection is done based on SCM sources and repository selection criteria in the configuration file that is passed to the `fcli license ncd-report create` command. The configuration file may list multiple sources, covering multiple SCM systems and/or multiple SCM-specific boundaries like organizations or groups, all of which will be combined in a single report.

For the single-run execution model, the configuration must cover all Fortify-scanned repositories across the reporting domain. For the federated execution model, all Fortify-scanned repositories for an individual reporting unit like team or department must be covered by the configuration.

The exact configuration format differs across supported SCM sources and is authoritatively documented in the output of the `fcli license ncd-report create-config` command; defer to that scaffold for precise syntax. Conceptually, sources share a few characteristics:
- Most sources require the SCM organizations/groups to be processed to be explicitly named in the configuration file.
- All sources support a `repositoryIncludeExpression`: a Spring Expression Language expression evaluated against repository properties.
- Some sources allow explicitly listing repository names; where not supported, the same effect can be achieved through `repositoryIncludeExpression`.

Based on the above, there are two primary repository selection strategies:
1. Auto-selection based on repository structure or metadata, for example (a combination of):
    - SCM-specific boundaries, i.e., selecting all repositories in an organization-wide or team/department-specific SCM group/organization 
    - Repository topics/labels/tags, i.e., selecting all repositories with 'scanned-with-fortify' tag
    - Naming conventions, i.e., selecting all repositories matching `<departmentId>-.*`
2. Manual selection, explicitly listing the repositories to be included in the report.

If needed, these strategies may be combined, for example auto-selecting all repositories from a GitLab group, amended with some specific repositories hosted on another SCM system.

Approach #1 (auto-selection) requires proper house-keeping on the SCM side — consistent boundaries, topics/labels, or naming conventions — but requires little to no maintenance effort on the NCD reporting side: future report runs automatically pick up all relevant repositories, including newly added ones. Its main risk is that the selection silently drifts from the intended scope when SCM house-keeping is incomplete (e.g. a Fortify-scanned repository that is missing the expected topic).

Approach #2 (manual selection) does not depend on SCM house-keeping and gives precise, explicit control over scope, but requires ongoing manual maintenance of the repository list as repositories are added, removed, or renamed.

The 'Repository Discovery Techniques' section below describes techniques that support both approaches: initial discovery of the repositories to list under approach #2, and validation under either approach — confirming that an auto-selection expression (approach #1) or an explicit list (approach #2) resolves to exactly the intended set of Fortify-scanned repositories.

## Repository Discovery Techniques

The selection approaches above describe *how* repositories are encoded in the config.
This section describes techniques for *figuring out* which repositories belong in
scope — used to build a manual list (approach 2), or to discover the right boundaries,
selectors, or naming patterns for auto-selection (approach 1), or to validate and
refine either approach against actual SCM/Fortify data. Techniques are frequently
combined (e.g., discover candidate repositories from Fortify metadata, then confirm
them via SCM-native queries).

**Technique 1: Fortify platform metadata (FoD or SSC)**
- Derive the in-scope repository set from the applications, releases, or application versions already registered in FoD or SSC, using Fortify-stored attributes such as repository URLs or application names.
- *Pro:* Directly tied to what Fortify is actually scanning; avoids including repositories that are not being assessed.
- *Con:* Only works if Fortify metadata contains accurate SCM links; attribute quality varies.

**Technique 2: SCM-native exploration**
- Query the SCM system (GitHub, GitLab, Azure DevOps, etc.) directly to enumerate organizations/groups/projects and discover repositories via structure, topics, labels, or naming patterns.
- *Pro:* Comprehensive coverage; reveals the boundaries and metadata that auto-selection selectors can target; can verify what repositories are actually being scanned by Fortify from pipeline/workflow files
- *Con:* Requires authenticated SCM access; depends on SCM metadata quality being consistent and well-maintained.

**Technique 3: Heuristic / sample-based**
- Start from a small set of known in-scope repositories and identify common patterns to build a broader selector expression.
- *Pro:* Useful when metadata is inconsistent or organizational structure is unclear; helps surface hidden patterns.
- *Con:* Inferred scope may include false positives or miss edge cases; requires validation against actual SCM data.

## Report output inventory

A generated or merged report (zip or directory) contains:

- `summary.txt` — author, commit, repository totals, report end date
- `contributors.csv` — the contributor list
- `checksums.sha256` — integrity checksums (validated by `update-contributor-status`)
- `report-config.yaml` — the effective config
- `details/repositories.csv`
- `details/commits-by-branch.csv`
- `details/commits-by-repository.csv`
- `details/contributors-by-repository.csv`

Prefer command-level access over direct file parsing when possible:

- `fcli license ncd-report get-summary -r <report>` for report-level counts and dates
- `fcli license ncd-report list-repositories -r <report>` for repository-level status and dormant fields
- `fcli license ncd-report list-contributors -r <report>` for contributor-level status and dormant fields

These commands are required by this skill workflow.

## Dormant classification notes

- `repositoryCounts.dormant` and `authorCount.dormant` can differ in summary output.
- `authorCount.dormant` may be lower than dormant repository count when a contributor appears in multiple repositories and is non-dormant in at least one repository.
- In list outputs, dormant can be `true`, `false`, or `unknown`.
  - `unknown` typically indicates insufficient data in legacy report formats.
- Dormant contributors still count toward NCD by default per criterion 2, unless evidence shows associated repositories were never scanned by Fortify and/or associated FoD/SSC applications were deleted.

## Contributor record fields

`list-contributors` exposes per-contributor fields. Editable for amendments:

- `duplicateOf` — set to another existing `authorId` to mark a duplicate
- `overrideStatus` — `contributing`, `duplicate`, or `ignored` (forces status)
- `overrideStatusConfidence` — `0.0`–`1.0`, used with `--min-confidence` on apply
- `overrideStatusNotes` — human-readable reason for the override

**Never modify `authorId`** or other system fields. `authorId` is the stable key that ties an update row back to a contributor.
