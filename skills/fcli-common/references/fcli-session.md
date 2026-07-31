# fcli Session Management Reference

Source: https://fortify.github.io/fcli/latest/

---

## Login Commands

```bash
fcli fod session login --url "https://api.ams.fortify.com" --user <user> --password <PAT> --tenant <tenantCode>
fcli ssc session login --url "https://my-ssc.com" --user <user> --password <pass>
```

Always run `session logout` when finished — cleans up tokens server-side:
```bash
fcli fod session logout
fcli ssc session logout
```

---

## Credential Best Practices

- **FoD**: use a Personal Access Token (PAT), not your account password. PATs are created in FoD under User Settings → Personal Access Tokens.
- **SSC**: use a **CIToken** or **AutomationToken** (long-lived automation tokens) rather than a session password — avoids creating a temporary session token on every login.
- **Never** output passwords or tokens in command output, logs, or chat messages. Pass credentials via `--password` interactively or via environment variables (e.g., `FOD_PAT`, `SSC_TOKEN`).

---

## Named Sessions

Use named sessions (`--fod-session`, `--ssc-session`) to work with multiple instances simultaneously or avoid collisions:

```bash
# Login to two different FoD tenants
fcli fod session login --fod-session prod --url "https://api.ams.fortify.com" ...
fcli fod session login --fod-session staging --url "https://api.ams.fortify.com" ...

# Use a specific session for subsequent commands
fcli fod release list --fod-session prod
fcli fod release list --fod-session staging
```

List active sessions:
```bash
fcli fod session ls
fcli ssc session ls
```

---

## CI/CD Considerations

- Set `FCLI_STATE_DIR` to a pipeline-specific path (e.g., a temp directory scoped to the job) to prevent session collisions between concurrent pipeline runs.
- Pin `@fortify/setup` to a specific fcli version for reproducible CI builds: `npx @fortify/setup@2 env init --tools=fcli:v3.23.0`
- Verify session is still valid server-side with `--validate` — useful when tokens may be revoked between runs and you can't trust the locally cached expiry:

```bash
fcli fod session ls --validate    # hits server to confirm OAuth token is still valid
fcli ssc session ls --validate    # hits server and refreshes cached expiry/state
```

---

## Checking Session Status

```bash
# Check default session is active and not expired
fcli fod session ls --query "name=='default' && expired=='No'"
fcli ssc session ls --query "name=='default' && expired=='No'"
```

---

## Never Bypass fcli for Fortify API Access

All Fortify API access must go through fcli commands — including when fcli seems slow or limited. **Never** read fcli's session or state files (`~/.fortify/fcli/state`, `$FCLI_STATE_DIR`), extract tokens or credentials from disk, or call Fortify APIs directly with `curl`, `wget`, `requests`, `Invoke-WebRequest`, or any other HTTP client.

The supported escape hatch for REST endpoints not covered by a named fcli command:

```bash
fcli fod rest call <path>    # FoD: call an arbitrary FoD REST endpoint
fcli ssc rest call <path>    # SSC: call an arbitrary SSC REST endpoint
```

If fcli is genuinely unworkable for a task, surface the limitation to the user — do not work around it.
