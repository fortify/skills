## Use Case: Starting a Scan

The user wants to scan their code with Fortify (e.g., "scan my code", "run a SAST scan", "check my code for vulnerabilities", "run an open source scan").

> If the release name is not already resolved, load `references/resolving-release.md` first.

> **Important**: Only start a scan when the user explicitly requests one. Fortify scans are comprehensive and resource-intensive — they should not be initiated automatically on every code change or as a side effect of another task.

---

### Step 1 — Determine the Scan Type

Ask the user which type of scan they want to run (or infer from context if obvious):

| Scan type | Description |
|-----------|-------------|
| **SAST** | Static code analysis — finds vulnerabilities in source code |
| **SCA (open source)** | Scans open source dependencies for known CVEs and license risk |
| **SAST + SCA** | Both of the above in one submission |
| **DAST** | Dynamic scan of a running application URL |

**Language check for SAST**: If the project uses C, C++, Scala, Objective-C, or Swift, inform the user that these languages require **local translation** by Fortify before submission. This is more complex than standard scanning. Direct them to the [FoD User Guide](https://www.microfocus.com/documentation/fortify-on-demand/) for instructions on local translation workflows. Do not proceed with automated packaging for these languages.

---

### Step 2 — Determine How the User Wants to Run the Scan

Before running any fcli commands, gather context to present informed options:

**2a. Detect the IDE context**

If the assistant is running inside an IDE (e.g., GitHub Copilot in VS Code, Cursor, JetBrains AI), that context is already known — use it directly rather than trying to detect it via CLI. Check whether a Fortify extension is installed by inspecting the extensions directory on disk (do **not** invoke `code --list-extensions`; it opens a new GUI window and does not return usable output in agent contexts):

- **VS Code**: list `~/.vscode/extensions/` (or `%USERPROFILE%\.vscode\extensions\` on Windows) and look for a directory matching `fortify*`
  ```bash
  # Unix/macOS
  ls ~/.vscode/extensions/ | grep -i fortify
  # Windows (PowerShell)
  Get-ChildItem "$env:USERPROFILE\.vscode\extensions" | Where-Object Name -like "*fortify*"
  ```
- **IntelliJ/JetBrains**: look for `fortify` in `~/.local/share/JetBrains/` or the IDE-specific plugins directory
- **Visual Studio**: check `%USERPROFILE%\AppData\Local\Microsoft\VisualStudio\` for Fortify extension folders
- **Eclipse**: look for `fortify` in the Eclipse `dropins` or `plugins` directory

**2b. Check for existing CI/CD integration**

Scan the workspace for pipeline files that may already include Fortify scanning. This is a representative list — other CI/CD-as-code tools (Travis CI, CircleCI, TeamCity, Bamboo, Bitbucket Pipelines, etc.) may also be present:
```
.github/workflows/*.yml       # GitHub Actions
.gitlab-ci.yml                # GitLab CI
Jenkinsfile                   # Jenkins
azure-pipelines.yml           # Azure DevOps
.travis.yml                   # Travis CI
.circleci/config.yml          # CircleCI
.teamcity/                    # TeamCity
bamboo-specs/                 # Bamboo
bitbucket-pipelines.yml       # Bitbucket Pipelines
```
Open any found files and check whether Fortify on Demand scanning steps are already configured (look for `fortify`, `fod`, `scancentral`, or `debricked` keywords).

**1c. Get the FoD tenant URL and release ID (for the web UI option)**

```bash
fcli fod session ls -o json
# → use the `url` field from the active session; this is the API URL (e.g. https://api.ams.fortify.com)
# Derive the portal base URL by removing the `api.` prefix from the hostname:
#   https://api.ams.fortify.com  →  https://ams.fortify.com
#   https://api.emea.fortify.com →  https://emea.fortify.com
# Use this derived portal URL (referred to as <fodPortalUrl>) to construct all portal links.
```

If the release has already been resolved, you will have the numeric release ID needed to construct direct portal links (see Step 2).

**1d. Present the options**

Based on the above context, present these options with relevant context highlighted:

| Option | When to suggest |
|--------|-----------------|
| **Web UI / Portal** | Always available. Good for ad-hoc scans. Provide the FoD portal link. |
| **CI/CD pipeline** | If pipeline files found — describe how to trigger existing integration. If no pipeline — offer to set one up using the `fortify-cicd-integration` skill. |
| **IDE plugin** | If plugin detected — point the user at it directly. If not detected — provide marketplace link for their IDE. SAST and SCA only, does not support DAST scans. |
| **CLI/agent (fcli)** | Recommended when the user explicitly wants to run a scan right now from the terminal or wants the agent to execute it end-to-end. |

Ask the user to confirm their preferred method before proceeding.

---

### Step 3 — Route Based on Chosen Method

**Web UI**: Direct the user to their release overview page to submit a scan:
```
<fodPortalUrl>/Releases/<releaseId>/Overview
```
Replace `<releaseId>` with the numeric release ID. No further action needed.

**CI/CD pipeline**:
- If Fortify is already configured in the pipeline: describe the trigger conditions (push to main, pull request, manual dispatch, etc.) and instruct the user to trigger accordingly.
- If not yet configured: offer to set it up by invoking the `fortify-cicd-integration` skill.

**IDE plugin**:
- If already installed: direct the user to the Fortify extension/plugin panel and describe how to initiate a scan from within the IDE.
- If not installed: provide the marketplace link for their IDE:
  - VS Code: https://marketplace.visualstudio.com/items?itemName=fortifyvsts.fortify-extension-for-vs-code
  - IntelliJ/JetBrains: https://plugins.jetbrains.com/plugin/9943-opentext-core-application-security
  - Visual Studio: https://marketplace.visualstudio.com/items?itemName=fortifyvsts.HPESecurityFortifyonDemandExtension-18282
  - Eclipse: https://marketplace.eclipse.org/content/hpe-security-fortify-demand-plugin

**CLI/agent**: continue to Step 4.

---

### Step 4 — Ask About Async vs. Wait-for (SAST and SCA only)

For SAST and SCA scans, ask whether the user wants to:
- **Start and return immediately** (async) — recommended for most cases
- **Wait for completion** — only if the user has previously scanned this app and knows it completes quickly

Highlight: while many scans complete in a few minutes, larger codebases can take significantly longer. Only offer wait-for if the user expects a short scan.

DAST scans always run asynchronously — do not offer wait-for for DAST.

---

### Step 5 — Verify Branch/Release Alignment

Run all commands below **before** prompting the user — collect all alignment information in one pass, then surface any issues in a single confirmation.

**Collect alignment data (run silently)**

```bash
# Current branch
git branch --show-current

# All releases for this app — compare names against the branch list
fcli fod release list --query "applicationName=='<appName>'" \
  -o table --columns releaseName,releaseId,sdlcStatusType,isPassed

# All local/remote branches — compare against the release list
git branch -a --format='%(refname:short)' | sed 's|^origin/||' | sort -u
```

**Evaluate findings**

| Finding | Treatment |
|---------|-----------|
| Release resolved from a user-supplied numeric ID | ✅ Skip all checks — explicit choice made. Proceed to Step 6. |
| `git branch` returned empty (detached HEAD / no git context) | ✅ No comparison possible. Proceed to Step 6. |
| Current branch **exactly matches** the resolved release name | ✅ Aligned. Proceed to Step 6. |
| Current branch **does not match** the resolved release name | ⚠️ Include in prompt |
| Another FoD release exists whose name matches the current branch | ⚠️ Include in prompt |
| Other git branches match other FoD release names (not the resolved target) | ℹ️ Mention alongside the prompt if one is required; otherwise note briefly when presenting Step 6 |

**Prompt the user (at most once)**

If all findings are ✅, proceed to Step 6 without prompting.

If any ⚠️ findings exist, send **one** message covering all of them:

> ⚠️ **Release alignment — please confirm before proceeding**
>
> - Current git branch: **`<currentBranch>`**
> - Target FoD release: **`<resolvedReleaseName>`**
> - [if names differ] These do not match. If you meant to scan `<currentBranch>`, a dedicated FoD release may need to be created first.
> - [if a release named `<currentBranch>` already exists in FoD] A release named `<currentBranch>` already exists (ID: `<id>`). Did you mean to target that one?
> - [if other branch/release overlaps] Other branches with matching FoD releases: `<list>`
>
> How would you like to proceed?
> - **Continue** — use the `<resolvedReleaseName>` release as originally resolved
> - **Switch** — switch to the `<currentBranch>` release (ID: `<id>`)
> - **Create** — create a new `<currentBranch>` release first (I'll handle it using `use-case-create-release.md`)
> - **Cancel** — stop here so you can manually correct the target release

Wait for the user's single reply, then act on it. Do **not** ask follow-up alignment questions.

---

### Step 6 — Execute the Scan

Before issuing any scan start command, verify all of the following:

- [ ] Release resolved — numeric ID or `App:Release` string confirmed
- [ ] Branch/release alignment verified — mismatch acknowledged or confirmed intentional
- [ ] Scan type confirmed by user (SAST / SCA / SAST+SCA / DAST)
- [ ] Execution method confirmed by user as CLI/agent
- [ ] Async vs. wait-for preference confirmed (SAST and SCA only; DAST is always async)
- [ ] In-progress scan check completed (first sub-step in the appropriate scan type below) — no active scans found

Do NOT issue any `*-scan start` command until all items above are ticked. The in-progress check is mandatory, not optional.

---

#### SAST Scan (or SAST + SCA)

**6a. Check for an in-progress SAST scan**

Before starting, verify there is no scan already in progress, queued, or paused for this release. Run this for SAST:

```bash
fcli fod release lss --rel=<releaseId> \
  -q "scanType=='Static' && (analysisStatusType=='InProgress' || analysisStatusType=='Waiting' || analysisStatusType=='Paused')" \
  -o json
```

If also submitting open source (SAST + SCA), check for open source scans too:

```bash
fcli fod release lss --rel=<releaseId> \
  -q "scanType=='OpenSource' && (analysisStatusType=='InProgress' || analysisStatusType=='Waiting' || analysisStatusType=='Paused')" \
  -o json
```

If either command returns one or more scans, **stop and inform the user**:
- There is already a **{analysisStatusType}** `{scanType}` scan for this release (In Progress / Queued / Paused)
- Scan ID: `{scanId}`
- View the release scans page: `<fodPortalUrl>/Releases/<releaseId>/Scans`

Do not start a new scan. Ask the user to wait for the existing scan to complete before submitting another.

**6b. Check and configure scan settings**
```bash
fcli fod sast-scan get-config --rel=<releaseId> -o json
```
If the scan is already configured (returns valid config), skip to packaging.

If not yet configured, set it up with these defaults:
```bash
fcli fod sast-scan setup --rel=<releaseId> \
  --audit-preference="Automated" \
  --assessment-type="Static Assessment" \
  --frequency="Subscription" \
  --skip-if-exists \
  --use-aviator
```

For **SAST + SCA**, add `--oss`:
```bash
fcli fod sast-scan setup --rel=<releaseId> \
  --audit-preference="Automated" \
  --assessment-type="Static Assessment" \
  --frequency="Subscription" \
  --skip-if-exists \
  --use-aviator \
  --oss
```

**Fallback handling**:
- If `--oss` or `--use-aviator` are not available or not enabled on this tenant, remove the unsupported flag(s) and retry.
- If the assessment type `"Static Assessment"` doesn't exist, find an available alternative:
  ```bash
  fcli fod release lsat --rel=<releaseId>
  ```
  Use the `assessmentTypeName` value from the output. Similarly check available `frequencyType` values and adjust `--frequency` if needed.

**Do NOT use** these flags: `--include-third-party-libs`, `--technology-stack`, `--use-source-control`.

**6c. Install the ScanCentral Client and package the code**

For FoD, use the **latest** sc-client version (see `references/fcli-install.md` for details and version selection guidance):

```bash
# Install the latest sc-client (skip if already installed)
fcli tool sc-client install -y

# Package the code using the fcli action (recommended — handles build tool detection)
fcli fod action run package --rel=<releaseId>
```
This produces a `package.zip` file in the current directory by default. Note the output path.

**6d. Start the scan**

Async (return immediately):
```bash
fcli fod sast-scan start --rel=<releaseId> -f package.zip -o json
```
After submitting, the user can monitor scan progress in the FoD portal:
```
<fodPortalUrl>/Releases/<releaseId>/Scans
```

Wait for completion:
```bash
fcli fod sast-scan start --rel=<releaseId> -f package.zip --store myScan -o json
fcli fod sast-scan wait-for ::myScan::
```

**6e. Retrieve scan summary** (after completion — skip if async and scan still running)

Get the scan ID from the start output or from release data:
```bash
fcli fod sast-scan get <scanId> -o json
```

Summarize the key results, especially changes:
- **New issues** (`totalIssues` delta or `issueCountDetails.newToRelease`)
- **Reopened issues**
- **Fixed / Fix Validated issues**
- Policy pass/fail status (`isPassed` from `fcli fod release get`)

---

#### SCA / Open Source Scan

**6a. Check for an in-progress open source scan**

Before starting, verify there is no open source scan already in progress, queued, or paused for this release:

```bash
fcli fod release lss --rel=<releaseId> \
  -q "scanType=='OpenSource' && (analysisStatusType=='InProgress' || analysisStatusType=='Waiting' || analysisStatusType=='Paused')" \
  -o json
```

If the command returns one or more scans, **stop and inform the user**:
- There is already a **{analysisStatusType}** open source scan for this release (In Progress / Queued / Paused)
- Scan ID: `{scanId}`
- View the release scans page: `<fodPortalUrl>/Releases/<releaseId>/Scans`

Do not start a new scan. Ask the user to wait for the existing scan to complete before submitting another.

**6b. Package the code**
```bash
fcli fod action run package --rel=<releaseId>
```

**6c. Start the scan**

Async:
```bash
fcli fod oss-scan start --rel=<releaseId> -f package.zip -o json
```
After submitting, the user can monitor scan progress in the FoD portal:
```
<fodPortalUrl>/Releases/<releaseId>/Scans
```

Wait for completion:
```bash
fcli fod oss-scan start --rel=<releaseId> -f package.zip --store myOssScan -o json
fcli fod oss-scan wait-for ::myOssScan::
```

**6d. Retrieve scan summary** (after completion)
```bash
fcli fod oss-scan get <scanId> -o json
```

Summarize: component counts, new CVEs found, license risk changes, and any policy impact.

---

#### DAST Scan

**6a. Confirm scan configuration**
```bash
fcli fod dast-scan get-config --rel=<releaseId> -o json
```

If the configuration is incomplete or not set up, **stop here**. Inform the user that DAST scans require configuration in the FoD web portal before they can be submitted. Provide the FoD portal URL and direct them to the DAST configuration section for their release. Do not attempt to configure DAST via fcli.

**6b. Check for an in-progress DAST scan**

Before starting, verify there is no DAST scan already in progress, queued, or paused for this release:

```bash
fcli fod release lss --rel=<releaseId> \
  -q "scanType=='Dynamic' && (analysisStatusType=='InProgress' || analysisStatusType=='Waiting' || analysisStatusType=='Paused')" \
  -o json
```

If the command returns one or more scans, **stop and inform the user**:
- There is already a **{analysisStatusType}** DAST scan for this release (In Progress / Queued / Paused)
- Scan ID: `{scanId}`
- View the release scans page: `<fodPortalUrl>/Releases/<releaseId>/Scans`

Do not start a new scan. Ask the user to wait for the existing scan to complete before submitting another.

**6c. Start the scan** (always async — DAST scans are long-running)
```bash
fcli fod dast-scan start --rel=<releaseId> -o json
```

Confirm the scan has been submitted and provide the scan ID. Direct the user to the FoD portal to monitor progress:
```
<fodPortalUrl>/Releases/<releaseId>/Scans
```
