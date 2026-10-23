# Policy file tuning and troubleshooting

Everything that tunes **how** a scan behaves lives in an external `.properties` policy file, never in the binary. A bare `-scan <dir>` is driven by `fortify-aa-default.properties` (shipped next to the executable, tuned for a deep complementary scan).

**Cost caps are user-managed.** Never edit or override `maxcost`, including through `-D` or policy substitution. If the user wants to change it, direct them to manually edit the selected policy file, verify the resulting cap, and obtain scan/resume confirmation.

## Selecting a policy

| Form | Resolves to |
|------|-------------|
| (default) | `fortify-aa-default.properties`, next to the executable |
| `--policy <name>` | `fortify-aa-<name>.properties`, next to the executable |
| `--config <path>` | the explicit `.properties` file at `<path>` |
| `-Dkey=value` | override a single property after the file loads (last one wins, repeatable) |

**Every property is required** — there are no built-in defaults. An absent property (even for a disabled stage) is a startup error. The whole configuration is validated up front, **before** the license check or any LLM spend, so you never pay only to hit a config error. The effective merged config is embedded in the output SARIF for traceability.

## Global settings

| Property | Default | Meaning |
|----------|---------|---------|
| `agent` | `pi` | Coding-agent backend: `pi` (bundled, in-process) or `claude` (your Claude Code CLI). A `fake` backend exists for no-cost plumbing tests. |
| `pi.provider` | `auto` | Pi only: which provider serves the model. `auto` probes candidates at scan start and uses the first that answers; or pin a concrete id. Only if access verification fails, consult `references/access-troubleshooting.md`. |
| `maxcost` | `50` | Soft USD cap for the **whole** scan; resumes continue the same total. `0` disables. Only the user may manually adjust this policy setting. |
| `output.transcripts` | `false` | Persist full agent conversations under the build dir's `transcripts/`. Adds up on disk (embeds source read). `--debug` turns it on. |

## File enumeration (always runs, before any LLM work)

A deterministic walk drops files not worth analyzing — this runs even when the plan stage is off.

| Property | Meaning |
|----------|---------|
| `files.excludedirs` | Directory names skipped wholesale at any depth (default: `.git`, `node_modules`, `target`, `build`, `dist`, `.idea`, …). |
| `files.excludeextensions` | Extensions treated as non-source (images, media, archives, binaries, data/lock/map files). |
| `files.excludenames` | Glob patterns matched against file names (minified JS/CSS, lockfiles, wrapper launchers). |
| `files.maxbytes` | Skip files larger than this (default `1000000`). Empty files are always skipped. |
| `files.excludegitignored` | Also skip git-ignored files (default `true`; needs `git` on PATH — set `false` for non-repo trees). |

## Per-stage model selection and effort

A scan is a pipeline of up to three stages, each selecting its model and effort independently via `<stage>` = `plan` / `analysis` / `validation`.

| Property | Meaning |
|----------|---------|
| `<stage>.effort` | Thinking level: `low`, `medium`, `high`, `max`. Affects reasoning depth, not batching. |
| `<stage>.model.family` | Model family, e.g. `opus`, `sonnet`, `haiku`. Matched without a provider prefix (`gpt-5.1`, not `openai/gpt-5.1`). Available families depend on the agent + provider. |
| `<stage>.model.version` | `latest`, a `major.minor` (e.g. `4.6`), or a bare major. |
| `<stage>.model.context` | Context-window preference: `default`, `max`, `1m`, `200k`. |

The default policy matches each stage to its job: `haiku` for the mechanical **plan** pass, `sonnet` for the heavy **analysis** stage, `opus` for the judgment-heavy **validation** stage. Analysis runs at `effort=high` (recall-critical); plan and validation at `medium`.

## The three stages

- **Plan** (`plan.enabled`, default `true`) — optional LLM pass that prioritizes enumerated files and skips low-value ones. When `false`, every enumerated source file is analyzed. Tunable: `plan.effort`, `plan.batchsize` (25), `plan.maxconcurrency` (16).
- **Analysis** (`analysis.enabled`, default `true`) — the stage that actually hunts vulnerabilities; coverage is guaranteed by the harness, not the agent's memory. Tunable: `analysis.effort` (high), `analysis.minbatchsize` (3), `analysis.maxbatchsize` (30), `analysis.maxconcurrency` (16). Turning it off while plan is on yields a plan-only diagnostic that writes no SARIF.
- **Validation** — an LLM pass that deduplicates (including against supplied existing results), drops false positives, and aligns categories to the Fortify taxonomy. Only after it completes are results written.

Each stage can be enabled/disabled individually — e.g. a cheap plan-only diagnostic, or skipping validation to save cost at the expense of quality.

## Example: a budget-capped complementary scan

Uses the existing cap in the selected policy; no cost-cap override:

```bash
fortifyaa -scan . \
  --fod-release "My App:Release 1" \
  --output faa.sarif --output-format sarif,zip \
  -Danalysis.model.family=sonnet \
  -Dvalidation.model.family=opus
```

> Note: resuming a saved scan is refused if you change a model, effort, batch sizing, concurrency, or a file-selection rule — those invalidate the saved state. `maxcost` and `output.transcripts` are the only exceptions. `-clean` forces a fresh (re-paid) scan.

## Custom models and extensions

If your model/endpoint is not in Pi's built-in catalog, declare it in `pi_models.json` next to the executable (or `--pi <path>`). If reaching it needs code (e.g. an API-gateway HMAC signature), add a Pi extension under `pi_extensions/`. See FAA User Guide, Chapter 7.

## Troubleshooting

- **Log files.** The default run log is `log/faa.log` under the Fortify working directory; `--log <path>` writes a complete console mirror elsewhere. Scan state and `debug.log` live under the per-project build dir (`%LOCALAPPDATA%\Fortify\faa26.4\build\<build-id>` on Windows, `~/.fortify/faa26.4/build/<build-id>` elsewhere).
- **Scan appears hung.** The stall warning names any batch with no model activity for several minutes; a unit idle for 15 minutes is aborted so the scan can be resumed instead of hanging. Re-run with `--debug` for a per-batch timeline; on an LLM call failure the exact request payload and HTTP response are dumped to `llm-failures/<unit>.json`.
- **Stage stops on first error.** Stages halt on the first LLM/agent error with a human-readable message rather than silently degrading. Fix the cause and re-run `-scan` to resume completed work.
- **Config error at startup.** All config problems are reported at once, before any spend. Fix the named properties and re-run.
- **Report issues / request enhancements.** Via the normal OpenText support channels (FAA User Guide, Chapter 11).
