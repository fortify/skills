# Task: Run Command With SCM Auth

Use this workflow to execute SCM-authenticated commands safely. Invoke it before
running `fcli license ncd-report create`, `fcli license ncd-report validate-sources`,
or direct SCM REST calls.

## Non-negotiable rules

- Apply non-negotiable rules from [../SKILL.md](../SKILL.md)
- Prefer environment variables and platform CLI auth reuse.
- Avoid `read -sp` in VS Code terminals (unreliable).
- Never print token values to logs, markdown output, or chat messages.
- Never request secrets through tools that route input through the model — the user
  should type secrets directly into the terminal or editor.
- Never modify or remove `#env("...")` credential expressions in `NcdReportConfig.yml` just to bypass missing-token/auth errors.
- If required token env vars are missing, stop and ask user to provide credentials through this workflow; do not silently switch to unauthenticated access.
- Only use unauthenticated SCM access if the user explicitly requests it.
- Do not change an existing config's authentication mode unless the user explicitly asks to update the config.
- Run the caller-provided command in Step 4 of this workflow; do not run it outside this workflow.

## Step 0: Capture command context and required tokens

Capture the command that needs to be executed. If it is an
`fcli license ncd-report create|validate-sources` command, read the config file
passed to that command and identify all referenced `#env("...")` variables.

If it is a direct SCM REST call, identify which SCM system(s) are targeted and
acquire only those required tokens.

## Step 1: Choose token acquisition strategy

Prefer the lightest secure path in this order:

1. **Use an existing CI-provided environment variable** if already present.
2. **Reuse an existing authenticated CLI session**:
   - GitHub: check `gh auth status`; if available and authenticated, use `gh auth token`.
   - GitLab: check `glab auth status`; if available and authenticated, use `glab auth token`.
3. **Manual token entry** into a local env file opened in editor.

If valid token(s) were detected using #1 or #2 above, continue to Step 3. Otherwise, continue to Step 2.

## Step 2: Use a local env file workflow

Create a temporary env file template and have the user fill secrets locally:

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
- Continue to Step 3.
- Remove the file after command execution in Step 4: `rm -f /tmp/.ncd-creds`.

## Step 3: Set and verify required env vars

Set each required command env var in the current shell from the token values collected
in Step 1/Step 2, then verify they are set.

- For `ncd-report create|validate-sources`, required env vars are the `#env("...")`
  variable names found in Step 0.
- For SCM REST calls, required env vars are those needed by the targeted SCM system.

- If all required env vars are set, continue to Step 4.
- If one or more are missing, stop and report missing env var names (names only), then continue Step 1/Step 2 until complete.
- If the user explicitly requests unauthenticated access, record that explicit request and continue to Step 4 without editing config auth expressions.

### Step 3 gate

 - [ ] Either all required token environment variables are set, or explicit user confirmation to proceed unauthenticated is captured

## Step 4: Continue command execution

Run the caller-provided command now:

- Authenticated path: required env vars are set.
- Unauthenticated path: only if explicitly requested by the user.

After command completion, return to the original task workflow.
