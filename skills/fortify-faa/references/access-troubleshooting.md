# Access verification failures

Load only after an access check fails. Diagnose the failing layer; do not rebuild a working setup or load unrelated configuration guidance.

User Guide: [Fortify Agentic Analyzer 26.4.0](https://www.microfocus.com/documentation/fortify-static-code-analyzer-and-tools/2640/fortify-agentic-analyzer-ugd-26.4.0.pdf). Section numbers below refer to this edition.

## Credential boundary

- Report the failed check and a sanitized error summary. Never request secrets in chat, read credential-store contents, or print credential environment values.
- Diagnose non-secret settings (selected backend, provider/model IDs, region, endpoint host, session name). Do not dump environment variables, policies, logs, or HTTP payloads that may contain secrets or source code.
- The user repairs authentication directly in their terminal, provider login, or CI secret store. Do not collect or supply passwords, tokens, API keys, or device codes through the conversation.
- Do not silently clear stored credentials, switch providers, weaken TLS verification, or change models. Explain any proposed non-secret configuration change and obtain approval before applying it.
- After repair, rerun the failed check and complete the remaining verification. If unresolved, stop and direct the user to the relevant guide section or OpenText support with a sanitized error summary.

## Fortify entitlement failure: `-license`

FAA reads FoD or SSC credentials from the environment; a logged-in fcli session does not satisfy this gate. Direct the user to **Section 3.5, Fortify entitlement**.

- Missing or rejected credentials: ask the user to configure or refresh **one** platform's set. FoD uses `FCLI_DEFAULT_FOD_CLIENT_ID` + `FCLI_DEFAULT_FOD_CLIENT_SECRET`, or `FCLI_DEFAULT_FOD_USER` + `FCLI_DEFAULT_FOD_PASSWORD` + `FCLI_DEFAULT_FOD_TENANT`. SSC uses `FCLI_DEFAULT_SSC_URL` + `FCLI_DEFAULT_SSC_TOKEN`, or the URL with `FCLI_DEFAULT_SSC_USER` + `FCLI_DEFAULT_SSC_PASSWORD`. Reference names only, never values.
- Wrong FoD region: confirm the endpoint host; tenants outside AMS need `FCLI_DEFAULT_FOD_URL` for their region.
- Connectivity or TLS error: identify the endpoint/network/trust failure without disabling certificate verification; have the user or administrator repair it.
- Stale entitlement result after credential repair: `fortifyaa -license clear` resets the 24-hour cache and re-checks. Otherwise use `fortifyaa -license`.

## Agent, provider, or LLM failure: `-selftest`

Use the same policy and overrides as the intended scan. Distinguish configuration failures from authentication or connectivity failures; `--no-live` cannot prove LLM access.

| Failure | Safe next action | User Guide |
|---------|------------------|------------|
| Missing Claude Code CLI or login | If the policy uses `agent=claude`, have the user restore their `claude` installation/login; Pi credentials do not repair this backend | Section 3.6 |
| No Pi credentials or expired login | Have the user refresh their provider's standard cloud-SDK/environment credentials, or run `fortifyaa -pilogin` themselves for a supported stored login | Sections 3.7.1-3.7.2 |
| Provider offers no configured model family | Compare provider/model IDs from no-live output; the default Claude families cannot be served by an OpenAI-only setup. Propose compatible non-secret settings, not a credential change | Section 3.7.4 |
| Invalid or incomplete policy | Identify named missing/invalid properties; load `references/policy-and-tuning.md` only if needed. Do not invent a partial policy | Chapters 5-6 |
| Endpoint, permission, quota, or rate-limit error | Report the specific layer; have the provider administrator resolve access, billing/quota, endpoint, or network restrictions. Do not repeatedly retry paid tests | Sections 3.7-3.8; Chapter 11 |
| Private model or gateway mismatch | Direct the user to custom model/provider configuration rather than guessing endpoint contracts | Chapter 7 |

For Pi, `fortifyaa -pilogin status` reports stored provider/kind/expiry without secrets; use it only when relevant. Never read the underlying credential file. Bedrock and Vertex use ambient cloud credentials, not `-pilogin`. For Claude Code, authentication belongs to that CLI.

After repair, rerun `fortifyaa -selftest --no-live`, then the live `fortifyaa -selftest`. Live tests make small paid LLM calls; explain this before running them.

## fcli or target-access failure (live deduplication only)

This is separate from FAA entitlement and LLM access. Apply only after selecting `--fod-release` or `--ssc-appversion`; local FPR and FAA-only scans do not require fcli.

- fcli unavailable: activate `fcli-common` for installation guidance.
- Missing/expired session: activate `fcli-common`; have the user authenticate directly in their terminal. Check only the selected platform's session list afterward.
- Wrong named session: confirm the intended session and select it with `--fcli-fod-session <name>` or `--fcli-ssc-session <name>`.
- Missing, ambiguous, or forbidden target: activate `fortify-fod` or `fortify-ssc` to resolve the release/application version and verify read access. Do not create a replacement target or silently omit deduplication.

See User Guide **Section 3.2** (fcli prerequisite) and **Section 4.4** (complementary scanning). Resume scan planning only after the selected session and target access are verified.