# SCM Credential Handling

Use this workflow when you need to acquire and configure SCM platform authentication
tokens for NCD report generation or config discovery. Referenced by
[prepare-config.md](prepare-config.md) and [generate-report.md](generate-report.md).

## Non-negotiable rules

- Apply non-negotiable rules from [../SKILL.md](../SKILL.md)
- Prefer environment variables and platform CLI auth reuse.
- Avoid `read -sp` in VS Code terminals (unreliable).
- Never print token values to logs, markdown output, or chat messages.
- Never request secrets through tools that route input through the model — the user
  should type secrets directly into the terminal or editor.

## Step 1: Choose token acquisition strategy

Prefer the lightest secure path in this order:

1. **Use an existing CI-provided environment variable** if already present.
2. **Reuse an existing authenticated CLI session**:
   - GitHub: check `gh auth status`; if authenticated, use `gh auth token`.
   - GitLab: check `glab auth status`; if authenticated, use `glab auth token`.
3. **Manual token entry** into a local env file opened in editor.

For Azure DevOps, there is no equally standard token print command like `gh auth token` or `glab auth token`. Prefer an existing PAT environment variable, or have the user add their PAT manually to the env file workflow below.

## Step 2: Use a local env file workflow (recommended)

When token env vars are missing, create a temporary env file template and have the user fill secrets locally:

```bash
cat > /tmp/.ncd-creds <<'EOF'
export GITHUB_TOKEN=
export GITLAB_TOKEN=
export AZURE_DEVOPS_TOKEN=
EOF
chmod 600 /tmp/.ncd-creds
```

Then:
- Ask the user to open and edit `/tmp/.ncd-creds` locally.
- Load secrets into the shell with `source /tmp/.ncd-creds`.
- Run report commands.
- Remove the file after use: `rm -f /tmp/.ncd-creds`.

If `gh` or `glab` is authenticated, users may populate the file without exposing values to the model, for example by running commands directly in terminal when editing.

## Step 3: Verify referenced env vars

Before executing any report command, verify that every token referenced by
`#env("...")` in `NcdReportConfig.yml` is set in the current shell.

Ask the user to confirm that all required tokens are available in the environment, or
offer to help set them up using the local env file workflow above.

## Credential handling gate

- [ ] Token acquisition strategy selected (CLI reuse, existing env, or manual env file)
- [ ] Required token env vars verified as set in the current shell
