# OpenText Fortify Skills

AI agent skills that teach Claude, GitHub Copilot and other AI agents how to use [OpenText Fortify](https://www.opentext.com/products/application-security) effectively — covering SAST/DAST/SCA scanning, vulnerability triage, audit workflows, remediation, dependency upgrades, CI/CD integration, FCLI commands and more.

## Skills

| Skill | Description |
|-------|-------------|
| **fortify-fod** | Fortify on Demand (SaaS) — applications, releases, scans, issues, OSS analysis, portfolio reporting |
| **fortify-ssc** | Software Security Center (on-premise) — manage application versions, artifacts, scan jobs, issue triage |
| **fortify-remediate** | Fix SAST and DAST vulnerabilities detected by Fortify; Aviator AI remediation |
| **fortify-dependency-upgrade** | Remediate SCA / open source findings (vulnerable dependencies, CVEs) by upgrading versions and fixing resulting breakage |
| **fortify-exploitability-analysis** | Triage whether a known CVE/GHSA is actually exploitable (reachable) in this project, as opposed to just present |
| **fortify-change-review** | Detect common, high impact security issues in code as it is being generated |
| **fortify-create-app** | Create new Fortify applications in FoD or SSC — guided onboarding with validation and defaults |
| **fortify-cicd-integration** | Add Fortify scanning to CI/CD pipelines — GitHub Actions, GitLab CI, Azure DevOps, Jenkins |
| **fcli-common** | Fortify CLI (fcli) — installation, authentication, output formats, SpEL queries, custom actions |


## Agents

Agents are multi-skill orchestrators that handle end-to-end workflows.

| Agent | Description |
|-------|-------------|
| **fortify-onboarding** | Onboard new applications into Fortify (FoD or SSC) — creates the app, configures settings, and optionally sets up CI/CD scanning pipelines. Handles single repos, bulk lists, or entire GitHub/GitLab/Azure DevOps organizations |
| **fortify-exploitability-analysis** | Batch-triage a list of known CVEs/GHSAs for reachability across a codebase — sourced from an SBOM, a Fortify on Demand release, an SSC application version, or an explicit list. Produces per-CVE reports plus a combined CycloneDX VEX file |

## Prerequisites

- **fcli** installed and on your PATH — [install instructions](skills/fcli-common/references/fcli-install.md)
- An active Fortify on Demand (FoD) or Software Security Center (SSC) account
- An AI assistant that supports Agent Skills (Claude Code, GitHub Copilot, Cursor, Gemini CLI, etc.)

## Installation

### Claude Code

Add the marketplace from GitHub, then install the plugin:

```bash
claude plugin marketplace add fortify/skills
claude plugin install fortify-skills@fortify
```

The plugin registers all nine skills and both agents automatically.

### GitHub Copilot

**Recommended: install the [Fortify Code Security](https://marketplace.visualstudio.com/items?itemName=fortifyvsts.fortify-code-security) VS Code extension.** It bundles all Fortify skills, can automatically install fcli, and adds full IDE integration (scanning, vulnerability review, Aviator AI remediation, and an optional fcli MCP server):

1. Open VS Code and search for **Fortify Code Security** in the Extensions panel, or install directly:
   ```
   ext install fortifyvsts.fortify-code-security
   ```
2. The extension registers all skills automatically and sets up GitHub Copilot Agent Mode integration.

**Alternative: manual install.** Copy the skills to your Copilot skills directory:

```
<user>/.copilot/skills/
```

This gives you the skills without the IDE features (scanning UI, vulnerability browser, Aviator inline fixes, MCP server).

### OpenAI Codex

This repository includes a marketplace catalog at `.agents/plugins/marketplace.json`. When the repo is your current workspace, Codex discovers it automatically as a repo-scoped marketplace. Open the plugin directory, select **OpenText Fortify**, and install **fortify-skills**.

To make the plugin available across all workspaces, add an entry to your personal marketplace at `~/.agents/plugins/marketplace.json` (create the file if it doesn't exist), replacing `<path>` with the absolute path to this directory:

```json
{
  "name": "fortify",
  "interface": { "displayName": "OpenText Fortify" },
  "plugins": [
    {
      "name": "fortify-skills",
      "source": { "source": "local", "path": "<path>" },
      "policy": { "installation": "AVAILABLE", "authentication": "ON_INSTALL" },
      "category": "Security"
    }
  ]
}
```

Then restart Codex. The plugin registers all nine skills automatically.

### Gemini CLI

Install directly from the GitHub repository:

```bash
gemini extensions install https://github.com/fortify/skills
```

The extension bundles all nine skills. Gemini CLI auto-discovers them and activates whichever skill is relevant to your task.

To test locally before publishing:

```bash
gemini extensions link /path/to/public
```

### Other AI assistants

Any assistant that supports the [Agent Skills](https://agentskills.io) standard can load skills from this directory. Point your assistant's skill path to the `skills/` subdirectory.

## Usage

Once installed, the skills activate automatically when relevant. Examples of prompts that trigger each skill:

| Prompt | Skill / Agent activated |
|--------|------------------------|
| "Onboard this repo to Fortify" | fortify-onboarding (agent) |
| "Create a new application in FoD for our payments team" | fortify-create-app |
| "Show me all critical issues in the payment-service release" | fortify-fod |
| "Upload my FPR and check policy compliance" | fortify-ssc |
| "Fix the SQL Injection findings in UserService.java" | fortify-remediate |
| "Remediate the open source / SCA findings by upgrading the vulnerable dependencies" | fortify-dependency-upgrade |
| "Is CVE-2021-44228 actually exploitable in this project?" | fortify-exploitability-analysis |
| "Triage all the CVEs in this SBOM for reachability" | fortify-exploitability-analysis (agent) |
| "Add a new endpoint that returns account details" | fortify-change-review |
| "Add Fortify scanning to my GitHub Actions workflows" | fortify-cicd-integration |
| "Create a custom fcli action to export FoD critical issues as CSV" | fortify-fod + fcli-common |


## Resources

- [Fortify CLI (fcli) — GitHub](https://github.com/fortify/fcli)
- [Fortify on Demand documentation](https://www.microfocus.com/documentation/fortify-on-demand/)
- [Software Security Center documentation](https://www.microfocus.com/documentation/fortify-software-security-center/)

## License

MIT
