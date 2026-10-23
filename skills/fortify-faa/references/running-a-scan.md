# Running a scan

The full `-scan` command reference. Verbs take a single dash; options a double dash. Short forms: `-o` = `--output`, `-b` = `--build-id`, plus `-D` property overrides.

## Step 1: Plan and confirm the scan

Load this reference only after installation and access verification pass in `SKILL.md`. Work through the following with the user before executing a scan:

1. **Scope.** Confirm the project checkout and full, scoped, or change-driven analysis. Narrowing scope reduces cost and coverage; do not silently exclude code to fit a budget.
2. **Deduplication.** Strongly prefer traditional Fortify SAST results from the same checkout: one of `--fod-release`, `--ssc-appversion`, or `--fpr`. If none is available, confirm FAA-only scanning without deduplication.
3. **Live-source access, only if selected.** Run `fcli --version` and the selected platform's `fcli fod session list` or `fcli ssc session list`. Verify the intended session and resolve the target with `fortify-fod` or `fortify-ssc`. FAA entitlement does not establish session or target access. On failure, consult only the fcli section of `references/access-troubleshooting.md`. Skip these checks for local FPR and FAA-only scans.
4. **Existing cost cap.** Identify the selected policy and its `maxcost` without dumping unrelated settings or secrets. The agent must not edit or override `maxcost`, including with `-D`, a different policy, or a generated config. If the user wants an adjustment, direct them to manually edit `maxcost` in that policy file; verify the resulting cap afterward. Do not assume the shipped default is the effective cap.
5. **Outputs.** Keep SARIF (or zip) for full-fidelity storage/conversion, add zip for SSC, and select AI-friendly Markdown for remediation as needed. Prefer `md-per-finding` for an agent processing one finding at a time; `md-full` provides all finding details in one document.
6. **Confirmation.** Present the exact command, scope, deduplication source, outputs, and policy cap. Explain that the cap is approximate, not a guaranteed maximum bill. Wait for explicit confirmation before scanning. If policy agent/provider/model settings changed, repeat the access self-tests with the intended settings first.

### Step 1 gate
- [ ] Scope and checkout confirmed
- [ ] Deduplication source selected or FAA-only explicitly accepted
- [ ] For live sources only: selected fcli session and target read access verified
- [ ] Existing policy cap verified and accepted; any adjustment made manually by the user
- [ ] Outputs selected, keeping SARIF or zip and considering Markdown for AI remediation
- [ ] Exact scan command and cost confirmed by the user

## Basic scan

```bash
fortifyaa -scan /path/to/project
```

Reads the default policy (`fortify-aa-default.properties`, next to the executable), runs the plan → analyze → validate pipeline, and writes `faa.sarif` into the current directory. Token usage and estimated cost print per stage and are summarized at the end. A stall warning names any batch with no model activity for several minutes.

## The recommended workflow

1. Run your normal Fortify SAST scan and obtain its results (FPR, FoD release, or SSC app version).
2. Run FAA locally against the **same checkout**, giving it those results so it reports only complementary findings.
3. Upload FAA's SARIF to your Fortify platform alongside the SAST results.

An **FAA-only** scan (omit the existing-results option) is fully supported — e.g. for a language traditional SAST does not cover — it simply forgoes deduplication.

## Scope

```bash
fortifyaa -scan /path/to/project --scope "src/auth,src/payments,config/web.xml"
```

`--scope` restricts to comma-separated directories and/or files (relative to the project, or absolute). A scoped **directory** is walked and filtered like a full scan; an individually named **file** is always analyzed (bypasses file filters, gitignore, and the plan stage). Scoping cuts both cost and time.

## Complementary scanning (deduplication) — strongly preferred

Give FAA your existing Fortify SAST results so anything SAST already found is dropped and you see only what SAST missed. Provide **exactly one** source (mutually exclusive):

```bash
# Local FPR
fortifyaa -scan . --fpr existing.fpr --output faa.sarif

# FoD release (live via fcli; needs an FoD session)
fortifyaa -scan . --fod-release "My App:Release 1" --output faa.sarif

# SSC application version (live via fcli; needs an SSC session)
fortifyaa -scan . --ssc-appversion "My App:1.0" --output faa.sarif
```

`--fod-release` / `--ssc-appversion` accept an id or the `"App[:Microservice]:Release"` / `"App Name:version"` form. Select a non-default session with `--fcli-fod-session <name>` / `--fcli-ssc-session <name>`. Both live sources require fcli installed and the selected platform's session and target access verified. Only if these checks fail, consult the fcli section of `references/access-troubleshooting.md`.

## Output files

SARIF is canonical and the **only** convertible format — every other output derives from it one-way. **Keep the SARIF** (or the zip, which contains it): with it you can produce any other format later for free; without it the only way back is to pay for another scan.

`--output-format` selects artifacts (comma-separated, case-insensitive):

| Value | What |
|-------|------|
| `sarif` | SARIF 2.1.0 — the default, consumable by FoD natively |
| `zip` | SARIF wrapped with a `scan.info` descriptor (`engineType=FORTIFY_AA`) for SSC upload |
| `md-summary` | Compact Markdown overview for triage; omits the full explanation and remediation detail |
| `md-full` | AI-friendly Markdown with full explanations and remediation guidance for all findings |
| `md-per-finding` | AI-friendly remediation input: one self-contained finding document with metadata, explanation, and fix guidance, plus `index.md` / `index.json` |

Markdown is designed for people and AI agents, especially remediation workflows that consume a finding at a time. It complements, rather than replaces, canonical SARIF; keep SARIF or zip so reports can be regenerated without another paid scan.

`--output <file>` names the SARIF and every other artifact derives from it (`<name>.zip`, `<name>-summary.md`, …). Override one with `--output-<format> <path>` (e.g. `--output-md-per-finding .faa/issues`). A run selecting only derived formats is **refused** unless you pass `--no-sarif` (it would throw its results away). Missing directories are created.

Other output options:
- `--message-format fod|plain` — `fod` (default) renders the plain-text channels as HTML for FoD; `plain` emits plain text for verbatim consumers. Does not affect Markdown or SSC.
- `--latin1` — restrict all output to Windows-1252 (for SSC on MS SQL Server, which cannot store other characters). Best-effort; lossy. Prefer applying it at upload time via `-convert` when the same scan serves multiple consumers.

## Cost management

Every scan spends real money. The policy `maxcost` property (shipped default **$50**) is a soft USD cap on the **whole** scan, including resumes, not per invocation. The agent must not change or override it. If the user wants to adjust the cap, have them manually edit `maxcost` in the selected `.properties` policy file and then verify their change. A value of `0` disables the cap; flag an existing uncapped policy explicitly during confirmation rather than silently accepting unlimited spend.

The first analysis batch runs alone to calibrate per-batch cost; thereafter batches dispatch only while the projected total (including in-flight work) stays within the cap, with a slice held back for validation. Treat it as a close approximation, not a hard guarantee. Model choice per stage and which stages run also drive cost — see `references/policy-and-tuning.md`.

## Scan state and resume

Scans are resumable. Every completed batch/validation unit is checkpointed to a per-project build directory under the Fortify working directory (`%LOCALAPPDATA%\Fortify\faa26.4` on Windows, `~/.fortify/faa26.4` elsewhere). The build id derives from the project path; override with `-b <build-id>`.

```bash
fortifyaa -resume -scan /path/to/project   # continue an interrupted scan (errors if nothing to resume)
fortifyaa -clean -scan /path/to/project    # discard state, scan fresh
fortifyaa -clean                           # just discard saved state
```

- A crash, LLM error, or `maxcost` stop never loses finished work; re-running `-scan` auto-resumes and does not re-pay for completed work.
- **`maxcost` caps the whole scan across resumes** — resuming a capped-out scan with the same cap fails immediately. Stop and report the spent amount and cap. Only the user may manually adjust the policy cap; a new cap is the whole-scan total, not an extra allowance. Reconfirm before resuming. Do not use `-clean` or a new build id to bypass an exhausted budget; a fresh scan pays again for completed work.
- FAA refuses to resume if the project, scope, existing-findings source, or policy changed (it says what differs; `-clean` to override). Exempt: `maxcost` (raising is supported) and `output.transcripts`.

## Keep findings stable across scans — `--baseline`

Each finding carries an **instance id** (derived from file, sink line, and category) that consumers use to recognize it across re-scans. FAA already anchors the sink by matching line text (not number) and records the category only after taxonomy alignment. `--baseline <prior.sarif>` closes the rest of the gap: given a previous scan's SARIF, FAA reuses the earlier id for findings it confidently matches (same file+category within a few lines, or the same sink line verbatim). Valid in **any** scan, including a plain full one:

```bash
fortifyaa -scan . --baseline previous.sarif --output current.sarif
```

The baseline is read-only; a baseline that can't be read warns and the scan continues with fresh ids.

## Change-driven scans

Both require a git repo and scope the paid analysis to changed files. The `<range>` is anything `git diff` accepts (`base...head` for a PR, `from..to`, `<sha>~1..<sha>`, `HEAD` for working-tree edits).

| Mode | Scans | Output | Baseline |
|------|-------|--------|----------|
| `--incremental <range>` | changed files | a **complete** SARIF: the baseline updated for changed files (untouched findings carried over) | **required** |
| `--differential <range>` | changed files | only issues **introduced** on changed lines | optional (adds id continuity) |

```bash
# A PR's worth of changes, full updated report:
fortifyaa -scan . --baseline prev.sarif --incremental main...HEAD --output updated.sarif

# Only what this PR introduced (good for gating) — no baseline needed:
fortifyaa -scan . --differential main...HEAD --output introduced.sarif
```

`--incremental` and `--differential` are mutually exclusive. **Limitation:** both analyze only changed files, so a change whose sink lands in an unchanged file is not re-derived — a full baseline scan remains the way to catch cross-file drift.

## Other useful options

- `--policy <name>` / `--config <path>` — select a shipped/custom policy, or an explicit `.properties` file.
- `-Dkey=value` — override a single policy property (repeatable), e.g. `-Dplan.enabled=false`.
- `--log <path>` — write the full run log (mirrors console) to a path instead of `log/faa.log`.
- `--debug` — write a timestamped diagnostic timeline + agent transcripts to `debug.log` in the build dir (and LLM request/response dumps on failure). Use when a scan appears to hang or misbehave.
- `--keep-scratch` — keep the temporary scratch directory for debugging.
