# Upload scan results

Load only after the current scan fully completes successfully and its output artifacts exist (Step 3 gate in `SKILL.md`). Do not upload stale artifacts from an earlier scan after a stopped, paused, or failed run.

FAA's output is standard **SARIF 2.1.0**. The scan runs locally; only the resulting file is uploaded to the Fortify platform.

## Step 1: Upload the completed scan's results

1. Identify the completed run's output paths and actual scan command. Keep the canonical SARIF or zip.
2. **Reuse the live deduplication target (best practice).** If the scan used `--fod-release`, upload to that same FoD release. If it used `--ssc-appversion`, upload to that same SSC application version. Reuse the resolved target ID, platform endpoint, and session (including `--fcli-fod-session` / `--fcli-ssc-session` when specified); do not substitute a similarly named target or ask the user to choose a destination again. These scan options are mutually exclusive, so reuse whichever was used, not both. If that target is inaccessible, report the blocker rather than silently uploading elsewhere.
3. **No live deduplication target?** For an FAA-only scan or one using `--fpr`, ask whether the user wants to upload results to FoD or SSC. Do not infer a destination from the FPR or an active session. If they decline, report the local artifact paths and end successfully without uploading. If they agree, ask for the platform and release/application version, then resolve the target and verify the selected session and read access through `fortify-fod` or `fortify-ssc` before upload. Do not create a new target without a separate request.
4. **FoD:** activate `fortify-fod` to import the SARIF into the reused or user-selected release; no parser plugin is required.
5. **SSC:** confirm the dedicated FAA parser plugin is installed, then activate `fortify-ssc` to upload the zip to the reused or user-selected application version. If only SARIF was produced, use `-convert` below; do not run another paid scan to obtain the zip.
6. Verify the platform import/upload succeeded and report the artifact paths and target. If upload is blocked, report the blocker and retain the local results without claiming upload succeeded.

### Step 1 gate
- [ ] Completed run's canonical output retained
- [ ] Live deduplication target reused, or upload preference and destination obtained when no live target was used
- [ ] Results imported into FoD, or the SSC zip uploaded with the FAA parser plugin installed; alternatively, user declined upload and local artifact paths were reported

## Fortify on Demand (FoD)

FoD supports FAA results **natively via SARIF import — no plugin**. Import the SARIF (default `faa.sarif`) into the target release through FoD's SARIF import. Findings appear with rule metadata, severity, and locations intact.

FoD renders the plain-text channels as HTML: the one-line summary (`message.text`) and description on the **Vulnerability** tab, remediation under **Recommendations**, and the snippet on the **Code** tab. FAA's default `--message-format fod` shapes them as HTML so they display well — no configuration needed. (If the same SARIF also feeds a verbatim consumer, produce a `--message-format plain` copy for that one.)

## Fortify Software Security Center (SSC)

Upload FAA results to SSC through a **dedicated parser plugin** (`fortify/fortify-ssc-parser-faa`, engine type `FORTIFY_AA`). Confirm the plugin is installed in SSC before uploading the completed scan's zip artifact (default `faa.zip`) to the selected application version. If the plugin is absent, direct the user to their SSC administrator before attempting upload.

The plugin renders the Markdown variants, so `--message-format` does not affect SSC. If your SSC runs on **MS SQL Server**, use `--latin1` (at scan or convert time) so the import cannot fail on character-set grounds.

## Prepare a missing upload artifact

You do not need to re-scan. `-convert` produces any output format from an existing SARIF (a `.sarif`, or a `.zip` containing one) with **no LLM calls and no cost**:

```bash
fortifyaa -convert faa.sarif --output-format zip                     # SSC upload artifact
fortifyaa -convert archive/run7.sarif --output-format md-summary,md-full
fortifyaa -convert archive/run7.sarif --output-format zip --latin1 \
  --output archive/run7-ssc.sarif                                    # restricted copy for SSC/MSSQL
```

`--output-format` is required (no default). The input is never modified or consumed; FAA refuses a conversion that would overwrite its own input, and warns if the SARIF came from a non-FAA tool (the reports rely on Fortify-specific content).