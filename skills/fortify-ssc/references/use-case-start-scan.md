## Use Case: Starting a Scan

The user wants to scan their code or application with Fortify (e.g., "scan my code", "run a SAST scan", "check my code for vulnerabilities", "run a DAST scan").

> If the application version is not already resolved, load `references/resolving-appversion.md` first.

> **Important**: Only start a scan when the user explicitly requests one. Fortify scans are comprehensive and resource-intensive — they should not be initiated automatically on every code change or as a side effect of another task.

---

### Step 1 — Determine the Scan Type

Ask the user which type of scan they want to run (or infer from context if obvious):

| Scan type | Description |
|-----------|-------------|
| **SAST** | Static code analysis via ScanCentral SAST — finds vulnerabilities in source code |
| **DAST** | Dynamic scan of a running application URL via ScanCentral DAST |

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
Open any found files and check whether Fortify SSC / ScanCentral scanning steps are already configured (look for `fortify`, `ssc`, `scancentral`, `sc-sast`, or `sc-dast` keywords).

**2c. Get the SSC URL (for the web UI option)**

```bash
fcli ssc session ls -o json
# → use the `url` field from the active session (e.g., https://ssc.example.com/ssc)
# This is the SSC base URL for constructing portal links.
```

If the application version has already been resolved, you will have the numeric version ID needed to construct direct portal links (see Step 2).

**2d. Present the options**

Based on the above context, present these options with relevant context highlighted:

| Option | When to suggest |
|--------|-----------------|
| **Web UI / Portal** | Always available. Good for ad-hoc DAST scans, does not support SAST scans. Provide the SSC portal link. |
| **CI/CD pipeline** | If pipeline files found — describe how to trigger existing integration. If no pipeline — offer to set one up using the `fortify-cicd-integration` skill. |
| **IDE plugin** | If plugin detected — point the user at it directly. If not detected — provide marketplace link for their IDE. SAST only, does not support DAST. |
| **CLI/agent (fcli + scancentral)** | Recommended when the user explicitly wants to run a scan right now from the terminal or wants the agent to execute it end-to-end. |

Ask the user to confirm their preferred method before proceeding.


---

### Step 3 — Route Based on Chosen Method

**Web UI**: Direct the user to the ScanCentral DAST module in SSC to start a scan:
```
<sscUrl>/html/ssc/scans/dast
```

**CI/CD pipeline**:
- If Fortify is already configured in the pipeline: describe the trigger conditions (push to main, pull request, manual dispatch, etc.) and instruct the user to trigger accordingly.
- If not yet configured: offer to set it up by invoking the `fortify-cicd-integration` skill.

**IDE plugin**:
- If already installed: direct the user to the Fortify extension/plugin panel and describe how to initiate a scan from within the IDE.
- If not installed: provide the marketplace link for their IDE:
  - VS Code: https://marketplace.visualstudio.com/items?itemName=fortifyvsts.fortify-extension-for-vs-code
  - IntelliJ/JetBrains: https://plugins.jetbrains.com/plugin/24486-fortify-analysis
  - Visual Studio: https://marketplace.visualstudio.com/items?itemName=fortifyvsts.fortify-complete-visual-studio-64
  - Eclipse: https://marketplace.eclipse.org/content/fortify-complete-plugin

**CLI/agent**: continue to Step 4.

---

### Step 4 — Gather additional input (SAST only)

**Language check for SAST**
If the project uses C, C++, Scala, Objective-C, or Swift, inform the user that these languages require **local translation** by Fortify before submission using `sourceanalyzer`. This is more complex than standard `scancentral package` workflows. Direct them to the [Fortify ScanCentral SAST documentation](https://www.microfocus.com/documentation/fortify-software-security-center/) for instructions on local translation and MBS submission workflows. Do not proceed with automated packaging for these languages.


**Ask About Async vs. Wait-for**
For SAST scans, ask whether the user wants to:
- **Start and return immediately** (async) — recommended for most cases
- **Wait for completion** — only if the user has previously scanned this app and knows it completes quickly

Highlight: while many scans complete in a few minutes, larger codebases can take significantly longer. Only offer wait-for if the user expects a short scan.

DAST scans always run asynchronously — do not offer wait-for for DAST.

---

### Step 5 — Verify Branch/Version Alignment

Run all commands below **before** prompting the user — collect all alignment information in one pass, then surface any issues in a single confirmation.

**Collect alignment data (run silently)**

```bash
# Current branch
git branch --show-current

# All versions for this application — compare names against the branch list
fcli ssc appversion list --query "application.name=='<AppName>'" \
  -o table --columns name,id,issueTemplate,committed

# All local/remote branches — compare against the version list
git branch -a --format='%(refname:short)' | sed 's|^origin/||' | sort -u
```

**Evaluate findings**

| Finding | Treatment |
|---------|-----------|
| Version resolved from a user-supplied numeric ID | ✅ Skip all checks — explicit choice made. Proceed to Step 6. |
| `git branch` returned empty (detached HEAD / no git context) | ✅ No comparison possible. Proceed to Step 6. |
| Current branch **exactly matches** the resolved version name | ✅ Aligned. Proceed to Step 6. |
| Current branch **does not match** the resolved version name | ⚠️ Include in prompt |
| Another SSC version exists whose name matches the current branch | ⚠️ Include in prompt |
| Other git branches match other SSC version names (not the resolved target) | ℹ️ Mention alongside the prompt if one is required; otherwise note briefly when presenting Step 6 |

**Prompt the user (at most once)**

If all findings are ✅, proceed to Step 6 without prompting.

If any ⚠️ findings exist, send **one** message covering all of them:

> ⚠️ **Version alignment — please confirm before proceeding**
>
> - Current git branch: **`<currentBranch>`**
> - Target SSC application version: **`<resolvedVersionName>`**
> - [if names differ] These do not match. If you meant to scan `<currentBranch>`, a dedicated SSC application version may need to be created first.
> - [if a version named `<currentBranch>` already exists in SSC] A version named `<currentBranch>` already exists (ID: `<id>`). Did you mean to target that one?
> - [if other branch/version overlaps] Other branches with matching SSC versions: `<list>`
>
> How would you like to proceed?
> - **Continue** — use the `<resolvedVersionName>` version as originally resolved
> - **Switch** — switch to the `<currentBranch>` version (ID: `<id>`)
> - **Create** — create a new `<currentBranch>` version first (I'll handle it using `use-case-create-version.md`)
> - **Cancel** — stop here so you can manually correct the target version

Wait for the user's single reply, then act on it. Do **not** ask follow-up alignment questions.

---

### Step 6 — Execute the Scan

Before issuing any scan start command, verify all of the following:

- [ ] Application version resolved — numeric ID or `App:Version` string confirmed
- [ ] Branch/version alignment verified (Step 5) — mismatch acknowledged or confirmed intentional
- [ ] Scan type confirmed by user (SAST / DAST)
- [ ] Execution method confirmed by user as CLI/agent
- [ ] Async vs. wait-for preference confirmed (SAST only; DAST is always async)
- [ ] In-progress scan check completed (first sub-step in the appropriate scan type below) — no active scans found
- [ ] For SAST: SC-SAST session verified to include client auth token (Step 6a)

Do NOT issue any `scan start` command until all items above are ticked. The in-progress check is mandatory, not optional.

#### SAST Scan (ScanCentral SAST)

**6a. Verify SC-SAST session**

Check the session supports SC-SAST before proceeding (see the Authentication Note in `references/use-case-scsast-scan-status.md` for full details):

```bash
fcli ssc session ls --query "name=='default' && expired=='No'"
```

Inspect the `url` field. If `SC-SAST: N/A (No client-auth-token)` appears, **stop and inform the user**: the session was created without a client auth token, and some SC-SAST commands may fail. Prompt them to re-login with `--client-auth-token`:

```bash
fcli ssc session login --url=<ssc-url> \
  --client-auth-token=<token> \
  --username=<user> --password=<pw>
```

**6b. Check for an in-progress SAST scan**

Before starting, verify there is no SC-SAST scan already running, queued, or pending for this application version:

```bash
fcli sc-sast scan list \
  --query "projectName=='<AppName>' && pvName=='<VersionName>' && (jobState=='RUNNING' || jobState=='QUEUED' || jobState=='PENDING')" \
  -o json
```

If this command returns one or more scans, **stop and inform the user**:
- There is already a **{jobState}** SAST scan for this application version
- Job token: `{jobToken}`
- View scan progress on the ScanCentral SAST portal or in SSC

Do not start a new scan. Ask the user to wait for the existing scan to complete before submitting another.

> **Note**: If SC-SAST is configured to allow replacement of existing scan jobs (the default for most deployments), this check is advisory. Use `--no-replace` on the `scan start` command if you want to preserve any in-progress scan instead of replacing it.

**6c. Install the compatible ScanCentral Client and package the code**

For SSC / ScanCentral SAST, the sc-client version must be **compatible with the sensors**. Determine and install the compatible version:

```bash
# Check compatible version
fcli sc-sast sensor list --latest-only
# → note the "Compatible client version" column (e.g., 25.4)

# Install and package (replace <compatibleVersion> with the value above)
fcli tool sc-client install -v <compatibleVersion> -y
fcli tool sc-client run -v <compatibleVersion> -- package -o package.zip
```

If auto-detection fails, specify the build tool explicitly with `-bt` (e.g., `-bt mvn`, `-bt gradle`, `-bt msbuild`, `-bt npm`).

See `references/fcli-install.md` for full sc-client installation details, version selection guidance, and additional options.

**6d. Start the scan**

Always include `--publish-to` to automatically publish the completed FPR to the SSC application version:

Async (return immediately):
```bash
fcli sc-sast scan start \
  -f package.zip \
  --publish-to="<AppName>:<VersionName>" \
  -o json
```
Note the `jobToken` from the output — the user can check scan progress with `fcli sc-sast scan status <jobToken>`.

Wait for completion (scan + SSC artifact processing):
```bash
fcli sc-sast scan start \
  -f package.zip \
  --publish-to="<AppName>:<VersionName>" \
  --store myScan \
  -o json
fcli sc-sast scan wait-for ::myScan::
```

> The `wait-for` command waits until the scan is fully processed in SSC (scan completion + FPR upload + SSC artifact processing), provided `--publish-to` was specified and the ScanCentral Controller version is > 22.1. For older controller versions, it waits until the FPR is published to SSC.

**6e. Retrieve scan summary** (after completion — skip if async and scan still running)

After the scan completes and the FPR is processed in SSC, retrieve the updated issue summary for the application version:

```bash
fcli ssc appversion get "<AppName>:<VersionName>" -o json
```

Summarize the key results, especially new issues since the last scan:
```bash
# Count new issues by priority
fcli ssc issue list --av="<AppName>:<VersionName>" \
  --query "scanStatus=='NEW'" \
  -o json
```

Highlight:
- **New issues** (scan status `NEW`)
- **Reintroduced issues** (scan status `REINTRODUCED`)
- **Removed issues** (present in prior scan but not found in this one — use `--include=visible,removed` and `--query "scanStatus=='REMOVED'"`)
- Policy compliance status: check `currentState.hasReport` or load the issue investigation use case for deeper analysis

---

#### DAST Scan (ScanCentral DAST)

**6a. Verify the session**

Confirm an active SSC session is available (SC-DAST uses the same session as SSC):

```bash
fcli ssc session ls --query "name=='default' && expired=='No'"
```

If expired or not present, prompt the user to run `fcli ssc session login`.

**6b. Confirm and select scan settings**

DAST scans require pre-configured scan settings that define the target URL, scan policy, authentication, and other parameters. List available settings:

```bash
fcli sc-dast scan-settings list -o json
```

Key fields in the output:

| Field | Description |
|-------|-------------|
| `id` | Numeric settings ID |
| `name` | Settings name |
| `cicdToken` | CI/CD token used to reference these settings when starting a scan |
| `url` | Target URL this settings profile scans |
| `scanPolicyDescription` | Scan policy (e.g., `Standard`, `Developer`) |

If no scan settings exist, **stop here**. Inform the user that DAST scans require scan settings to be configured in the SSC / ScanCentral DAST web portal before they can be submitted. Direct them to the SSC portal and the ScanCentral DAST configuration section. Do not attempt to configure DAST scan settings via fcli.

If settings exist, confirm with the user which settings profile to use (by name or `cicdToken`). Note the `cicdToken` — you will need it for the next step.

**6c. Check for an in-progress DAST scan**

Before starting, verify there is no DAST scan already running, queued, or pending using the same settings:

```bash
fcli sc-dast scan list \
  --query "(scanStatusTypeDescription=='Running' || scanStatusTypeDescription=='Queued' || scanStatusTypeDescription=='Pending') && name matches '(?i).*<scanNameKeyword>.*'" \
  -o json
```

Or check more broadly for any active scan associated with the target application version (if known):

```bash
fcli sc-dast scan list \
  --query "applicationName=='<AppName>' && applicationVersionName=='<VersionName>' && (scanStatusTypeDescription=='Running' || scanStatusTypeDescription=='Queued' || scanStatusTypeDescription=='Pending' || scanStatusTypeDescription=='Paused')" \
  -o json
```

If any active scans are found, **stop and inform the user**:
- There is already a **{scanStatusTypeDescription}** DAST scan (ID: `{id}`, Name: `{name}`)
- View it in the SSC / ScanCentral DAST portal

Do not start a new scan. Ask the user to wait for or resolve the existing scan before submitting another.

**6d. Start the scan** (always async — DAST scans are long-running)

```bash
fcli sc-dast scan start \
  -s <cicdToken> \
  -n "<ScanName>" \
  -o json
```

Optional overrides:
- `-m <mode>` — override scan mode: `CrawlOnly`, `CrawlAndAudit`, or `AuditOnly`
- `-p <policy>` — override scan policy (name or ID)
- `-l <loginMacroBinaryFileId>` — override the login macro

Confirm the scan has been submitted and provide the scan ID from the output. Direct the user to the SSC / ScanCentral DAST portal to monitor progress:
```
<sscUrl>/html/ssc#/sc-dast/scans
```
