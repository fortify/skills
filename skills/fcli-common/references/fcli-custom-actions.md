# Fcli Custom Action Development

Fcli actions are YAML files that define automation workflows for Fortify CLI. They can interact with FoD and SSC REST APIs, run fcli commands, process data, generate reports, perform policy checks, and write output in various formats. Actions are run via `fcli <module> action run <action>`.

**Key references:**
- [Action Development Guide](https://fortify.github.io/fcli/latest/action-development.html)
- [Action Schema (v2.9.0)](https://fortify.github.io/fcli/schemas/action/fcli-action-schema-2.9.0.json)
- [Built-in action source code](https://github.com/fortify/fcli/tree/dev/v3.x/fcli-core)

**Local references (load when needed):**

| File | When to read |
|------|-------------|
| [action-yaml-reference.md](action-yaml-reference.md) | Complete reference for YAML instructions, step types, SpEL functions |

To discover available commands for `run.fcli` steps, run `fcli util all-commands list` against the installed version. Add `-h` to any module or command to see its options and usage (e.g., `fcli fod -h` to list all FoD commands; `fcli fod release list -h` for help on a specific command).

---

## Step 1: Determine what to build

Before writing any YAML, clarify the action's purpose:

1. **Target platform** — Will this run against FoD (`fcli fod action run`), SSC (`fcli ssc action run`), or be platform-agnostic (`fcli action run`)?
2. **Core function** — What does the action do? Examples: export vulnerabilities, check security policy, generate reports, automate workflows.
3. **Inputs** — What CLI options does the user need to provide? (e.g., output file, application version, thresholds)
4. **Outputs** — What should the action produce? (file output, console output, pass/fail exit code)
5. **Data sources** — REST API calls, fcli command output, or both?

---

## Step 2: Scaffold the action YAML

Every action YAML file requires these primary instructions. Use the schema reference for IDE code-completion and validation.

```yaml
# yaml-language-server: $schema=https://fortify.github.io/fcli/schemas/action/fcli-action-schema-2.9.0.json

author: <organization or individual name>

usage:
  header: <Short one-line summary>
  description: |
    <Multi-line description of what this action does,
    what inputs it expects, and what output it produces.>

config:
  rest.target.default: fod  # or 'ssc' — sets default REST target for rest.call steps

cli.options:
  file:
    names: -f, --file
    description: Output file path
    required: false
    default: output.csv

steps:
  - log.info: Action started
```

**Critical rules:**
- The `$schema` declaration is **required** — either as a YAML comment (`# yaml-language-server: $schema=...`) or as a YAML property (`$schema: ...`). At least one must be present.
- Current schema version is **2.9.0** for fcli 3.23.x. Always use `https://fortify.github.io/fcli/schemas/action/fcli-action-schema-2.9.0.json` (not the `-dev-` URL).
- `author`, `usage` (with `header` and `description`), and `steps` are **required**.

---

## Step 3: Define CLI options

CLI options are defined in the `cli.options` map. Each key becomes accessible via `${cli.<key>}` in expressions.

```yaml
cli.options:
  appversion:
    names: --appversion, --av
    description: "Application version id or <app>:<version> name"
    required: true
  file:
    names: -f, --file
    description: Output file name
    required: false
    default: output.sarif
  severity:
    names: --severity
    description: Minimum severity threshold (1-4)
    type: int
    required: false
    default: 3
  include-fixed:
    names: --include-fixed
    description: Include fixed issues
    type: boolean
    required: false
    default: false
```

**Option types:** `string` (default), `boolean`, `int`, `long`, `double`, `float`, `array`.

**Environment variable defaults:**
```yaml
  token:
    names: --token
    description: API token
    default: ${#env('MY_TOKEN')}
    mask:
      sensitivity: high
```

---

## Step 4: Implement action steps

Steps are the core logic. Each step is a single instruction. See [action-yaml-reference.md](action-yaml-reference.md) for the complete step instruction reference.

### Common patterns

#### Setting variables
```yaml
steps:
  - var.set:
      myVar: Hello ${cli.appversion}
      counter: 0
```

#### Making REST API calls (FoD or SSC)
```yaml
steps:
  - rest.call:
      issues:
        target: fod   # or ssc; defaults to config.rest.target.default
        uri: /api/v3/releases/${releaseId}/vulnerabilities
        type: paged   # automatically pages through all results
        records.for-each:
          record.var-name: issue
          do:
            - log.info: "Processing issue: ${issue.id}"
```

#### Running fcli commands

To discover what commands are available in the installed version of fcli, run:
```bash
fcli util all-commands list
```
Add `-h` to any command to see its options and usage (e.g., `fcli fod release get -h`) or to any module to see available commands (e.g., `fcli fod -h`).

```yaml
steps:
  - run.fcli:
      getRelease:
        cmd: fod release get ${cli.appversion}
        records.collect: true
  - var.set:
      releaseId: ${getRelease.records[0].releaseId}
```

#### Writing output to files
```yaml
steps:
  - out.write:
      ${cli.file}: ${#jsonStringify(results, true)}
```

#### Policy checks (pass/fail)
```yaml
steps:
  - check:
      CRITICAL_COUNT:
        display-name: "Critical issue count must be zero"
        failIf: ${criticalCount > 0}
        ifSkipped: PASS
```

#### Using writers for structured output (CSV, JSON, etc.)
```yaml
steps:
  - with:
      writers:
        csvWriter:
          to: ${cli.file}
          type: csv
    do:
      - rest.call:
          vulns:
            uri: /api/v3/releases/${releaseId}/vulnerabilities
            type: paged
            records.for-each:
              record.var-name: vuln
              do:
                - writer.append:
                    csvWriter:
                      id: ${vuln.id}
                      severity: ${vuln.severity}
                      category: ${vuln.category}
```

#### Session management (for actions that need their own session)
```yaml
steps:
  - with:
      sessions:
        - login: fcli fod session login --url ${#env('FOD_URL')} --tenant ${#env('FOD_TENANT')} --user ${#env('FOD_USER')} --password ${#env('FOD_PAT')} --fod-session=actionSession
          logout: fcli fod session logout --fod-session=actionSession
    do:
      - run.fcli:
          myCmd:
            cmd: fod release ls --fod-session=actionSession
```

#### Conditional execution
```yaml
steps:
  - if: ${cli.include-fixed}
    log.info: Including fixed issues in output
  - if: ${criticalCount > 0}
    do:
      - log.warn: "Found ${criticalCount} critical issues!"
      - exit: 1
```

#### Error handling
```yaml
steps:
  - rest.call:
      riskyCall:
        uri: /api/v1/some-endpoint
        on.fail:
          - log.warn: "REST call failed: ${lastException.message}"
          - var.set:
              fallbackUsed: true
```

#### Iterating over collections
```yaml
steps:
  - records.for-each:
      from: ${myCollection}
      record.var-name: item
      do:
        - log.info: "Processing: ${item.name}"
      breakIf: ${item.status == 'DONE'}
```

---

## Step 5: Use expressions effectively

Expressions use Spring Expression Language (SpEL) wrapped in `${ }`. Key patterns:

| Pattern | Example |
|---------|---------|
| Variable reference | `${myVar}` |
| CLI option | `${cli.file}` |
| Property access | `${issue.severity}` |
| Comparisons | `${count > 10}` |
| Ternary | `${count > 0 ? 'FAIL' : 'PASS'}` |
| String concat | `${firstName + ' ' + lastName}` |
| Null-safe | `${myVar?.property}` |
| Environment variable | `${#env('MY_VAR')}` |
| Format string | `${#fmt('Found %d issues', count)}` |
| Date formatting | `${#formatDateTime('yyyy-MM-dd')}` |
| Check if blank | `${#isBlank(myVar)}` |
| Join array | `${#join(', ', myArray)}` |
| JSON stringify | `${#jsonStringify(obj, true)}` |

See [action-yaml-reference.md](action-yaml-reference.md) for the full SpEL function reference.

---

## Step 6: Use formatters for complex output

Formatters are reusable templates declared at the top level and referenced in steps:

```yaml
formatters:
  sarifIssue:
    ruleId: ${issue.id}
    message:
      text: ${issue.issueName}
    level: ${issue.severity == 'Critical' ? 'error' : 'warning'}
    locations:
      - physicalLocation:
          artifactLocation:
            uri: ${issue.primaryLocationFull}

steps:
  - var.set:
      formattedIssue: {fmt: sarifIssue}
```

---

## Step 7: Test and validate

### Viewing built-in actions for reference
```bash
# View the YAML source of a built-in action
fcli fod action get <action-name>
fcli ssc action get <action-name>

# View help for an action
fcli fod action help <action-name>
```

### Running the action locally
```bash
# Run from a local YAML file
fcli fod action run /path/to/my-action.yaml --<options>
fcli ssc action run /path/to/my-action.yaml --<options>

# Run with debug logging
fcli fod action run /path/to/my-action.yaml --log-file action.log --log-level DEBUG --<options>
```

### Importing for persistent use
```bash
# Import a custom action
fcli fod action import --from /path/to/my-action.yaml

# Import all actions from a zip
fcli ssc action import --from /path/to/actions.zip

# Reset to remove all imported actions
fcli fod action reset
```

---

## Step 8: Sign and distribute

For production/CI/CD use, sign actions before distributing:

```bash
# Sign the action with a private key
fcli fod action sign my-action.yaml --with /path/to/private.key

# Import the corresponding public key for verification
fcli config public-key import --from /path/to/public.key
```

---

## Design guidelines

1. **Hard-code criteria, not parameters.** Instead of a highly configurable `check-policy` action where criteria are passed as parameters, hard-code the criteria into the action. This prevents thousands of pipelines from needing parameter updates when criteria change.
2. **Use application attributes for variation.** If different applications need different criteria (e.g., based on business risk), read SSC/FoD attributes like `Business Risk` to determine which criteria to apply.
3. **Keep actions focused.** One action per purpose. Prefer `check-high-risk-policy` and `check-low-risk-policy` over a generic `check-policy` with many parameters.
4. **Use `config.rest.target.default`** to avoid repeating `target:` on every `rest.call`.
5. **Prefer `run.fcli` over raw `rest.call`** when equivalent fcli commands exist — it's simpler and benefits from fcli's built-in handling.
