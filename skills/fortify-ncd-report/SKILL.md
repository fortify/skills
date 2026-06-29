---
name: fortify-ncd-report
description: "Generate Number of Contributing Developers (NCD) reports with `fcli license ncd-report`. Use when users ask for Fortify NCD reports, `ncd-report`, contributor counting, license reporting; discover Fortify-scanned repositories in GitHub, GitLab, or Azure DevOps; compare repositories against FoD or SSC applications; run current or historical reports; or AI-assist contributor deduplication."
license: MIT
metadata:
  version: "1.0.0"
  tested-with:
    fcli: "3.21"
argument-hint: "SCM platform, Fortify platform, role (producer or consolidator), org-wide model, current vs historical"
---

# Fortify NCD Reports

This skill guides end-to-end generation and cleanup of Fortify Number of Contributing Developers (NCD) reports using `fcli license ncd-report`.

By default, this skill assumes an organization-wide reporting goal. If license pools are split across organizational units (for example departments), treat each unit as a separate reporting domain and run the workflow per domain.

> Stay within the report workflow. Do not jump ahead to merge, deduplication, or Fortify inventory comparisons before the current step establishes that they are needed.

Use this skill when the user needs to:
- Generate or customize an `NcdReportConfig.yml`
- Discover which repositories are likely scanned by Fortify
- Compare repository inventory against FoD or SSC application names
- Produce a current-period or historical NCD report
- Merge multiple team reports into one reporting-domain result (organization-wide by default)
- Review and correct duplicate or ignored contributors with `lsc` and `ucs`

For detailed command patterns and repository discovery heuristics, load [references/ncd-report-workflow.md](./references/ncd-report-workflow.md).

If the user needs deeper FoD or SSC portfolio analysis while preparing the report, activate the `fortify-fod` or `fortify-ssc` skill as appropriate. If `fcli` is missing or authentication needs setup help, activate `fcli-common`.

## Step 0: Route to the right NCD task first

Determine which path applies before asking broader questions.

### Step 0a: New report workflow

Use this path when the user wants to:
- create or update `NcdReportConfig.yml`
- discover which repositories belong in scope
- compare repo inventory against FoD or SSC apps
- generate a current or historical report (using either existing or newly generated config)

Proceed to Step 1.

### Step 0b: Merge existing reports

Use this path when the user already has two or more team reports and wants one reporting-domain result (organization-wide by default).

Skip repository discovery unless the user explicitly asks for it. Proceed to Step 5.

### Step 0c: Review contributor identities in an existing report

Use this path when the user already has a report and wants to inspect or correct duplicate, ignored, or bot contributors using `lsc` or `ucs`.

Skip config generation and repository discovery unless the user explicitly asks for them. Proceed to Step 6.

### Step 0d: Ambiguous request

If the user asked for an "NCD report" without enough detail to choose one path, narrow the ambiguity with the fewest needed questions:
- New report vs merge vs cleanup of an existing report
- Default organization-wide execution model: single org-wide create run vs team reports plus merge
- User role: producer (team or domain report generator), consolidator, or both
- License pool model: one shared org pool vs separate unit pools (which implies separate reporting domains)
- Current cycle vs historical cycle

Step 0 gate
- [ ] One path selected
- [ ] Unneeded workflow branches deferred

## Step 1: Establish scope before generating anything

Confirm or infer these decisions first:
- Confirm that the reporting goal is **organization-wide by default**.
- If license pools are split (for example different departments/unit purchasing their own licenses), confirm the specific **reporting domain** for this run.
- Which execution model should produce the reporting goal for this run?
  - **Single-run model**: one `create` config processes all in-scope Fortify-scanned repos.
  - **Federated model**: teams/departments generate separate reports that are merged at the reporting domain (usually organization-wide).
- Confirm the user role for this run: **producer**, **consolidator**, or **both**.
- If the user is acting as a producer for a new report, ask whether they already have an `NcdReportConfig.yml` or equivalent config file and whether it should be reused as-is or adjusted.
- Is the user generating a **current cycle** report or a **historical** report for a prior 90-day window?
- Which SCM systems are in scope: GitHub, GitLab, Azure DevOps, or a mix?
- Which Fortify deployment is the source of truth for comparison: FoD, SSC, or neither?

Choose between execution models based on operating constraints. Prefer the federated model (team reports + `merge`) when ownership boundaries or access boundaries are split by team/department. Prefer the single-run model when one centrally managed scope can process all in-scope repos reliably.

Role-to-model mapping guidance:
- **Single-run model**: one actor often performs both roles (producer + consolidator) in one end-to-end run.
- **Federated model**: producers generate team/domain reports; consolidators collect those reports and run `merge` for the reporting-domain result.
- **Both roles in federated mode**: one user may still execute both responsibilities, but should keep producer outputs and consolidation steps logically separated.

If the user is acting as a producer in this path, ask whether they already have an `NcdReportConfig.yml` or equivalent config file.
- If yes, ask whether it can be used as-is or whether the user wants help adjusting it.
- If it can be used as-is, skip most repository discovery and config-authoring work and move directly to report execution.
- If it needs adjustment, use the existing file as the starting point rather than generating a new config from scratch.

If the Fortify platform was not specified, check for an active fcli session before asking:

```bash
fcli fod session ls --query "expired=='No'"
fcli ssc session ls --query "expired=='No'"
```

- If only FoD is active, treat FoD as the comparison target when comparison is needed.
- If only SSC is active, treat SSC as the comparison target when comparison is needed.
- If both are active, ask the user which platform should drive the comparison.
- If neither is active, continue with SCM-side report setup and ask about Fortify comparison only if that branch is still needed.

For historical reports, capture the report end date up front. The `create` command uses `--end-date yyyy-MM-dd` and treats that date as the inclusive end of the 90-day window.

If the federated model is chosen, align report windows by using the same end date across all source reports before merging, for example by suggesting the end date of the most recent completed quarter.

Step 1 gate
- [ ] Reporting goal confirmed (organization-wide by default)
- [ ] Reporting domain for this run confirmed (organization by default, or specific unit/domain for split pools)
- [ ] Execution model selected (single-run or federated)
- [ ] User role identified (producer, consolidator, or both)
- [ ] If a producer is generating a new report, existing config status identified
- [ ] License-pool constraints captured
- [ ] Current vs historical identified
- [ ] SCM system identified
- [ ] Fortify comparison target identified or intentionally deferred

## Step 2: Discover repositories that belong in the report

If the user is a producer generating a new report and already has a config that can be reused as-is, skip this step.

Before doing repository discovery, ask whether the user wants AI-assisted discovery. Make the tradeoff explicit: this can be time-consuming and may require SCM access.

Offer these paths:
1. **AI-assisted discovery**: inspect repositories, CI/CD definitions, labels/topics, and possibly Fortify inventory to help determine the right scope.
2. **User-provided selectors**: help encode `repositoryIncludeExpression`, organizations, groups, projects, or explicit repo filters based on repo names, labels, topics, or other criteria the user already has.
3. **Fortify-stored repo metadata**: use repository URLs or similar metadata already stored on FoD or SSC application records if the customer maintains that data there.
4. **Skeleton only**: generate a config stub from the earlier answers and tell the user exactly which sections to update manually.

Only do full repository discovery if the user explicitly wants path 1.

For path 3, first ask whether repo URLs are stored in FoD or SSC and, if the user knows, ask for the attribute name.
- For **FoD**, ask whether the repo URL is stored on the **application** or **release** record.
- For **SSC**, assume attributes exist only at the **application version** level.

If an FoD or SSC session is available and the user does not know the attribute name, you may auto-discover candidate attributes by listing a single representative app, release, or application version with embedded attributes and looking for attribute names or values that appear to contain repository URLs. Treat this only as a hint and verify the chosen attribute with the user before relying on it for discovery.

If AI-assisted discovery is selected, inspect the repository host, CI/CD definitions, and naming conventions before editing the config.

Priorities:
1. Find Fortify-related workflow or pipeline steps that indicate a repo is scanned.
2. Recommend repository metadata that makes inclusion deterministic.
3. Compare discovered repositories against either FoD or SSC application names when the Fortify platform is known.

Prefer repository topics or labels that let `repositoryIncludeExpression` select scanned repos automatically. For GitHub and GitLab, repository topics are usually the cleanest option after verifying scan markers in CI. For Azure DevOps, where topic support may not exist, prefer project scoping plus explicit repo name filters.

If the user already provided a curated repository list, labels/topics, or Fortify attribute metadata and does not want discovery help, skip directly to Step 3 and encode the provided selectors.

If the user is a producer generating a new report and the existing config only needs small adjustments, prefer editing that file over re-discovering repository scope from scratch.

Step 2 gate
- [ ] Discovery path chosen: AI-assisted, user-provided selectors, Fortify-stored metadata, or skeleton only
- [ ] Repo inclusion method chosen: CI evidence, metadata, Fortify attributes, explicit filters, curated list, or manual follow-up
- [ ] If Fortify-stored metadata was chosen, attribute scope and attribute name identified or confirmed with the user
- [ ] If AI-assisted discovery was chosen, findings captured well enough to encode config rules

## Step 3: Generate the config stub, then harden it

If the user is a producer generating a new report and already has a config:
- **Reuse as-is**: skip this step and proceed to Step 4, Step 5, or Step 6 as appropriate.
- **Adjust existing config**: use the existing file as the starting point and make the smallest required changes.

Start from the generated stub, not a hand-written file:

```bash
fcli license ncd-report create-config -y -c NcdReportConfig.yml -o yaml
```

Then customize it with these rules:
- Keep credentials externalized through `#env("...")` expressions whenever possible.
- Use `repositoryIncludeExpression` to encode the repo selection rule.
- Keep `ignoreExpression` and `duplicateExpression` readable and multiline.
- Prefer organization/group/project selectors over long hand-maintained repo lists when the SCM metadata is reliable.
- Keep YAML close to the generated structure unless the user has a strong reason to restructure it.

Apply Step 3 based on the Step 2 path:
- **AI-assisted discovery**: encode the discovered repo selection logic directly in the config.
- **User-provided selectors**: translate the provided repo names, labels, topics, organizations, groups, or projects into the minimal config needed.
- **Fortify-stored metadata**: use the confirmed FoD or SSC attribute as the source for repo selection rules or for generating a curated list to encode in the config.
- **Skeleton only**: generate the scaffold, prefill what is already known, and clearly mark which sections the user still needs to update manually.

If Fortify-stored metadata is available, prefer that path over time-consuming SCM crawling when it gives a reliable repo list or repo URL source.

If the federated execution model is chosen, structure config and output boundaries so each team/department report maps cleanly to its reporting domain. This makes org-level merge behavior and license accounting easier to reason about.

Step 3 gate
- [ ] Config scaffold generated from `create-config`
- [ ] Tokens externalized
- [ ] Repo selection logic encoded clearly, or manual follow-up sections identified explicitly
- [ ] If Fortify-stored metadata was used, chosen attribute was confirmed with the user before depending on it
- [ ] Team boundaries explicit when needed

## Step 4: Run the report and validate the output

Prefer zip output by default for cleaner artifact handling, unless the user explicitly asks for directory output.

Run one of these forms:

```bash
fcli license ncd-report create -y -c NcdReportConfig.yml -z ncd-report.zip
fcli license ncd-report create -y -c NcdReportConfig.yml -d ncd-report
fcli license ncd-report create -y -c NcdReportConfig.yml -d ncd-report-2026Q1 --end-date 2026-03-31
fcli license ncd-report create -y -c NcdReportConfig.yml -z ncd-report-2026Q1.zip --end-date 2026-03-31
```

Treat the report as valid only if the output contains at least:
- `summary.txt`
- `contributors.csv`
- `checksums.sha256`
- `report-config.yaml`
- detail CSV files under `details/`

Review the summary for author counts, commit counts, repository counts, and any obvious scope mistakes.

If the federated execution model is being used, ensure each source report uses the same end date before proceeding to Step 5.

If the user role is producer-only in a federated workflow, the default next action after Step 4 is handoff to the consolidator rather than continuing into merge.

If validation exposes obvious scope problems, return to Step 2 or Step 3 before moving on to merge or cleanup.

Step 4 gate
- [ ] Report generated successfully
- [ ] Expected files present
- [ ] Summary reviewed for obvious scope/count problems

## Step 5: Merge federated reports into one reporting-domain result

This step is primarily for consolidators (or users acting in both roles).

Use `merge` when teams generated separate reports within the same reporting domain or when cross-team contributor overlap would otherwise double-count people.

Prefer zip output for merged reports as well, unless the user explicitly prefers a directory.

```bash
fcli license ncd-report merge -r team-a-report.zip,team-b-report.zip -z full-report.zip -y
fcli license ncd-report merge -r team-a-report.zip,team-b-report.zip -d full-report -y
```

After merging, review the merged contributor list and summary to confirm cross-report deduplication produced sensible counts.

Only use `merge` when the federated model is selected or source reports already exist.

Do not merge reports across different licensing/reporting domains unless the user explicitly confirms this is allowed for their compliance and licensing model.

For consistency, source reports should ideally use the same end date before merge. If source reports use different end dates, flag this to the user because merged totals may not represent a single consistent reporting window.

Step 5 gate
- [ ] Source reports identified
- [ ] All expected source reports for the reporting domain were received, or missing reports explicitly acknowledged
- [ ] Source reports belong to the same reporting domain, or cross-domain merge explicitly approved
- [ ] Source reports have matching end date, or mismatch acknowledged explicitly
- [ ] Merged summary reviewed
- [ ] Cross-report deduplication checked at a high level

## Step 6: Use `lsc` and `ucs` for manual or AI-assisted cleanup

This step is usually performed by consolidators for merged reports, but may also be used by producers for local cleanup before handoff when requested.

The cleanup loop is:
1. Export contributors from a report.
2. Review duplicates or ignored contributors manually or with AI assistance.
3. Feed the edited result back into the report.

If the user ultimately needs one reporting-domain result from multiple team reports, prefer running `merge` first and then doing `lsc`/`ucs` cleanup on the merged report. This allows deduplication to happen once across the full contributor set and catches cross-team duplicates that would be invisible inside individual team reports.

Example:

```bash
fcli license ncd-report list-contributors -r ncd-report.zip -o json --to-file contributors.json
fcli license ncd-report update-contributor-status -r ncd-report.zip -c contributors.json
```

The update file should come from `list-contributors` output and must include `authorId`. Useful editable fields are `duplicateOf`, `overrideStatus`, `overrideStatusConfidence`, and `overrideStatusNotes`.

Use [assets/lsc-to-ucs-review-template.md](./assets/lsc-to-ucs-review-template.md) as a starting point when you want the AI to review `list-contributors` output and emit a minimal JSON update file for `update-contributor-status`.

If the user needs to see the exact input and output shape first, use [assets/lsc-to-ucs-worked-example.md](./assets/lsc-to-ucs-worked-example.md) before generating the real review prompt.

If AI-generated updates are used, keep or raise the confidence threshold unless the user explicitly wants a looser review:

```bash
fcli license ncd-report update-contributor-status -r ncd-report.zip -c contributors.json --min-confidence 0.90
```

After applying updates, rerun `list-contributors` to spot-check that the intended rows changed and that no broad misclassification was introduced.

Step 6 gate
- [ ] If org-wide deduplication is needed across federated reports, merge happened first or was consciously skipped
- [ ] Updates derived from `lsc` output
- [ ] `authorId` preserved exactly
- [ ] Confidence threshold chosen deliberately
- [ ] Post-update spot check completed

## Step 7: Completion checks

Determine completion based on the user's role in the federated process.

If the user is a team/domain producer, consider their workflow complete when:
- Execution model choice (single-run or federated) is documented
- Repo inclusion logic is documented and reproducible
- Credentials are externalized instead of embedded
- The generated report files and summary look internally consistent
- The report artifact for their domain/team was generated and handed off to the consolidator

If the user is the consolidator, consider the workflow complete only when:
- Reporting goal for the current domain is met (organization by default, or specific unit/domain for split pools)
- All relevant source reports were received or missing reports explicitly acknowledged
- Any needed merge has been performed
- Any needed `lsc`/`ucs` cleanup has been applied and rechecked
