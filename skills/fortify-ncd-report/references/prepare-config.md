# Task: Prepare an NCD Report Config

Use this workflow to **create a new `NcdReportConfig.yml`** or **update an existing
one**. This config is the input to the `fcli license ncd-report create` command. To
run a report from a ready config, use [generate-report.md](generate-report.md).

> **This workflow operates on `NcdReportConfig.yml` — the report generation config
> file.** It does not amend contributor data in a generated report artifact. For
> that, use [review-amend-report.md](review-amend-report.md).

## Non-negotiable rules

- Apply non-negotiable rules from [../SKILL.md](../SKILL.md)
- For new config creation, **must** run:
  `fcli license ncd-report create-config -y -c <path> -o yaml`.
  Do **not** hand-scaffold a fresh `NcdReportConfig.yml` from memory.
- If updating an existing config, edit that file in place; do not replace it with a
  brand-new hand-written scaffold.
- Always ask the user where to store a new config file; never write to an arbitrary
  location.

## Overall process

1. Determine whether to create a new config or update an existing one.
2. For new configs: confirm storage location, then generate the scaffold.
3. Discover repositories and author or update the config content.
4. Review the config summary and confirm readiness.

## Step 1: Create new or update existing config?

Ask the user explicitly whether they want to:
- **Create a new config** — no existing config will be used as a starting point.
- **Update an existing config** — start from an existing `NcdReportConfig.yml`.

If the user chooses **update**:
- Run `find . -name "NcdReportConfig.yml"` to locate candidate files.
- If one or more are found, present them as selectable options, plus a **"Specify a
  different path"** option.
- If none are found, ask the user to provide the path to the existing config.
- Confirm the selected config path before proceeding.

If the user chooses **create**, continue to Step 2.

### Step 1 gate
- [ ] User confirmed: create new or update existing
- [ ] If update: config file path confirmed

## Step 2: Choose config file location (new config only)

Skip this step if updating an existing config.

Ask the user where to store the new config file. Offer:
- **Dedicated temp directory** — e.g. `/tmp/ncd-report/NcdReportConfig.yml`
- **Current working directory** — `./NcdReportConfig.yml`
- **Custom location** — user specifies path

Confirm the chosen path before proceeding.

### Step 2 gate
- [ ] Config file path confirmed (new config only)

## Step 3: Discover repositories and author the config

Load [discovery-and-config.md](discovery-and-config.md) and follow it to discover
in-scope repositories and produce or update `NcdReportConfig.yml`. Return here once
the discovery + config gate in that file passes.

**Scope note.** The config's `repositoryIncludeExpression` defines which repositories
count toward the NCD total. In a **federated model**, each producer's config should
cover only their team/department's repositories; in a **single-run model**, the config
covers the full organization. The execution model does not need to be confirmed
explicitly, but the repository selection must reflect the intended reporting scope —
choose selectors accordingly.

**Fortify platform comparison.** If the user wants to cross-reference discovered repos
against FoD or SSC app inventory, check active sessions
(`fcli fod session ls --query "expired=='No'"` /
`fcli ssc session ls --query "expired=='No'"`). Use whichever platform is active, or
ask if both are active. Skip comparison if neither is active and the user does not
request it.

### Step 3 gate
- [ ] Discovery + config gate from [discovery-and-config.md](discovery-and-config.md) passed

## Step 4: Config readiness review

Show the user a summary of the completed config:
- Config file path
- SCM sources configured (platform, org/group/project, selector type)
- Repository include expression(s)
- Credential references (env var names only — never show values)
- Contributor ignore/duplicate expressions

Then ask the user what they want to do next:
- **Make further modifications** — return to Step 3
- **Stop here** — config is saved; task is complete
- **Proceed to generate report** — continue with [generate-report.md](generate-report.md)

### Step 4 gate
- [ ] Config summary reviewed and confirmed by user
- [ ] Next action chosen (modify further / stop / proceed to generate)
