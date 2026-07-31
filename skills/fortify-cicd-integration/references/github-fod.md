# GitHub Actions — Fortify Integration

## Contents
- [Official documentation — fetch when needed](#official-documentation--fetch-when-needed)
- [Repository investigation](#repository-investigation)
- [FoD: Fortify on Demand](#fod-fortify-on-demand)
- [Critical rules](#critical-rules)
- [Build tool setup reference](#build-tool-setup-reference)
- [Secrets configuration](#secrets-configuration)

---

## Official documentation — fetch when needed

Do **not** rely solely on this file for complete parameter and configuration details. Fetch the relevant URL below on demand whenever you need the full list of supported `env:` variables, optional features, or advanced configuration options. This avoids duplicating documentation that may change independently of this skill.

| Need | URL to fetch |
|---|---|
| Full FoD env var reference, all optional features | https://fortify.github.io/fcli/v3/ci/github/v3.0.x/ast-action-fod.html |
| Action overview, quick-start examples, custom workflows | https://github.com/marketplace/actions/fortify-ast-scan |

Fetch proactively when: the user asks about a specific feature or parameter not covered by the templates below, when generating an FoD workflow, or when the user mentions an option you are unsure about.

---

## Repository investigation

Before writing a single line of YAML, read the repository. Use the findings to populate the generated workflow with real values, not placeholders.

### 1. Find existing workflows

List `.github/workflows/` and read every YAML file there. Extract:

- **Branch names** under `on.push.branches` and `on.pull_request.branches` — use these directly in the Fortify workflow instead of template variables.
- **`runs-on:`** value — match the existing runner OS (default `ubuntu-latest` if not found).
- **Existing build/setup steps** — capture the exact `uses:` line and any `with:` parameters (e.g., `actions/setup-python@v6` + `python-version: '3.12'`). Reproduce these verbatim in the Fortify workflow so the build environment is identical to other pipelines in the repo.
- **Existing `permissions:` blocks** — note what is already granted; the Fortify workflow needs at minimum `actions: read`, `contents: read`, `security-events: write`.
- **Any existing Fortify steps** — if a workflow already contains `fortify/github-action` or `fcli`, read it to understand what is already configured.

### 2. Detect build tool from source files (fallback)

Only needed if existing workflows have no setup steps. Check the repo root for build manifests — see [Build tool setup reference](#build-tool-setup-reference).

---

## FoD: Fortify on Demand

### Gold-standard workflow template

```yaml
name: OpenText Fortify AST Scan

on:
  push:
    branches: [ $default-branch, $protected-branches ]
  pull_request:
    branches: [ $default-branch ]
  workflow_dispatch:

jobs:
  Fortify-AST-Scan:
    runs-on: ubuntu-latest
    permissions:
      actions: read
      contents: read
      security-events: write

    steps:
      - name: Check Out Source Code
        uses: actions/checkout@v6

      # Replace with the correct setup action for the project's build tool (see Build tool setup reference below, example provided for Go as a reference)
      - name: Set up build tools
        uses: actions/setup-go@v6
        with:
          go-version: '1.23'

      - name: Run Fortify Scan
        uses: fortify/github-action@v3
        env:
          FOD_URL: https://ams.fortify.com          # Hardcode or use vars.FOD_URL — NEVER a secret
          FOD_TENANT: ${{ secrets.FOD_TENANT }}     # Tenant name
          FOD_USER: ${{ secrets.FOD_USER }}         # Tenant user login
          FOD_PASSWORD: ${{ secrets.FOD_PAT }}      # Personal Access Token (preferred over API client)
          DO_SCA_SCAN: true                         # Enable SCA (open source) scanning
          DO_AVIATOR_AUDIT: true                    # Enable AI-assisted audit (Aviator)
          # FOD_RELEASE: MyApp:main                 # Optional: defaults to <repo>:<branch>
          # FCLI_BOOTSTRAP_VERSION: v3.23.0         # Optional: pin for stability
```

### Customisation guidance

- **Branches**: Populate from the investigation findings (see [Repository investigation](#repository-investigation)). It's OK to leave `$default-branch` or `$protected-branches`.
- **Build tool step**: Replace `actions/setup-python@v6` / `python-version: '3.10'` with the setup action and exact version detected from the repo (see [Build tool setup reference](#build-tool-setup-reference)). Remove this step entirely only if the project has no build dependencies to install.
- **FOD_URL region**: Use the URL matching the user's FoD tenant. https://ams.fortify.com is the most popular.  Other options include emea.fortify.com, sgp.fortify.com, apac.fortify.com and fed.fortifygov.com.
- **FOD_RELEASE**: Only needed if the application/release name in FoD differs from the GitHub repo/branch. Format: `AppName:ReleaseName`.
- **Additional optional parameters** (e.g., `DO_SCA_SCAN`, `RUN_AVIATOR_AUDIT`, `DO_PR_COMMENT`, ): fetch the FoD reference URL from [Official documentation](#official-documentation--fetch-when-needed) for the full list.

---

## Critical rules

These are the common failure points observed across AI-generated Fortify GitHub workflows.

### 1. FoD authentication — use tenant user + PAT, not API client credentials unless the user explicitly asks to use API client credentials (this is ok)

```yaml
# CORRECT — preferred auth method
env:
  FOD_TENANT: ${{ secrets.FOD_TENANT }}
  FOD_USER: ${{ secrets.FOD_USER }}
  FOD_PASSWORD: ${{ secrets.FOD_PAT }}   # Personal Access Token

# WRONG — API client credentials are valid but NOT the preferred method.
# Do not default to these unless the user explicitly requests them.
env:
  FOD_CLIENT_ID: ${{ secrets.FOD_CLIENT_ID }}
  FOD_CLIENT_SECRET: ${{ secrets.FOD_CLIENT_SECRET }}
```

### 2. FOD_URL — never place in secrets

```yaml
# CORRECT — hardcoded (preferred by gold standard)
FOD_URL: https://ams.fortify.com

# ALSO ACCEPTABLE — repository variable (not secret)
FOD_URL: ${{ vars.FOD_URL }}

# WRONG — FOD_URL is not sensitive; placing it in secrets violates Fortify guidance
FOD_URL: ${{ secrets.FOD_URL }}
```

### 3. Permissions block — all three entries required for FoD

```yaml
permissions:
  actions: read           # Required — do not omit
  contents: read          # Required
  security-events: write  # Required for GitHub Security tab integration
```

### 4. Do not confuse `with:` parameters (v1 style) with `env:` configuration (v3 style)

```yaml
# CORRECT (v3) — configuration goes in env:
- uses: fortify/github-action@v3
  env:
    FOD_URL: https://ams.fortify.com
    ...

# WRONG — v1 syntax, incompatible with @v3
- uses: fortify/github-action@v3
  with:
    sast-scan: true
```

---

## Build tool setup reference

Derive the setup step from the existing workflow files or source manifests (see [Repository investigation](#repository-investigation)). Do not guess or default to Java.  Use the latest version of the setup action.  Here is a representative list of what to consider:

| Language / Build tool | Detected by | GitHub setup action | Version param |
|---|---|---|---|
| Python | `requirements.txt`, `pyproject.toml`, `setup.py`, `Pipfile`, `poetry.lock` | `actions/setup-python@v6` | `python-version:` |
| Java (Maven) | `pom.xml` | `actions/setup-java@v4` | `java-version:`, `distribution: 'temurin'` |
| Java (Gradle) | `build.gradle`, `build.gradle.kts` | `actions/setup-java@v4` | `java-version:`, `distribution: 'temurin'` |
| .NET / MSBuild | `*.csproj`, `*.sln`, `global.json` | `actions/setup-dotnet@v4` | `dotnet-version:` |
| Node.js | `package.json` | `actions/setup-node@v4` | `node-version:` |
| Go | `go.mod` | `actions/setup-go@v6` | `go-version:` |

**Version selection priority:**
1. Use the version already specified in an existing workflow file in the repo.
2. Use the version already specified in a runtime/manifest file (e.g., `.python-version`, `.nvmrc`, `<java.version>` in `pom.xml`).
3. Fall back to a current LTS/stable default and note the assumption in a comment.

**Do not add `setup-java` for non-Java projects.** A common AI error is unconditionally including Java setup regardless of the project language.

---

## Secrets configuration

Tell the user to configure the following in **Settings → Secrets and variables → Actions** (PAT auth — recommended):

| Secret / Variable | Type | Value |
|---|---|---|
| `FOD_TENANT` | Secret | FoD tenant name |
| `FOD_USER` | Secret | Tenant user login |
| `FOD_PAT` | Secret | Personal Access Token |
| `FOD_URL` | Variable (not secret) | e.g., `https://ams.fortify.com` |
