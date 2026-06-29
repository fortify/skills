---
name: fcli-common
description: Generic reference for the Fortify CLI (fcli). Install, upgrade, authenticate, fcli environment variables, output formats, SpEL query syntax, variable chaining and custom action framework.
license: MIT
metadata:
  version: "1.0.2"
  tested-with:
    fcli: "3.18"
---

fcli (Fortify CLI) is the unified command-line interface for all Fortify products: Fortify on Demand (FoD), Software Security Center (SSC), ScanCentral SAST, and ScanCentral DAST.

Docs: https://fortify.github.io/fcli/latest/ | Source: https://github.com/fortify/fcli | Skill content reflects fcli v3.18.

This skill covers **cross-product fcli functionality**: installation, session management, output formatting, SpEL queries, variable chaining, custom action development, and ScanCentral management operations. For product-specific operations (querying issues, managing scans, auditing vulnerabilities), defer to the `fortify-fod` or `fortify-ssc` skills — they provide deeper coverage of product-specific commands, API fields, and workflows.

## Top-Level Command Tree

| Module | Purpose |
|--------|---------|
| `fcli fod` | Fortify on Demand (SaaS) |
| `fcli ssc` | Software Security Center (on-premise) |
| `fcli sc-sast` | ScanCentral SAST scan management |
| `fcli sc-dast` | ScanCentral DAST scan management |
| `fcli action` | Cross-product automation actions |
| `fcli tool` | Install/manage fcli and other Fortify tools |
| `fcli config` | Proxy, trust store, public key settings |
| `fcli util` | Utilities (variables, command listing, etc.) |

Use `fcli --help` to list all top-level commands, and `fcli <module> -h` to drill down.

## Verifying Commands and Flags

**When uncertain whether a flag or subcommand exists, run `fcli <command> --help` before using it.** FCLI is fully self-documenting — every module, subcommand, and flag is discoverable at runtime. This is the authoritative source; do not invent flags or subcommands based on patterns from other CLIs.

```bash
fcli fod issue list --help    # verify available flags before constructing a command
fcli fod release --help       # list all release subcommands
fcli util all-commands list   # list every command available in the installed version
```

## Version Compatibility

Skill content reflects fcli v3.18 (see `tested-with` in frontmatter). If a command fails with "unknown subcommand", "unrecognized option", or a 404/405 REST error, check versions first: run `fcli --version` and, for product-specific failures, ask the user for their SSC or FoD version. Older installs may lack a command; newer installs are generally backward compatible — use `fcli util all-commands list` or `--help` to discover available commands. Do not silently retry with guessed syntax.

## Installation & Upgrade

See `references/fcli-install.md` for full procedures. Do NOT silently download or execute anything without user confirmation. Verify: `fcli --version`

## Session Management

Most fcli product modules require a login session before use:

```bash
fcli fod session login --url "https://api.ams.fortify.com" --user <user> --password <PAT> --tenant <tenantCode>
fcli ssc session login --url "https://my-ssc.com" --user <user> --password <pass>
```

Always run `session logout` when finished. Never output passwords or tokens in logs or chat. All Fortify API access must go through fcli — the escape hatch for endpoints not covered by a named command is `fcli fod rest call` / `fcli ssc rest call`. For credential best practices, named sessions, CI/CD setup, and the `--validate` flag, see `references/fcli-session.md`.

## Output Formats

**Prefer `-o json` for programmatic work.** Parse JSON and summarize rather than relying on table output, which omits fields and truncates values.

For extended format options and JSON processing patterns, see `references/output-formats.md`. For `--style` options and SpEL query syntax, see `references/fcli-query-output.md`.

## Filtering Quick Reference

`-q` / `--query` evaluates a SpEL expression per record:

```bash
-q "name=='MyApp'"                              # equality
-q "{'Critical','High'}.contains(severity)"    # multi-value match (preferred over ||)
```

For full syntax (date functions, null safety, regex, server-side filtering), see `references/fcli-query-output.md`.

## Variable Chaining (`--store`)

Store output from one command for use in the next, avoiding shell parsing of JSON:

```bash
fcli ssc appversion get MyApp:main --store myAV
fcli ssc artifact upload -f scan.fpr --appversion ::myAV::

fcli fod sast-scan start --rel MyApp:main -f pkg.zip --store myScan
fcli fod sast-scan wait-for ::myScan::
```

## Actions

Built-in actions automate common workflows (CI/CD pipeline runs, SARIF/GitHub/GitLab vulnerability export, policy checks, PR comments):

```bash
fcli fod action list               # list available FoD actions
fcli ssc action list               # list available SSC actions
fcli fod action help <action>      # view action usage/options
fcli fod action run <action> ...   # run an action
```

Documentation: https://fortify.github.io/fcli/latest/fod-actions.html and https://fortify.github.io/fcli/latest/ssc-actions.html

### Custom Action Development

Fcli actions are YAML files that define custom automation workflows. Use them to export vulnerabilities, enforce security policy, generate reports, integrate with external systems, etc.

See `references/fcli-custom-actions.md` for the full development workflow (scaffold, CLI options, step patterns, SpEL expressions, testing, signing).

## Environment Variables

| Variable | Purpose |
|---------|---------|
| `FCLI_DEFAULT_FOD_URL` | Default FoD URL for all fod commands |
| `FCLI_DEFAULT_SSC_URL` | Default SSC URL for all ssc commands |
| `FCLI_DEFAULT_<CMD>_<OPT>` | Default for any option — see docs |
| `FCLI_STATE_DIR` | Override fcli session/variable storage location |
| `HTTPS_PROXY` / `HTTP_PROXY` | Proxy for fcli network requests |
| `FCLI_ENCRYPT_KEY` | Custom encryption key for session data |

Command-prefixed defaults walk the command tree, e.g. `FCLI_DEFAULT_FOD_RELEASE_LIST_APP` sets the default `--app` for `fcli fod release list`.

## Reference Files

Load these **only when needed** for the specific task at hand:

| File | When to load |
|------|-------------|
| `references/fcli-install.md` | Installing, upgrading, or troubleshooting fcli setup |
| `references/fcli-session.md` | Session best practices, named sessions, CI/CD token guidance, `--validate` |
| `references/fcli-query-output.md` | Complex queries, SpEL syntax, null safety, date functions, server-side filtering |
| `references/output-formats.md` | Extended format options (`json-properties`, `--fetch=1`) and JSON processing scripting patterns |
| `references/mutating-operations.md` | Full safety rules for delete, create/update, access control, and REST mutations |
| `references/fcli-custom-actions.md` | Creating or editing custom fcli action YAML files |
| `references/action-yaml-reference.md` | Full YAML instruction set or SpEL function reference |
