# Task: Review and Amend an NCD Report

Use this workflow to **inspect or modify** an existing report — list contributors,
explain an unexpected count, or amend contributor status (mark duplicates, ignore
accounts, override status). For the NCD count rules and field definitions, see
[concepts.md](concepts.md).

## Non-negotiable rules

- Apply non-negotiable rules from [../SKILL.md](../SKILL.md)

## Command reference

Core commands used in this workflow:

- `fcli license ncd-report get-summary -r <report path>`
- `fcli license ncd-report list-repositories -r <report path> ...`
- `fcli license ncd-report list-contributors -r <report path> ...`
- `fcli license ncd-report update-contributor-status -r <report path> -c <updates file>`

Alias equivalents (same behavior):

- `list-repositories` = `lsr`
- `list-contributors` = `lsc`
- `update-contributor-status` = `ucs`

## Mandatory Workflow

Complete each step before proceeding. Do not skip steps.

### Step 1: Select report input and route if needed

Ask the user to provide report input first unless already provided. Do not begin broad filesystem hunts before asking.

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

#### Step 1 gate
- [ ] Report input provided or explicitly requested from user
- [ ] Any auto-discovery was limited to one quick pass and user-confirmed
- [ ] If multiple reports/wildcard were provided, merge workflow was executed first

### Step 2: Display report summary

Run `fcli license ncd-report get-summary -r <(merged) report file or directory>`, and show the output to user. Do not remove any data, just add proper formatting if needed to make output more visually appealing.

#### Step 2 gate
- [ ] Full report summary shown to user

### Step 3: Task selection

Offer user the following choices in Q/A style dialog:
- List & update repositories => Continue to step 4
- List & update contributors => Continue to step 5
- Something else => Identify what the user wants to do, and use any information listed in this workflow file as background info to assist the user in performing that task

### Step 4: List & update repositories

Use this step to review repository scope in the current report and optionally
prepare repository-driven contributor updates.

Large output rule (mandatory):
- For commands expected to produce large output, use `--to-file` and work from files rather than rendering full JSON inline.
- If your assistant supports memory files, persist compact artifacts there. Otherwise persist compact artifacts to local files next to the report.

#### Step 4a: Export and present included repositories

Run:

```bash
fcli license ncd-report list-repositories \
    -r <report path> \
    -q 'status=="included"' \
    --embed=contributors \
    -o json \
    --to-file repositories-included.json
```

Use `repositories-included.json` to present three formatted lists (omit empty
lists):
- Dormant repositories (`dormant == 'true'`)
- Non-dormant repositories (`dormant == 'false'`)
- Unknown-dormancy repositories (`dormant == 'unknown'`)

For each repository entry, include:
- `repositoryUrl` (primary identifier)
- `fork`, `dormant`, `status`, `sourceReport` (if present)
- raw counts: `contributorCountRaw`, `commitCountRaw` (if available)
- up to 3 contributor emails from embedded `contributors`, plus a continuation marker if there are more

If any dormant repositories exist, explain immediately after that list:
- Dormant repositories are relevant for NCD reporting per criterion 2 in [concepts.md](concepts.md).
- Dormant repositories do not map 1:1 to NCD count.

Then ask user to choose:
- AI-assisted repository scope cleanup => continue to Step 4b
- Continue with another task => return to Step 3

##### Step 4a gate
- [ ] Included repositories exported to file
- [ ] Three dormancy lists presented (omitting empties)
- [ ] Dormant-repository impact explained when applicable

#### Step 4b: Identify repositories likely out of reporting scope

Analyze `repositories-included.json` for likely out-of-scope repositories, such as:
- forks that should not count (`fork == 'true'`)
- test/sandbox/demo/sample/docs repositories
- third-party or mirror repositories
- archived/deprecated repositories that are no longer in Fortify reporting domain (but note that those repositories still count if ever scanned by Fortify)

Present recommendations as confidence-tiered groups (`high`, `medium`, `low`) with:
- repository URL
- short evidence
- suggested config adjustment (`includeForks`, `repositoryIncludeExpression`, or source-level selector narrowing)

Present the above in multiple-choice lists, allowing user to click the repositories to be excluded. If not supported by assistant, present repositories in numbered list and ask user to enter repository numbers to be excluded.

If no repositories were selected, return to step 3, otherwise continue below.

Before proposing config changes, load effective config from the report (mandatory):
- Unmerged report input:
    - Locate and read top-level `report-config.yaml` from the report directory/zip.
- Merged report input:
    - For each selected repository, identify `sourceReport` from Step 4a output.
    - Locate and read `sources/<sourceReport>/report-config.yaml` for each impacted source report.
- If any required config file cannot be found or read, stop config-edit recommendations and report outcome as **inconclusive** for that source; continue with any sources for which config is available.

Then propose config changes using explicit mapping rules:
- Fork-only exclusions:
    - If selected repositories are predominantly forks and user intent is to exclude forks generally, propose `includeForks: false` for impacted source(s).
- Pattern-based exclusions (for example names containing `test`, `demo`, `sample`, `docs`, `sandbox`):
    - Propose tightening `repositoryIncludeExpression` to exclude those patterns.
    - Prefer broad but intentional patterns over long per-repository lists.
- Specific repository exclusions (explicit selected repository URLs/names):
    - Propose explicit negative-match clauses in `repositoryIncludeExpression` for the selected repositories.
    - Keep expressions readable; group by common prefix where possible.
- Mixed selections:
    - Combine `includeForks` and `repositoryIncludeExpression` changes as needed.

Output requirements for proposed config edits:
- Show current relevant config snippet(s) first (from loaded `report-config.yaml` files).
- Show proposed snippet(s) and a short rationale per change.
- Distinguish recommendations by source report for merged inputs.

Application routing:
- Unmerged report: offer to run [prepare-config.md](prepare-config.md) to apply edits to user-owned config.
- Merged report: explain that changes must be applied in each source team's config at origin, and provide per-source change instructions.

Once completed, ask user whether they want to update the current report to ignore contributors that only contributed to the selected repositories to be excluded:
- If yes, continue to step 4c
- If no, return to step 3.

##### Step 4b gate
- [ ] Out-of-scope candidate repositories identified with confidence and evidence
- [ ] User selection of repositories to exclude confirmed
- [ ] Effective `report-config.yaml` loaded for each impacted source (or explicitly marked inconclusive)
- [ ] Config update recommendations provided from loaded config (and routed to prepare-config if requested)

#### Step 4c: Apply repository-driven ignore candidates

Apply if explicitly confirmed in step 4b.

Prepare a minimal `contributors-reviewed.json` update file, with each array entry containing `authorId`, `overrideStatus='ignored'`, `overrideStatusConfidence`, `overrideStatusNotes`, using one of the following inputs:
- `fcli license ncd-report list-contributors -r <report path> --embed=repositories -o json -q 'contributionStatus=="contributing" && repositories.size()>0 && repositories.?[!({"<excludedRepoUrl1>","<excludedRepoUrl2>",...}.contains(repositoryUrl))].size()==0'` (use `--to-file` option if needed)
- Manual filtering of contributors that **only** appear on repositories to be excluded from the `repositories-included.json` file generated in step 4a. Do not include contributors that also appear on repositories that have not been marked for exclusion.

Then apply updates:

```bash
fcli license ncd-report update-contributor-status \
    -r <report path> \
    -c contributors-reviewed.json
```

Spot-check immediately after apply:

```bash
fcli license ncd-report list-contributors \
    -r <report path> \
    -o json \
    --to-file contributors-after-step4c.json
```

Inform user about progress, then return to Step 3.

##### Step 4c gate
- [ ] Selected repository URL set persisted from Step 4b
- [ ] Contributors to be ignored identified
- [ ] Mixed contributors (excluded + non-excluded repos) skipped from updates
- [ ] Minimal `contributors-reviewed.json` prepared
- [ ] Updates applied without additional confirmation
- [ ] Spot-check completed and summarized

### Step 5: List & update contributors

Use this step for contributor-focused analysis, including AI-assisted duplicate
and ignore candidate discovery.

Large output rule (mandatory):
- Use `--to-file` for contributor exports.
- If your assistant supports memory files, persist compact artifacts there. Otherwise persist compact artifacts to local files next to the report.

#### Step 5a: Export and present contributing contributors

Run:

```bash
fcli license ncd-report list-contributors \
    -r <report path> \
    -q 'contributionStatus=="contributing"' \
    --embed=repositories \
    -o json \
    --to-file contributors.json
```

Use the output to present three lists (omit empty lists):
- Dormant contributors (`dormant == 'true'`)
- Non-dormant contributors (`dormant == 'false'`)
- Unknown-dormancy contributors (`dormant == 'unknown'`)

For each contributor, include:
- `authorId`, `authorName`, `authorEmail`
- `sourceReport` from embedded repositories if present
- up to 3 associated `repositoryUrl` values with continuation marker if more

If dormant contributors are present, explain they still count toward NCD by default per criterion 2 in [concepts.md](concepts.md).

Then ask user to choose:
- AI-assisted contributor cleanup => continue to Step 5b
- Continue with another task => return to Step 3

#### Step 5b: Run AI-assisted identity analysis

Run a mandatory, detailed duplicate/ignore analysis on `contributors.json` as generated in step 5a to produce the following:
1. Potential ignored users (bot/service/admin/non-human accounts)
2. Potential duplicates (name variants, misspellings, abbreviations, email-local-part overlaps)
For both, confidence and notes (explaining why this is considered ignored/duplicate user) **must** be collected.

Output findings from the above. If no suggested updates, return to step 3. If suggested updates available, ask user in Q/A dialog with clear, user-friendly wording:
- Apply all
- Apply all except for the ones I select
- Apply only the ones I select

Based on selection, proceed directly to step 5c (apply all), or present a multiple-choice list to allow user to select which updates to (not) apply before proceeding to step 5c.

##### Step 5b gate
- [ ] User confirmed which auto-detected ignore/deduplication updates to apply, if there are any

#### Step 5c: Update report

Based on the selection made in step 5b, create a minimal `contributors-reviewed.json` update file, with each array entry containing `authorId`, `overrideStatus`, `overrideStatusConfidence`, `overrideStatusNotes`, and `duplicateOf` (must point to existing author id) if applicable, then run:

```bash
fcli license ncd-report update-contributor-status -r <report path> -c contributors-reviewed.json
```

Then, re-export and confirm the intended rows changed and no broad misclassification was introduced:

```bash
fcli license ncd-report list-contributors -r <report path> -o json --to-file contributors-after.json
```

#### Step 5 gate
- [ ] Contributors exported for review
- [ ] Only changed rows emitted, `authorId` preserved exactly
- [ ] Amendment count reviewed and user confirmed before applying
- [ ] Amendments applied
- [ ] Spot-check confirms the intended changes

### Completion

Work is complete when the necessary corrections are applied, the spot-check confirms
the changes are as intended, and no broad misclassification was introduced. If this was a merged report and you are the consolidator, your work is done; otherwise return to [generate-report.md](generate-report.md) or [merge-reports.md](merge-reports.md) if a regenerate/re-merge is needed.
