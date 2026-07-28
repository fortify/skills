# GitLab CI — Fortify Application Security Center (SSC) Integration

## Official documentation — fetch when needed

Do **not** rely solely on this file for complete parameter and configuration details. Fetch the relevant URL below on demand when you need the full list of supported environment variables, optional features, or advanced configuration options.

| Need | URL to fetch |
|---|---|
| Full SSC env var reference, all optional features | https://fortify.github.io/fcli/v3/ci/gitlab/v2.0.x/ast-scan-ssc.html |
| fcli component for custom workflows | https://fortify.github.io/fcli/v3/ci/gitlab/v2.0.x/fcli-component.html |
| Component index | https://fortify.github.io/fcli/v3/ci/gitlab/v2.0.x/index.html |

Fetch proactively when: the user asks about a specific feature or parameter not covered by the templates below or when the user mentions an option you are unsure about.

---

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

---

## How GitLab integration works

The Fortify GitLab integration uses **GitLab Components** (not shell scripts or raw CLI). The `Fortify/components/ast-scan` component is included via `include: component:` and provides two runnable jobs:

- **`<job-name>-prepare`** — downloads and installs fcli
- **`<job-name>`** — runs the AST scan (you customize this job with the Docker image and variables)
- **`<job-name>-publish-sast`** — publishes SAST results to GitLab's security dashboard (automatic)
- **`<job-name>-publish-debug-output`** — optionally publishes debug logs (requires `debug: true`)

Component versioning: use `@1` for the latest v1.x release (recommended), or pin to an exact version like `@1.0.1` for maximum reproducibility.


---

## Gold-standard pipeline template for SSC

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
    SSC_URL: $SSC_URL                           # Non-masked CI/CD variable (not sensitive)
    SSC_TOKEN: $SSC_TOKEN                       # Masked CI/CD variable — CIToken (unified login token)
    SC_SAST_TOKEN: $SC_SAST_TOKEN               # Masked CI/CD variable — ScanCentral SAST sensor token
    # SSC_APPVERSION: MyApp:main                # Optional: defaults to <group/repo>:<branch>
    # FCLI_BOOTSTRAP_VERSION: v3.23.0           # Optional: pin fcli version for stability
```

For the full list of SSC env vars and optional features (Aviator, Debricked SCA, policy checks, MR comments), fetch the SSC reference URL from [Official documentation](#official-documentation--fetch-when-needed) before generating.

### SSC-specific notes

- **Debricked SCA**: Set `DEBRICKED_ACCESS_TOKEN` to enable open source scanning alongside SAST. Unlike FoD, SSC SCA is not enabled via `DO_SCA_SCAN` — it uses `DO_DEBRICKED_SCAN` requires the Debricked token to be present.
- **Aviator** (SSC): Requires both `AVIATOR_URL` and `AVIATOR_TOKEN` to be configured. Set `DO_AVIATOR_AUDIT: "false"` to disable if configured but not desired for a specific pipeline.

---

## Critical rules

These are the common failure points specific to GitLab Fortify integration.

### 1. SSC_URL — never mask as secrets

GitLab allows variables to be marked as **Masked** (hidden in logs) or **Protected** (only available on protected branches). SSC_URL is **not sensitive** and must be rendered as plain text by the CI/CD system.

```yaml
# CORRECT — hardcoded (simplest, recommended for most cases)
variables:
  SSC_URL: "https://fortify.example.com/ssc"

# ALSO CORRECT — non-masked CI/CD variable
variables:
  SSC_URL: $SSC_URL    # Stored as a non-masked variable in Settings → CI/CD → Variables
```

A masked variable would prevent the component from reading the URL and fail authentication.

### 2. All scan configuration goes in the job's `variables:` block, not component `inputs:`

```yaml
# CORRECT — scan configuration in variables: on the main job
fortify-ast-scan:
  variables:
    SSC_URL: "https://fortify.example.com/ssc"
    DO_DEBRICKED_SCAN: "true"

# WRONG — these are not valid component inputs
include:
  - component: $CI_SERVER_FQDN/Fortify/components/ast-scan/linux@1
    inputs:
      SSC_URL: "https://fortify.example.com/ssc"    # Not valid — env vars go on the job, not inputs
      DO_DEBRICKED_SCAN: "true"                   # Not valid
```

### 3. Component `inputs:` accepts only the defined input parameters

Valid `inputs:` are: `job-name`, `stage`, `fcli-version`, `reports-access`, `logs-access`, `logs-expire-in`, `debug`, `alpine-image`. Everything else belongs in the job's `variables:`.

### 4. The Docker image is required and must provide your build toolchain

The component does not assume any build environment. If you omit `image:`, the job may run without the tools needed to package your source code and the scan will fail.

### 5. Do not override `needs:` or `extends:` without preserving originals

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

| Variable | Masked | Protected | Value |
|---|---|---|---|
| `SSC_TOKEN` | Yes | No | CIToken (unified login token) |
| `SC_SAST_TOKEN` | Yes | No | ScanCentral SAST sensor token |
| `SSC_URL` | **No** | No | SSC server URL (NOT masked — must render as plain text) |
