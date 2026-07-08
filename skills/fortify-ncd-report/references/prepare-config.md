# Task: Prepare an NCD Report Config

Use this workflow to **create a new `NcdReportConfig.yml`** or **update an existing
one**. This config is the input to the `fcli license ncd-report create` command. To
run a report from a ready config, use [generate-report.md](generate-report.md).

## Non-negotiable rules

- Apply non-negotiable rules from [../SKILL.md](../SKILL.md)
- For new config creation, **must** run:
  `fcli license ncd-report create-config -y -c <path> -o yaml`.
  Do **not** hand-scaffold a fresh `NcdReportConfig.yml` from memory.
- If updating an existing config, edit that file in place; do not replace it with a
  brand-new hand-written scaffold.
- Unless unauthenticated access is explicitly requested by user, **always** include an env-based token expression in source configurations
- For `fcli license ncd-report create`, `fcli license ncd-report validate-sources`, and direct SCM REST calls, execute through mandatory [run-cmd-with-scm-auth.md](run-cmd-with-scm-auth.md) workflow.

## Mandatory Workflow

Complete each step before proceeding. Do not skip steps.

### Step 0: Read auth instructions

Load [run-cmd-with-scm-auth.md](run-cmd-with-scm-auth.md). Use this workflow whenever this task needs to run `fcli license ncd-report validate-sources` or direct SCM REST calls.

### Step 1: Create new or update existing config?

Ask the user explicitly whether they want to:
- **Create a new config** => continue to step 1a
- **Update an existing config** => continue to step 1b

#### Step 1a: Create a new config file

If the user chooses **create**:

- Ask the user where to store the new config file. Offer:
  - **Dedicated temp directory** — e.g. `/tmp/ncd-report/NcdReportConfig.yml`
  - **Current working directory** — `./NcdReportConfig.yml`
  - **Custom location** — user specifies path
- If selected config file location already exists, ask user what to do:
  - Overwrite with new sample config -> continue with next item
  - Use as-is -> skip to step 2
- Run the following command to generate the sample config:
  ```bash
  fcli license ncd-report create-config -y -c <path> -o yaml
  ```
  Replace `<path>` with the confirmed path from user response above.
- Continue to Step 2

##### Step 1a gate
- [ ] Config file created at confirmed location

#### Step 1b: Update an existing config file

If the user chooses **update**:

- Run `find . -name "NcdReportConfig.yml"` to locate candidate files.
- If one or more are found:
  - Present them as selectable options.
  - Also offer a **"Specify a different path"** option.
- If none are found, ask the user to provide the path to the existing config.
- Confirm the selected config path before proceeding to Step 2.

##### Step 1b gate
- [ ] Existing config file located and path confirmed

#### Step 1 gate
- [ ] Config file location known (new file created or existing file located)

### Step 2: Review and confirm generic settings

Review the non-source settings in the config (settings unrelated to SCM sources and
repository selection). Common examples:
- Contributor name ignore/exclusion patterns
- Contributor duplicate/alias patterns

Provide a summary of these settings to the user, then ask user whether to update these settings:
- If yes, proceed to Step 2a.
- If no, continue to Step 3.

#### Step 2a: Update generic settings

If the user requests assistance:

- Ask which specific settings need to be changed.
- Provide guidance and assistance with updating them:
  - For new configs, the scaffold contains placeholders; explain what each setting controls.
  - For existing configs being updated (e.g., to improve deduplication/ignore patterns based on findings from a previous report run), explain the change and impact. If applicable, offer to list contributors found in previous report run using `fcli license ncd-report lsc` command and use the output to suggest improvements to ignore & deduplication patterns.
- Update config file based on user input; note that thismay be an iterative process.
- Once satisfied, proceed to Step 3.

#### Step 2 gate
- [ ] Generic (non-source) settings reviewed
- [ ] User confirmed settings are acceptable or changes have been made

### Step 3: Configure SCM sources and repository selection

#### When updating an existing config:
Show the user a summary of the **current** SCM sources and repository selection
criteria configured in the file, then ask user whether to update these settings:
- If yes, continue to step 3a
- If no, continue to step 4

#### When creating a new config:
Ask the user whether they need assistance discovering and configuring SCM sources and repository selection criteria:
- If yes, continue to Step 3a.
- If no, allow them to fill in the details later (mark as TODO in config) and skip to Step 4.

#### Step 3a: Choose the repository selection approach

Read "Report Repository Selection" section from [concepts.md](concepts.md) if not already read before and apply these concepts to guide user in right direction in subsequent steps. Then, ask the user whether they would like to see or revisit the repository selection information:
- If yes → display the section referenced above, then continue to Step 3b
- If no → continue to Step 3b

#### Step 3b: Choose repository selection approach

Ask the user which of these fits their situation:
- **Auto-selection** — select repositories by SCM boundary (org/group/project),
  topics/labels, and/or naming patterns via `repositoryIncludeExpression`.
- **Manual selection** — explicitly list the repositories to include.
- **Combination** — auto-select a base set and amend it with explicit
  includes/excludes.

Route based on the choice:
- **Auto-selection** and the user already knows the target boundary/selectors → they
  may skip to Step 3e and encode it directly. If they are unsure which boundaries,
  topics, or patterns to target, continue to Step 3d to discover them.
- **Manual selection** or **combination** → continue to Step 3d (discovery is needed
  to build the explicit repository list).

##### Step 3b gate
- [ ] Selection approach chosen (auto / manual / combination)

#### Step 3c: Offer discovery techniques reference

Read "Repository Discovery Techniques" section from [concepts.md](concepts.md) if not already read before and apply these concepts to guide user in right direction in subsequent steps. Then, ask the user whether they would like to see or revisit the repository discovery techniques:
- If yes → display the section referenced above, then continue to Step 3d
- If no → continue to Step 3d

##### Step 3c gate
- [ ] User offered reference material; ready to proceed to discovery

#### Step 3d: Assist with repository discovery and selection

Help the user work through repository discovery and selection through a free-format
discussion, guided by the strategy they have chosen or are converging on, applying repository selection and discovery concepts as described in [concepts.md](concepts.md), and considering the 'useful resources & tips' listed below. Adapt to the user's situation; no fixed sub-steps apply here. **Ask individual questions** rather than bundled question sets so answers guide the next question.

Having a complete list of all Fortify-scanned repositories is vital for correct report output; make every effort to guide user in determining exact list of repositories. 

Once a (full or partial) candidate set of repositories has been discovered, check for any indications that this list is incomplete, and/or the list includes non-relevant repositories. Some examples:
- If some repositories show explicit indication that they are scanned with Fortify, for example based on workflow/pipeline names & contents (keywords: fortify, SSC, FoD, fcli), but other repositories don't have a similar indication, then the latter likely shouldn't be included in the report
- Forks, mirrors, and 3rd-party repositories are likely out of scope for NCD reports
- Repositories containing test code only might be out of scope
- Archived/inactive repositories may be in scope if they've ever been scanned with Fortify

Never assume discovered repositories are out of scope. Build a list of potentially out of scope repositories, then use a multiple-choice dialog to display this list of potentially out of scope repositories including reason, asking user to tick each repository that must remain in scope.

**Useful resources & tips:**

- **fcli commands for FoD/SSC** — Query FoD/SSC apps, releases, or application versions filtered by team/department attributes to identify in-scope applications. If SCM repository info is stored in app/release/version or scan attributes, extract it to seed or cross-reference the repository list. Particularly useful in the federated model to scope discovery to the current team/department without needing broad SCM access.
- **fcli license ncd-report validate-sources** command — Given partial sources configuration (for example in temporary config file), can be used to discover available repositories in a given SCM source, and the available repo properties that can be used in `repositoryIncludeExpression`. This requires appropriate sources to be identified/confirmed by user/configured in config file first; run this command through [run-cmd-with-scm-auth.md](run-cmd-with-scm-auth.md).
- **SCM CLI tools (`gh`, `glab`, ...)** — For broader SCM exploration: discovering available organizations, groups, or projects; listing repositories within them; querying topic/label metadata; or enumerating recent pipeline/workflow activity. If CLI tools are not available, offer to install them, or use direct REST calls.
- **Git repositories in the current workspace** — If the user is working in a workspace that contains checked-out repositories, inspect their `remote.origin.url`  values for a cheap, no-auth signal of in-scope repositories and their SCM boundaries. Useful as a starting sample for the heuristic technique or to confirm the SCM host/org/group to target.
- **Multiple SCM systems** — Repositories to be included in the report may live in multiple SCM systems, or multiple organizations/groups on these SCM systems, all of which may need to be explicitly listed.
- **Combine multiple discovery techniques** — discovery techniques can be combined — e.g., discover candidate repositories from Fortify metadata, then confirm them via SCM-native queries.
- **Prefer selector-based scope** — Prefer topics/labels/tags, org/group/project boundaries, naming patterns over large hard-coded repository lists; selectors are more maintainable and adapt as repositories are added or renamed. If not in place yet, offer user to apply appropriate topics/labels/tags to discovered repositories, to allow repository selection based on those rather than explicit repository lists.

##### Step 3d gate
- [ ] Repository scope identified and confirmed with the user

#### Step 3e: Update the config file with discovered scope

Once the repository scope is agreed, write the results into the config file:
- Add or update the SCM source(s) in the config.
- Encode the selection into `repositoryIncludeExpression` (plus minimal
  include/exclude overrides only where a selector cannot express the scope).
- Keep credentials externalized via `#env(...)`; never embed secrets.
- Confirm the updated config reflects the intended reporting scope.

##### Step 3e gate
- [ ] Config file updated with SCM source(s) and repository include expression
- [ ] Credentials are externalized via `#env(...)`; no secrets embedded

#### Step 3f: Optional validation of configured repositories (recommended)

Before proceeding to the final readiness review, offer the user the option to
**validate the configured repository selection** against the actual SCM sources:

Precondition (mandatory if validation is selected): run this command through
[run-cmd-with-scm-auth.md](run-cmd-with-scm-auth.md).

```bash
fcli license ncd-report validate-sources -c <config-path>
```

**Why validate:**
- Confirms that the `repositoryIncludeExpression` and source configurations return
  the expected set of repositories.
- Detects typos, auth issues, or selector mismatches early.
- Allows iterative refinement before running the full report.

**If validation reveals issues:**
- Review the returned repository list (or error messages if auth fails).
- Ask whether to refine the expression/criteria and return to Step 3d, or proceed
  as-is.
- If refinement is needed, update config and revalidate.

**Skip if:**
- User deferred SCM source configuration (marked as TODO in config).
- User explicitly prefers to skip ahead to report generation.

##### Step 3f gate
- [ ] Validation run (or explicitly skipped); config ready for report generation

#### Step 3 gate
- [ ] SCM sources configured (or deferred as TODO for new config)
- [ ] Repository include expression(s) defined and confirmed
- [ ] Optional filtering applied (or confirmed as not needed)
- [ ] Credentials are externalized via `#env(...)`; no secrets embedded
- [ ] Validation completed (or explicitly deferred)

### Step 4: Config readiness review and next steps and next steps

Show the user a summary of the completed (or partially completed) config:
- **Config file path**
- **SCM sources** configured (platform, org/group/project, selector type); note
  if multi-source
- **Repository include expression(s)** and any include/exclude overrides
- **Validation status** (completed, skipped, or pending)
- **Credential references** (env var names only — never show values)
- **Generic settings** (ignore/duplicate patterns, etc.)
- **Incomplete sections** marked as TODO (if applicable for new configs)

Then ask the user what they want to do next:
- **Make further modifications** — return to Step 2 or Step 3 as appropriate
- **Run validation** — if not yet done in Step 3f, run it now to check repository
  scope
- **Stop here** — config is saved; task is complete
- **Proceed to generate report** — continue with [generate-report.md](generate-report.md)

#### Step 4 gate
- [ ] Config summary reviewed and confirmed by user
- [ ] Next action chosen (modify further / validate / stop / proceed to generate)
