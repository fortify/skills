---
name: fortify-faa
description: >-
  Run an OpenText Fortify Agentic Analyzer (FAA) scan — an AI/LLM-driven,
  agentic SAST scan of a local source checkout. Use when the user wants to
  "run FAA", "run an agentic analyzer / agentic AST scan", "deep AI scan this
  codebase", or find design/business-logic flaws traditional SAST misses.
  Covers verifying the fortifyaa install, Fortify entitlement and LLM access, the
  self-test, running the (paid) scan with FoD/SSC deduplication, and uploading
  results. NOT for the deterministic sourceanalyzer/ScanCentral SAST scan
  (use fortify-fod / fortify-ssc) or a free local change review
  (use fortify-change-review).
license: MIT
metadata:
  version: "1.0.0"
  tested-with:
    faa: "26.4"
    fcli: "3.23"
---

# Fortify Agentic Analyzer (FAA)

OpenText™ Fortify Agentic Analyzer drives an autonomous coding agent (backed by an LLM) over a local source checkout to find vulnerabilities that deterministic SAST misses — flawed authZ/authN designs, business-logic flaws, and subtle injection patterns — on effectively any language. It emits standard **SARIF 2.1.0** for consumption by Fortify on Demand (FoD), Fortify SSC, or any SARIF-aware tool.

Docs: [Fortify Agentic Analyzer User Guide 26.4.0](https://www.microfocus.com/documentation/fortify-static-code-analyzer-and-tools/2640/). The executable is `fortifyaa` (`fortifyaa.exe` on Windows). Verbs use a single dash (`-scan`, `-selftest`, `-license`), options a double dash (`--output`, `--scope`).

> **⚠ FAA costs real money, every run.** Each scan makes paid, bring-your-own LLM calls. A full scan of a real application can cost **tens to hundreds of US dollars**. It is a deliberate, risk-based complement to SAST — not a per-commit gate. Never start a scan without (a) a self-test passing and (b) explicit user confirmation of the scope and cost cap (Step 2).

> **FAA always runs locally**, against a checkout of the code. The scan is code-resident and the LLM spend is yours. Stay inside this workflow; don't improvise side-quests.

Work the steps in order — each has a gate that must pass before the next.

---

## Step 0: Verify FAA is installed

FAA is a separate native executable; it is **not** bundled with fcli or installed by this skill. Probe for it first (no cost):

```bash
fortifyaa -help
```

- **Found** → continue to Step 1.
- **Not found / not on PATH** → direct the user to obtain FAA from the **OpenText software download portal**, or to **contact OpenText support / their Fortify account team** if they lack access. **Stop the workflow here; do not download or install FAA.** If it is already installed elsewhere, the user can provide its full path; verify that executable with `-help` before continuing and use that path in subsequent commands.

### Step 0 gate
- [ ] `fortifyaa -help` succeeds (on PATH or via a user-provided full path)

---

## Step 1: Verify Fortify entitlement and LLM access

Verify the existing setup on every workflow run; **do not load setup guidance when checks pass**. FAA entitlement uses environment credentials, not an fcli session. The self-test verifies the configured coding agent, provider, and models.

Run these checks in order, stopping at the first failure:

```bash
fortifyaa -license
fortifyaa -selftest --no-live
```

The license check may reuse its 24-hour cache. The no-live self-test lists provider candidates and per-stage models without paid calls; it does not prove connectivity. Tell the user that the following live verification makes small paid LLM calls, then run:

```bash
fortifyaa -selftest
```

Use the same `--policy`, `--config`, and `-D` settings intended for the scan. If agent, provider, or model settings change later, repeat the self-tests with those settings before scanning.

**Only on failure**, load `references/access-troubleshooting.md` and consult the matching failure section. Identify what broke, provide safe diagnostic guidance and the relevant User Guide section, then pause for user-managed credential repair when needed. Rerun the failed verification after repair; do not scan until all checks pass.

> Never request secrets in chat, read credential-store contents, or print credential environment values. The user configures credentials directly in their terminal, provider login, or secret store; the agent diagnoses non-secret settings and reports sanitized errors only.

### Step 1 gate
- [ ] `-license` confirms Fortify entitlement
- [ ] `-selftest --no-live` lists the expected provider and per-stage models
- [ ] Live `-selftest` passes with the intended scan settings

---

## Step 2: Plan the scan — scope, deduplication, and cost (confirmation gate)

**Only after Steps 0 and 1 pass**, load `references/running-a-scan.md` and follow its planning checklist for scope, complementary deduplication, policy cost cap, and outputs (including AI-friendly Markdown for remediation).

**Never edit or override `maxcost`.** If the user wants a different cap, direct them to manually edit the selected policy file. Present the planned command and existing policy cap, then wait for explicit scan confirmation.

### Step 2 gate
- [ ] Reference planning checklist complete
- [ ] **User has explicitly confirmed the scan and its cost cap** — do not run until confirmed

---

## Step 3: Run the scan

Execute the confirmed command. The scan runs the plan → analyze → validate pipeline and writes the output.

- **Stopped, paused, or incomplete?** End this workflow here, including when the user intentionally stops/pauses it or a budget/error interruption prevents completion. Report the status and saved-state/resume guidance; do not automatically restart an intentionally stopped scan. **Skip Step 4 and do not load its reference.** Existing artifacts from earlier runs are not evidence that this scan completed.
- **Cost/progress** is printed per stage as it runs and summarized at the end. A stall warning names any silent batch.
- **Resumable.** A crash, LLM error, or `maxcost` stop never loses finished work — re-running `-scan` auto-resumes and does not re-pay for completed work. `-resume -scan <dir>` resumes explicitly; `-clean` discards saved state for a fresh scan. The cap counts work already spent across resumes.
- **Hung scan?** Re-run with `--debug` for a timestamped timeline; see troubleshooting in `references/policy-and-tuning.md`.

### Step 3 gate
- [ ] The current scan fully completed successfully and its output artifact(s) exist; otherwise end here

---

## Step 4: Upload scan results

**Only after the Step 3 completion gate passes**, load `references/upload-scan-results.md` and follow its upload checklist. Do not load it for stopped, paused, failed, or otherwise incomplete scans.

### Step 4 gate
- [ ] Reference upload checklist complete

---

## Reference files

| File | Load when |
|------|-----------|
| `references/access-troubleshooting.md` | Only after entitlement, self-test, or selected live-deduplication access verification fails; consult the matching failure section |
| `references/running-a-scan.md` | Step 2 only, after install and access checks pass: planning checklist, scan options, outputs, user-managed cost cap, and resume |
| `references/upload-scan-results.md` | Only after the current scan fully completes successfully: upload checklist, FoD/SSC upload, conversion, and rendering options |
| `references/policy-and-tuning.md` | Tuning the policy file (per-stage models, effort, stages, cost), and troubleshooting/log locations |

## Cross-Skill Dependencies

| Skill | Used for |
|-------|----------|
| `fcli-common` | fcli installation check and session management (required for `--fod-release` / `--ssc-appversion` dedup) |
| `fortify-fod` | Resolving the FoD release and importing the resulting SARIF |
| `fortify-ssc` | Resolving the SSC application version and uploading the FAA zip |
| `fortify-remediate` | Fixing the SAST-style findings FAA reports once they are in FoD/SSC |
