# Repository Discovery and Config Authoring

Load this when you need to decide which repositories belong in the reporting domain
and how to encode that selection in `NcdReportConfig.yml`.

## Step 1: Choose a discovery approach

Before crawling anything, ask which approach the user prefers and make the cost
explicit:

1. **AI-assisted discovery** (thorough, slower) — inspect repositories, CI/CD
   definitions, and metadata to infer which repos are Fortify-scanned.
2. **User-provided selectors** (quick) — the user supplies repo names, topics/labels,
   organizations, groups, or projects; you encode them directly.
3. **Fortify-stored metadata** (if available) — repo URLs already stored in FoD/SSC
   attributes.
4. **Skeleton only** (fastest) — generate a minimal config and tell the user exactly
   which sections to complete manually.

Only do the crawling work for option 1. For the others, skip to config authoring and
encode what the user or Fortify already knows.

Always infer and show what you found before asking for confirmation. Ask for a manual
repo list only when local workspace discovery, SCM CLI/API discovery, and
Fortify-stored metadata are unavailable or inconclusive.

## Step 2 (option 1): AI-assisted discovery heuristics

### CI/CD scan markers

Search repositories' pipeline definitions for Fortify markers:

- GitHub Actions workflows containing `fortify`, `fcli`, `sourceanalyzer`,
  `scancentral`, `FoD`, or `SSC`
- GitLab CI jobs / included templates with the same markers
- Azure DevOps pipelines with Fortify tasks, `fcli`, `sourceanalyzer`, or `scancentral`
- Jenkinsfiles or shared libraries invoking Fortify tooling

If working inside a code workspace, search workflow files for these markers before
asking the user for a manual list. Local git discovery is fastest:

```bash
find <workspace> -name ".git" -type d
git config --file <repo>/.git/config --get remote.origin.url
```

### Prefer metadata-driven selection

Recommend repository **topics** that mark Fortify-scanned repos (GitHub/GitLab), e.g.
`scanned-by-fortify`, `fortify-integration`, `fortify-ssc`, `fortify-fod`. Encode the
topic in `repositoryIncludeExpression`:

```yaml
repositoryIncludeExpression: >
  topics.contains("scanned-by-fortify")
```

If topics aren't used consistently, fall back to explicit org/group/project scoping,
repo name patterns, or a curated include list. For **Azure DevOps**, prefer
project-level scoping plus name filters — repo topics aren't typically available.

## Step 2 (option 3): Fortify-stored metadata

Ask whether repo URLs are already stored in Fortify before crawling SCM:

- **FoD** — ask whether the attribute is on the **application** or **release** record.
- **SSC** — assume the attribute is on the **application version** record.

If the user knows the attribute name, use it. If not, but a session is active, inspect
one representative app/release/application version for candidate attribute names or
values containing `repo`, `repository`, `git`, `scm`, `url`, or values that look like
GitHub/GitLab/Azure DevOps URLs. **Always confirm the candidate attribute with the
user** before relying on it. Prefer reliable Fortify metadata over SCM crawling.

## Step 3 (optional): Compare against Fortify inventory

If the user wants to reconcile discovered repos against Fortify apps:

```bash
fcli fod app ls -o json   # FoD
fcli ssc app ls -o json   # SSC
```

Normalize names before comparing (app names often differ from repo names by prefix,
suffix, or environment label). Watch for mismatch categories:

- repo scanned in CI but no matching FoD/SSC app
- FoD/SSC app with no active repo mapping
- multiple repos → one app (monorepo or naming drift)
- one repo → multiple apps (branches, microservices, migration history)

When mismatches appear, ask whether the report should follow repository reality,
Fortify app inventory, or a curated business-owned list. If it's unclear whether the
customer uses FoD or SSC, ask — don't assume both. For deeper analysis, activate the
`fortify-fod` or `fortify-ssc` skill.

## Step 4: Generate the config scaffold

If adjusting an existing config, treat it as the source artifact and change only what's needed. Otherwise generate a stub:

```bash
fcli license ncd-report create-config -y -c NcdReportConfig.yml -o yaml
```

This command is mandatory for new configs. Do not hand-write a brand-new
`NcdReportConfig.yml` from scratch. For "skeleton only", still run this command, then
leave explicit TODO markers in the generated structure.

Then pick the lightest authoring path:

- **AI-assisted** — encode the repo selection discovered in Steps 2–3.
- **User-provided selectors** — convert names/topics/orgs/groups/projects into
  `repositoryIncludeExpression` and source selectors.
- **Fortify-stored metadata** — convert confirmed attribute values into selectors,
  name filters, or a manual include list.
- **Skeleton only** — prefill platform, org/group/project, and credential settings,
  then mark clearly which sections the user must finish.

### Authoring rules

- Treat the generated scaffold as the source of truth for structure/field names.
- Keep credentials externalized via `#env("...")` — never embed secrets.
- Use `repositoryIncludeExpression` to encode the selection rule.
- Keep `ignoreExpression` and `duplicateExpression` readable and multiline.
- Prefer org/group/project selectors over hand-maintained repo lists when metadata is
  reliable.
- Stay close to the generated structure unless there's a strong reason to restructure.

### Source examples

GitHub:

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

GitLab:

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

Azure DevOps:

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

### Contributor expressions

The generated sample includes practical defaults — preserve or refine them rather
than replacing them with opaque logic:

```yaml
contributor:
  ignoreExpression: >
    lcName matches '.*\[bot\]'
  duplicateExpression: >
    a1.cleanName==a2.cleanName ||
    a1.cleanEmailName==a2.cleanEmailName ||
    a1.cleanName==a2.cleanEmailName
```

## Step 5: SCM credential handling

Load [scm-credential-handling.md](scm-credential-handling.md) and follow it to acquire
and verify SCM authentication tokens for config discovery and repository access.

## Discovery + config gate

- [ ] Discovery method chosen (AI-assisted, user-provided, Fortify metadata, skeleton)
- [ ] Repo selection logic encoded in `repositoryIncludeExpression` (or manual
      sections clearly marked)
- [ ] If Fortify metadata was used: attribute scope and name confirmed with user
- [ ] Credentials externalized via `#env(...)`
- [ ] Credential handling gate from [scm-credential-handling.md](scm-credential-handling.md) passed
