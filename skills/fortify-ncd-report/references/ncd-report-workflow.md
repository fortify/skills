# Fortify NCD Report Workflow Reference

Use this reference when you need exact command patterns for repository discovery, config authoring, report execution, merging, and contributor cleanup.

Organization-wide reporting is the default end goal. If license pools are split across organizational units, treat each unit as a separate reporting domain and run this workflow per domain. There are two valid execution models:
- **Single-run model**: one `create` configuration processes all in-scope Fortify-scanned repositories in one run.
- **Federated model**: teams/departments generate separate reports that are merged at the reporting-domain level (organization-wide by default).

Choose between execution models based on operating constraints. Prefer the federated model when ownership boundaries or access boundaries are split by team/department. Prefer the single-run model when one centrally managed scope can process all in-scope repos reliably.

Determine execution model first, then make role assignments explicit:
- In **single-run** mode, one actor often performs both roles (producer + consolidator) in one end-to-end run.
- In **federated** mode, role assignment is explicit:
  - **Producer**: generates a team or domain source report and hands it off.
  - **Consolidator**: collects source reports, merges, and optionally performs post-merge cleanup.
  - **Both**: one user performs both producer and consolidator responsibilities, but should keep producer outputs and consolidation steps logically separated.

Before doing any discovery work for a producer generating a new report, ask whether the user already has an `NcdReportConfig.yml` or equivalent config file.
- If it can be reused as-is, skip repository discovery and config authoring.
- If it needs adjustment, use it as the starting point and avoid rebuilding scope from scratch unless the existing config is clearly incomplete or wrong.

## Step 1: Discover which repositories are likely scanned by Fortify

Skip this step if a producer already has a config that can be reused as-is.

Before doing discovery work, ask the user whether they want AI-assisted repository discovery. Make the cost explicit: it can be time-consuming and may require SCM access.

Offer three modes:
- **AI-assisted discovery**: inspect repositories and CI/CD definitions to infer the report scope.
- **User-provided selectors**: use repo names, labels/topics, organizations, groups, or project names the user already has.
- **Fortify-stored metadata**: use repository URLs or similar metadata already stored in FoD or SSC attributes.
- **Skeleton only**: skip discovery and generate a config the user can finish manually.

Only continue with this step if the user chose AI-assisted discovery. Otherwise, skip to Step 3 and encode what the user already knows or what Fortify already stores.

### Fortify-stored metadata option

Before doing SCM crawling, ask whether repo URLs are already stored in Fortify metadata.

- For **FoD**, ask whether the relevant attribute is on the **application** or **release** record.
- For **SSC**, assume the relevant attribute is on the **application version** record.

If the user knows the attribute name, use that directly.

If the user does not know the attribute name but an FoD or SSC session is active, you may inspect one representative app, release, or application version with embedded attributes and look for candidate attribute names or values that appear to hold repository URLs. Examples might include names containing `repo`, `repository`, `git`, `scm`, `url`, or values that resemble GitHub, GitLab, or Azure DevOps URLs.

Do not silently trust guessed attribute names. Confirm the candidate attribute with the user before using it as the basis for repo discovery.

Inspect source repositories before drafting the final config.

### CI/CD indicators to look for

Common scan markers include:
- GitHub Actions workflows containing `fortify`, `fcli`, `sourceanalyzer`, `scancentral`, `FoD`, or `SSC`
- GitLab CI jobs or included templates containing the same markers
- Azure DevOps pipelines with Fortify tasks, `fcli`, `sourceanalyzer`, or `scancentral`
- Jenkinsfiles or shared pipeline libraries invoking Fortify tooling

If you are working inside a code workspace, search for Fortify markers in workflow files before asking the user for additional manual repo lists.

### Prefer metadata-driven repo selection

For GitHub and GitLab, recommend repository topics that clearly mark repos scanned by Fortify, for example:
- `scanned-by-fortify`
- `fortify-integration`
- `fortify-ssc`
- `fortify-fod`

Then encode those topics directly in `repositoryIncludeExpression`, for example:

```yaml
repositoryIncludeExpression: >
  topics.contains("scanned-by-fortify")
```

If the organization does not use topics consistently, fall back to:
- explicit project or organization scoping
- repo name patterns
- curated include lists maintained in config comments or companion docs

For Azure DevOps, prefer project-level scoping and name-based filters because repo topics are not typically available.

## Step 2: Compare repositories against FoD or SSC applications

Use fcli product commands to spot mismatches between discovered repos and Fortify inventory.

### FoD inventory

```bash
fcli fod app ls -o json
```

Use this when the user wants to compare repo names to FoD application names. Normalize names before comparing because app names may differ from repo names by prefixes, suffixes, or environment labels.

If you need deeper FoD guidance, activate the `fortify-fod` skill.

### SSC inventory

```bash
fcli ssc app ls -o json
```

Use this when the user wants to compare repo names to SSC applications. Expect one app per repo in cleaner environments, but do not assume a perfect one-to-one match.

If you need deeper SSC guidance, activate the `fortify-ssc` skill.

### Comparison heuristics

Check for these mismatch categories:
- repo appears scanned in CI but no matching FoD or SSC app exists
- FoD or SSC app exists but no active repository seems to map to it
- multiple repos appear to map to one app, which may indicate monorepo or naming drift
- one repo maps to multiple apps, which may indicate branch, microservice, or migration history

When mismatches are found, ask whether the NCD report should follow repository reality, Fortify app inventory, or a curated business-owned list.

If it is unclear whether the customer uses FoD or SSC, ask before doing product inventory comparisons. Do not assume both are relevant. Most environments use one primary Fortify platform.

## Step 3: Create and customize the NCD config

If a producer already has a config:
- **Reuse as-is**: skip directly to report execution.
- **Adjust existing config**: skip stub generation; treat the current config as the source artifact and make only the necessary changes.

Generate the stub first:

```bash
fcli license ncd-report create-config -y -c NcdReportConfig.yml -o yaml
```

Then choose the lightest config-authoring path that satisfies the user:
- **AI-assisted discovery**: encode the repository selection rules discovered in Step 1 and any comparison insights from Step 2.
- **User-provided selectors**: convert repo names, labels/topics, orgs, groups, or projects into `repositoryIncludeExpression` and related source selectors.
- **Fortify-stored metadata**: convert the confirmed FoD or SSC attribute values into curated repo selectors, repo-name filters, or manual include lists as appropriate.
- **Skeleton only**: prefill known platform, org/group/project, and credential settings, then leave clear placeholders or comments for the user to finish manually.

If Fortify-stored metadata yields a reliable list of repository URLs, prefer it over SCM crawling because it is usually faster and easier to validate with the user.

### Credential handling

Prefer environment variables over embedded secrets:

```yaml
tokenExpression: >
  #env("GITHUB_TOKEN")
```

Equivalent patterns work for GitLab and Azure DevOps tokens.

### GitHub example

```yaml
sources:
  github:
  - apiUrl: https://api.github.com
    tokenExpression: >
      #env("GITHUB_TOKEN")
    repositoryIncludeExpression: >
      topics.contains("scanned-by-fortify")
    organizations:
    - name: my-org
```

### GitLab example

```yaml
sources:
  gitlab:
  - baseUrl: https://gitlab.com
    tokenExpression: >
      #env("GITLAB_TOKEN")
    repositoryIncludeExpression: >
      topics.contains("scanned-by-fortify")
    groups:
    - id: my-group
```

### Azure DevOps example

```yaml
sources:
  ado:
  - baseUrl: https://dev.azure.com
    tokenExpression: >
      #env("AZURE_DEVOPS_TOKEN")
    repositoryIncludeExpression: >
      name matches "service-a|service-b|portal"
    organizations:
    - name: my-org
      projects:
      - name: shared-platform
```

### Contributor cleanup defaults

The generated sample already includes practical defaults. Preserve or refine them rather than replacing them with opaque logic:

```yaml
contributor:
  ignoreExpression: >
    lcName matches '.*\[bot\]'
  duplicateExpression: >
    a1.cleanName==a2.cleanName ||
    a1.cleanEmailName==a2.cleanEmailName ||
    a1.cleanName==a2.cleanEmailName
```

## Step 4: Run current-cycle or historical reports

Prefer zip output by default for cleaner artifact handling and easier sharing/storage, unless the user explicitly asks for directory output.

### Current cycle

```bash
fcli license ncd-report create -y -c NcdReportConfig.yml -z ncd-report.zip
fcli license ncd-report create -y -c NcdReportConfig.yml -d ncd-report
```

### Historical cycle

```bash
fcli license ncd-report create -y -c NcdReportConfig.yml -z ncd-report-2026Q1.zip --end-date 2026-03-31
fcli license ncd-report create -y -c NcdReportConfig.yml -d ncd-report-2026Q1 --end-date 2026-03-31
```

Use `-d` instead of `-z` if the user wants directory output.

If the federated model is used, keep the same `--end-date` across all source reports so merged results represent one consistent reporting window.

### Validate output

Check that the report includes:
- `summary.txt`
- `contributors.csv`
- `checksums.sha256`
- `report-config.yaml`
- `details/repositories.csv`
- `details/commits-by-branch.csv`
- `details/commits-by-repository.csv`
- `details/contributors-by-repository.csv`

## Step 5: Merge team reports into a reporting-domain result

This step is primarily for consolidators (or users acting in both roles).

Use `merge` when separate teams manage separate configs within the same reporting domain or when overlap across team reports would double-count contributors.

Prefer zip output for merged reports by default.

```bash
fcli license ncd-report merge -r team-a-report.zip,team-b-report.zip,team-c-report.zip -z full-report.zip -y
fcli license ncd-report merge -r team-a-report.zip,team-b-report.zip,team-c-report.zip -d full-report -y
```

The merge command reuses contributor expressions and performs cross-report deduplication. Review the merged summary and contributor output rather than assuming the counts are automatically correct.

Before merging, confirm with the user that all expected team/domain source reports for the reporting domain were received. If some are missing, explicitly note that merged totals are partial.

For consistency, all source reports should ideally be generated with the same end date. If end dates differ, warn the user that merged totals may mix reporting windows.

For split-pool environments, verify that all source reports, and all repositories listed in those reports, belong to the same licensing/reporting domain before doing a merge.

## Step 6: AI-assisted deduplication with `lsc` and `ucs`

This step is usually consolidator-led for merged reports, but producers may use it for local pre-handoff cleanup when requested.

If the user has multiple team reports and wants one reporting-domain result (organization-wide by default), prefer this sequence:
1. Merge team reports first.
2. Run `list-contributors` on the merged report.
3. Perform AI-assisted review once on the merged contributor set.
4. Apply `update-contributor-status` to the merged report.

This is usually better than reviewing each team report separately because it catches duplicates that span teams and reduces repeated review effort.

### Export contributors for review

```bash
fcli license ncd-report list-contributors -r ncd-report.zip -o json --to-file contributors.json
```

Instead of `json`, `yaml` or `csv` are also valid output formats if they fit the user workflow better, for example if user wants to manually review the CSV output in Excel.

If the user wants AI assistance, start from [assets/lsc-to-ucs-review-template.md](../assets/lsc-to-ucs-review-template.md). It gives the AI a strict prompt shape and a minimal JSON output schema suitable for `ucs` input.

If the user is unsure what the exported contributor JSON looks like or what the reviewed `ucs` payload should contain, show [assets/lsc-to-ucs-worked-example.md](../assets/lsc-to-ucs-worked-example.md) first.

### Fields an AI or reviewer may update

Useful fields include:
- `duplicateOf`
- `overrideStatus`
- `overrideStatusConfidence`
- `overrideStatusNotes`

Every update row must include `authorId`.

### Apply reviewed updates

```bash
fcli license ncd-report update-contributor-status -r ncd-report.zip -c contributors.json
```

If confidence is part of the workflow, use an explicit threshold:

```bash
fcli license ncd-report update-contributor-status -r ncd-report -c contributors.json --min-confidence 0.90
```

### Suggested review pattern

1. Export contributors in JSON.
2. Ask the AI to flag probable duplicates, bots, and obviously non-human accounts.
3. Keep confidence and notes for traceability.
4. Apply updates with `ucs`.
5. Re-run `lsc` to inspect the new contribution status distribution.

Prefer having the AI emit only changed rows, not a full copy of the contributor list. This keeps human review and rollback simpler.

## Step 7: Template for AI review input

Use the template asset as-is or adapt it to the user's preferred model. The important constraints are:
- preserve `authorId` values exactly as exported
- set `duplicateOf` only to another existing `authorId`
- use `overrideStatus` only when the reviewer is intentionally forcing `contributing`, `duplicate`, or `ignored`
- include `overrideStatusConfidence` and `overrideStatusNotes` for every AI-suggested change
- output only rows that should change

The resulting JSON can be written directly to a file and applied with:

```bash
fcli license ncd-report update-contributor-status -r ncd-report.zip -c contributors-reviewed.json --min-confidence 0.90
```


## Step 8: Decision rules

Prefer the federated model (one report per team, then `merge`) when:
- different teams own different SCM scopes
- token access differs by team
- business review happens per team before a domain-level rollup

Prefer the single-run model (one domain-wide config, typically org-wide) when:
- repository metadata is clean and centrally managed
- one automation identity can read the full SCM scope
- the user wants a single repeatable scheduled run

Prefer repository-topic discovery when:
- GitHub or GitLab topics are available and enforced
- the organization wants low-maintenance onboarding for newly scanned repos

Prefer CI or pipeline discovery as the first pass when:
- the current repo set is not yet labeled consistently
- the user wants evidence that a repo is actually scanned before tagging it
- you need to recommend which topics or labels should be standardized later

Prefer explicit repo filters when:
- topics are inconsistent
- Azure DevOps is the primary SCM
- regulated reporting requires a tightly curated scope

## Step 9: Completion by role

Completion depends on whether the user is producing sub-reports or consolidating them.

### Producer completion

For a team/domain producer in a federated workflow, completion means:
- their report was generated for the agreed reporting window
- artifact quality checks passed
- the report artifact was handed off to the consolidator

Producer completion does not require running merge.

### Consolidator completion

For the consolidator, completion means:
- all relevant source reports were received (or known gaps were explicitly acknowledged)
- source report domain boundaries and end-date consistency were checked
- merge output was generated and reviewed
- optional post-merge `lsc`/`ucs` cleanup was completed and spot-checked