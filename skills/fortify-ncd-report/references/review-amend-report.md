# Task: Review and Amend an NCD Report

Use this workflow to **inspect or modify** an existing report — list contributors,
explain an unexpected count, or amend contributor status (mark duplicates, ignore
accounts, override status). For the NCD count rules and field definitions, see
[concepts.md](concepts.md).

## Non-negotiable rules

- Apply non-negotiable rules from [../SKILL.md](../SKILL.md)

## Mandatory Workflow

Complete each step before proceeding. Do not skip steps.

### Step 1: Select report input and route if needed

#### Step 1a: Identify whether report to be reviewed is already known

If user explicitly requested a specific report to be reviewed in the report review & amend prompt (as free-format text or attachment), continue to step 1c.

If any reports were generated or otherwise discussed earlier in this chat, present a Q&A dialog with title 'Which report would you like to review or amend' and exactly the following choices:
- A choice item for every previously generated or discussed report, with the choice item showing the report path, optionally together with a short description/reference of why this item is included in this list
- 'A different report' choice (without free-format entry; which different report will be determined in step 1b) 

If user selects a specific report, use the selected report path to continue to step 2.
If user selects 'A different report' or doesn't make a selection, continue to step 1b.

#### Step 1b: Ask user to provide report path

Ask user to provide the path to the report to be reviewed. Do not use Q&A dialog; suggest or use the most user-friendly approach for selecting a file or directory, for example:
- If assistant supports opening a file selection dialog, use this to allow user to select file or directory
- User attaches file or directory to next prompt
- User provides file or directory path in next prompt

Once user has provided a file or directory path (which may include wildcard patterns), continue to step 1c.

##### Step 1b gate
- [ ] File or directory path provided by user, which may include wildcard patterns

#### Step 1c: Resolve report path

Resolve the report path(s) identified in previous steps, which may include wildcard patterns, against actual file system paths.

For a report to be considered valid, all of the following rules **must** match:
- A valid report may be contained in either a *zip file with `.zip` extension*, or an arbitrarily named *directory*
- A valid report contains at least `summary.txt`, `report-config.yaml`, `contributors.csv`, and `details/*.csv` at zip/dir root level
- `sources/*` sub-directories inside a valid report zip or dir **are not** valid reports for this task; they represent source reports in a merged report; only the merged report itself is a review/amend candidate

For each of the resolved path(s), identify all **valid** report paths based on these criteria:
- A resolved path points directly at a valid report
- A resolved path contains direct children that represent one or more valid reports; **do not** perform a recursive search

Based on the valid report paths found above, apply these routing rules:
- If analysis found a single valid report, use this zip file or directory and continue to step 2.
- If analysis found multiple valid reports, continue to step 1d.
- If none of the routing rules above apply, inform the user and go back to step 1b.

#### Step 1d: Select or merge reports

For each of the valid reports identified in step 1c, check whether they contain a `sources` sub-directory at root level of the report zip/dir:
- If present, classify the report as a previously merged report
- If not present, classify the report as an regular (non-merged) report

Explain to user in chat output in concise, user-friendly wording that multiple reports were found, but only one report can be reviewed/amended at a time; if these are reports for individual reporting units for a single reporting domain in the federated execution model (see [concepts.md](concepts.md)), reports should be merged before review/amend.

After this explanation, present in chat output the list of paths for the identified reports, clearly segregated by report type (regular, non-merged reports vs previously merged reports) as identified above, listing non-merged reports first, merged reports after. Use the report paths themselves as the canonical labels for these candidates in subsequent prompts, optionally prefixed with the regular/merged classification.

Then, ask user how to continue and route accordingly:
- Review/amend single report => Show single-choice Q&A dialog listing all identified reports using the same canonical labels already shown in chat output; continue to step 2 once user has chosen single report
- Merge regular reports, excluding previously merged reports => Continue to step 1f
- Merge all reports, including previously merged reports => Continue to step 1f
- Manually select the reports to be merged => Show multiple-choice list using the same canonical labels already shown in chat output; do not restate a second separately formatted full inventory before the prompt; continue to step 1f once user has chosen the reports to be merged
- Something else => Return to step 1b
 
#### Step 1f: Merge reports

Based on user selection in step 1d, run the [merge-reports.md](merge-reports.md) workflow to merge the reports that match user's selection; while executing this workflow, do not ask any questions related to source report selection, as that selection is already known.

Once selected reports have been successfully merged, continue to step 2 with the merged report as input.

#### Step 1 gate
- [ ] Report input provided or explicitly requested from user
- [ ] If applicable, merge workflow was executed
- [ ] A single, valid report path was identified on which subsequent steps will operate

### Step 2: Display report summary

Run `fcli license ncd-report get-summary -r <(merged) report file or directory>`, then show literal output in chat output, just applying proper formatting to make the output more user-friendly (like converting property names to human-readable names) and visually appealing.

#### Step 2 gate
- [ ] Literal, formatted report summary shown to user

### Step 3: Task selection

Offer user the following choices in Q/A style dialog:
- List & update repositories => Continue to step 4
- List & update contributors => Continue to step 5
- Something else => Identify what the user wants to do, and use any information listed in this workflow file as background info to assist the user in performing that task

### Artifact-first review mode

For anything beyond a tiny report, prefer review artifacts over streaming full inventories in chat.
Use `--to-file` exports, generate a markdown worksheet next to the report or in the approved working
directory, and keep chat output to a short summary plus the artifact location.

Recommended pattern:
- Export the raw report data to JSON with `--to-file`.
- Generate a compartmented markdown worksheet from the JSON export.
- Use the worksheet as the review surface: the user can tick boxes, add notes, and fill in fields such
    as `duplicateOf`, `overrideStatus`, or config-change hints.
- If the assistant/runtime can render markdown directly from fcli expression output, use that; otherwise
    use a small local script to transform the JSON export into the worksheet.

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

Use `repositories-included.json` to populate three formatted sections in the worksheet (omit empty
lists):
- Dormant repositories (`dormant == 'true'`)
- Non-dormant repositories (`dormant == 'false'`)
- Unknown-dormancy repositories (`dormant == 'unknown'`)

Chat output requirement (mandatory):
- Persist full repository lists to a review artifact markdown file and, when useful, a machine-readable companion selection file. Prefer the worksheet in [assets/repository-scope-review-template.md](../assets/repository-scope-review-template.md).
- In chat, show a concise summary and only a bounded preview, or no preview at all if the worksheet is the primary review surface.
- Do not respond with only a file path, "export complete", or "analysis complete" message.

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

Chat output requirement (mandatory):
- Persist the full recommendation set to a review artifact markdown file and, if helpful, a machine-readable
    companion file such as `repositories-reviewed.json` containing the checked rows from the worksheet.
- In chat, print only a concise recommendation preview before asking the user how they want to continue.
- For each shown preview candidate, include repository URL, confidence tier, short evidence, and suggested config adjustment.

Prefer the worksheet in [assets/repository-scope-review-template.md](../assets/repository-scope-review-template.md) as the user-facing review surface. If the candidate set is small and the runtime supports it, you may still present a clickable shortlist; otherwise ask the user to mark the worksheet and return the edited file.

If no repositories were selected, return to step 3; otherwise continue below.

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
- If the user is reviewing the worksheet offline, summarize the selected rows and map them back to the relevant source report before proposing config changes.

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
- If the review was done through `repositories-reviewed.json` or the markdown worksheet, derive the excluded repository URL set from that reviewed artifact first and then filter contributors against that set.

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

Use the output to populate three formatted sections in the worksheet (omit empty lists):
- Dormant contributors (`dormant == 'true'`)
- Non-dormant contributors (`dormant == 'false'`)
- Unknown-dormancy contributors (`dormant == 'unknown'`)

Chat output requirement (mandatory):
- Persist full contributor lists to a review artifact markdown file and, when useful, a machine-readable companion selection file. Prefer the worksheet in [assets/contributor-review-worksheet-template.md](../assets/contributor-review-worksheet-template.md).
- In chat, show a concise summary and only a bounded preview, or no preview at all if the worksheet is the primary review surface.
- Do not respond with only a file path, "export complete", or "analysis complete" message.

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

Output findings from the above in chat and persist a full findings artifact markdown file. Prefer to populate the worksheet in [assets/contributor-review-worksheet-template.md](../assets/contributor-review-worksheet-template.md) first, then use [assets/lsc-to-ucs-review-template.md](../assets/lsc-to-ucs-review-template.md) when turning approved rows into `contributors-reviewed.json`.

If no suggested updates are available, return to step 3. If suggested updates are available, keep chat output to a concise candidate summary and the artifact path(s). Let the user either:
- tick boxes and fill fields in the worksheet, then hand the edited file back for conversion, or
- use a small interactive shortlist only when the candidate count is tiny and the runtime makes that practical.

When a shortlist is used, preserve stable candidate ids/indexes in the worksheet and in chat output. For high-volume reports, split the worksheet into bounded sections/pages instead of pushing the full set through chat.

##### Step 5b gate
- [ ] User confirmed which auto-detected ignore/deduplication updates to apply, if there are any

#### Step 5c: Update report

Based on the selection made in step 5b, or on the checked rows in the contributor worksheet, create a minimal `contributors-reviewed.json` update file, with each array entry containing `authorId`, `overrideStatus`, `overrideStatusConfidence`, `overrideStatusNotes`, and `duplicateOf` (must point to existing author id) if applicable, then run:

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
