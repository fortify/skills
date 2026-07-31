# GitLab CI — Fortify on Demand Integration

## Official documentation

Do **not** rely solely on this file for complete parameter and configuration details. Fetch the relevant URL below on demand when you need the full list of supported environment variables, optional features, or advanced configuration options.

| Need | URL to fetch |
|---|---|
| Full FoD env var reference, all optional features | https://fortify.github.io/fcli/v3/ci/gitlab/v2.0.x/ast-scan-fod.html |
| fcli component for custom workflows | https://fortify.github.io/fcli/v3/ci/gitlab/v2.0.x/fcli-component.html |
| Component index | https://fortify.github.io/fcli/v3/ci/gitlab/v2.0.x/index.html |

Fetch proactively when: the user asks about a specific feature or parameter not covered by the templates below or when the user mentions an option you are unsure about.

## Repository investigation

Before writing a single line of YAML, inspect the repository to gather context.

### 1. Find the existing pipeline file

Look for `.gitlab-ci.yml` at the repo root. If found, read it and extract:

- **Branch filters** under `rules:` or `only:` — use these in the Fortify job rather than template placeholders.
- **`image:`** value used by other jobs — match the runner image type (default `ubuntu:latest` if not found).
- **Existing build/setup steps** — capture the exact Docker image already used (e.g., `maven:3-openjdk-17`). Use the same image for the Fortify scan job so the build environment is consistent.
- **`stages:` list** — the Fortify job must reference a stage already declared. Add a `security` or `test` stage if none is suitable.
- **Any existing Fortify configuration** — if Fortify components or fcli are already referenced, read them to avoid duplicating work.

### 2. Detect build tool from source files (fallback)

Only needed if no existing pipeline gives sufficient context. Check for build manifests (`pom.xml`, `package.json`, `go.mod`, `*.csproj`, `requirements.txt`, etc.) to identify the language and select the correct Docker image. See [Build tool image reference](#build-tool-image-reference).


## How GitLab integration works

The Fortify GitLab integration uses **GitLab Components** (not shell scripts or raw CLI). The `Fortify/components/ast-scan` component is included via `include: component:` and provides two runnable jobs:

- **`<job-name>-prepare`** — downloads and installs fcli
- **`<job-name>`** — runs the AST scan (you customize this job with the Docker image and variables)
- **`<job-name>-publish-sast`** — publishes SAST results to GitLab's security dashboard (automatic)
- **`<job-name>-publish-debug-output`** — optionally publishes debug logs (requires `debug: true`)

Component versioning: use `@1` for the latest v1.x release (recommended), or pin to an exact version like `@1.0.1` for maximum reproducibility.

---

### Gold-standard pipeline template for FoD

```yaml
include:
  - component: $CI_SERVER_FQDN/Fortify/components/ast-scan/linux@1
    inputs:
      job-name: fortify-ast-scan
      stage: test
      fcli-version: v3

stages: [test]   # Add or merge with your existing stages list

fortify-ast-scan:
  image: maven:3-openjdk-17   # Replace with your project's build image — see Build tool image reference
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
  variables:
    FOD_URL: "https://ams.fortify.com"          # Hardcode or use a non-masked CI/CD variable
    FOD_TENANT: $FOD_TENANT                     # Masked CI/CD variable
    FOD_USER: $FOD_USER                         # Masked CI/CD variable
    FOD_PASSWORD: $FOD_PAT                      # Personal Access Token (masked CI/CD variable)
    DO_SCA_SCAN: "true"                         # Enable SCA (open source) scanning
    DO_AVIATOR_AUDIT: "true"                    # Enable AI-assisted audit
    # FOD_RELEASE: MyApp:main                   # Optional: defaults to <group/repo>:<branch>
    # FCLI_BOOTSTRAP_VERSION: v3.23.0           # Optional: pin fcli version for stability
```

### Customisation guidance

- **Docker image**: Replace `maven:3-openjdk-17` with the image appropriate for your build tool. See [Build tool image reference](#build-tool-image-reference). This is a required customization.
- **Rules**: Populate from the existing `.gitlab-ci.yml` branch/MR rules. The template uses standard GitLab rules syntax. `only:` is the older equivalent but `rules:` is preferred.
- **FOD_URL region**: Use the URL matching the user's FoD tenant. `https://ams.fortify.com` is the most common; others include `emea.fortify.com`, `sgp.fortify.com`, `apac.fortify.com`, `fed.fortifygov.com`.
- **FOD_RELEASE**: Only needed if the application/release name in FoD differs from the GitLab group/repo and branch. Format: `AppName:ReleaseName` (non-microservices) or `AppName:MicroserviceName:ReleaseName` (microservices).
- **DO_SETUP**: By default the component will auto-create the FoD application/release if they don't exist. Set `DO_SETUP: "false"` to disable.
- **Additional optional features** (e.g., `DO_PR_COMMENT`, `DO_CHECK_POLICY`, release summary controls): fetch the FoD reference URL from [Official documentation](#official-documentation--fetch-when-needed) for the full list.

---

## Critical rules

These are the common failure points specific to GitLab Fortify integration.

### 1. FoD authentication — prefer user credentials + PAT; API key is a valid alternative

```yaml
# CORRECT (preferred) — user credentials with Personal Access Token
variables:
  FOD_TENANT: $FOD_TENANT
  FOD_USER: $FOD_USER
  FOD_PASSWORD: $FOD_PAT      # Personal Access Token stored as CI/CD variable

# ALSO CORRECT — API key credentials (explicit user request or enterprise preference)
variables:
  FOD_CLIENT_ID: $FOD_CLIENT_ID
  FOD_CLIENT_SECRET: $FOD_CLIENT_SECRET

# WRONG — never combine both credential sets
variables:
  FOD_TENANT: $FOD_TENANT
  FOD_USER: $FOD_USER
  FOD_PASSWORD: $FOD_PAT
  FOD_CLIENT_ID: $FOD_CLIENT_ID       # Do not include both
  FOD_CLIENT_SECRET: $FOD_CLIENT_SECRET
```

> **Note**: The official GitLab quick-start example uses API key credentials. Our guidance defaults to user+PAT for consistency with FoD PAT auth best practices, but both are fully supported.

### 2. FOD_URL — never mask as secrets

GitLab allows variables to be marked as **Masked** (hidden in logs) or **Protected** (only available on protected branches). FOD_URL is **not sensitive** and must be rendered as plain text by the CI/CD system.

```yaml
# CORRECT — hardcoded (simplest, recommended for most cases)
variables:
  FOD_URL: "https://ams.fortify.com"

# ALSO CORRECT — non-masked CI/CD variable
variables:
  FOD_URL: $FOD_URL    # Stored as a non-masked variable in Settings → CI/CD → Variables
```

A masked variable would prevent the component from reading the URL and fail authentication.

### 3. All scan configuration goes in the job's `variables:` block, not component `inputs:`

```yaml
# CORRECT — scan configuration in variables: on the main job
fortify-ast-scan:
  variables:
    FOD_URL: "https://ams.fortify.com"
    FOD_TENANT: $FOD_TENANT
    DO_SCA_SCAN: "true"

# WRONG — these are not valid component inputs
include:
  - component: $CI_SERVER_FQDN/Fortify/components/ast-scan/linux@1
    inputs:
      FOD_URL: "https://ams.fortify.com"    # Not valid — env vars go on the job, not inputs
      DO_SCA_SCAN: "true"                   # Not valid
```

### 4. Component `inputs:` accepts only the defined input parameters

Valid `inputs:` are: `job-name`, `stage`, `fcli-version`, `reports-access`, `logs-access`, `logs-expire-in`, `debug`, `alpine-image`. Everything else belongs in the job's `variables:`.

### 5. The Docker image is required and must provide your build toolchain

The component does not assume any build environment. If you omit `image:`, the job may run without the tools needed to package your source code and the scan will fail.

### 6. Do not override `needs:` or `extends:` without preserving originals

If you customize `needs:` or `extends:` on the main job, you must retain the original component values:

```yaml
fortify-ast-scan:
  extends:
    - .fortify-ast-scan-vars-ci    # Required: keep the component-provided extends
    - .my-org-shared-config        # Optional: add organization-wide settings
  needs:
    - fortify-ast-scan-prepare     # Required: keep the prepare job dependency
    - some-other-job               # Optional: add additional dependencies
```

---

## Build tool image reference

Select the Docker image that provides the build tools for your project. Use whatever image is already specified in other jobs in the pipeline; refer to this table only as a fallback.

| Language / Build tool | Detected by | Recommended Docker image |
|---|---|---|
| Java (Maven) | `pom.xml` | `maven:3-openjdk-21` |
| Java (Gradle) | `build.gradle`, `build.gradle.kts` | `gradle:8-jdk21` |
| .NET / MSBuild | `*.csproj`, `*.sln` | `mcr.microsoft.com/dotnet/sdk:9.0` |
| Node.js | `package.json` | `node:22` |
| Python | `requirements.txt`, `pyproject.toml` | `python:3.12` |
| Go | `go.mod` | `golang:1.23` |
| Generic / multi-language | — | `ubuntu:latest` (add tool install steps) |

**Version selection priority:**
1. Use the image already specified in an existing `.gitlab-ci.yml` job.
2. Use the version specified in a runtime/manifest file (e.g., `.java-version`, `.nvmrc`, `<java.version>` in `pom.xml`).
3. Fall back to a current LTS/stable image and note the assumption in a comment.

Use official Docker Hub or registry images. For air-gapped environments, replace with an internal mirror image.

---

## Component inputs reference

These inputs are specified in the `include: component:` block, not in the job's `variables:`.

| Input | Description | Default |
|---|---|---|
| `job-name` | Name for all component-provided jobs. Must be a valid filesystem path name. | `fortify-ast-scan` |
| `stage` | Pipeline stage in which to run AST scan jobs. Must be declared in `stages:`. | `test` |
| `fcli-version` | fcli version to use. Supports semver ranges (e.g., `v3`, `v3.15`). Avoid overriding unless pinning for stability. | `v3` |
| `reports-access` | GitLab access level for SAST report artifacts. Values: `all`, `developer`, `none`. | `developer` |
| `logs-access` | GitLab access level for debug log artifacts. Values: `all`, `developer`, `none`. | `developer` |
| `logs-expire-in` | Artifact expiry for log artifacts (sensitive data). | `1 day` |
| `debug` | Enable collection of debug logs (may include sensitive data like tokens). | `false` |
| `alpine-image` | Override the Alpine image used for prepare/publish jobs (e.g., for internal registries). | `alpine:latest` |

---

## Variables configuration

Tell the user to configure the following in **Settings → CI/CD → Variables**.

**User credentials + PAT (recommended):**

| Variable | Masked | Protected | Value |
|---|---|---|---|
| `FOD_TENANT` | Yes | No | FoD tenant name |
| `FOD_USER` | Yes | No | Tenant user login |
| `FOD_PAT` | Yes | No | Personal Access Token |
| `FOD_URL` | **No** | No | e.g., `https://ams.fortify.com` (NOT masked — must render as plain text) |

**API key credentials:**

| Variable | Masked | Protected | Value |
|---|---|---|---|
| `FOD_CLIENT_ID` | Yes | No | API key Client ID |
| `FOD_CLIENT_SECRET` | Yes | No | API key Client Secret |
| `FOD_URL` | **No** | No | e.g., `https://ams.fortify.com` (NOT masked) |
