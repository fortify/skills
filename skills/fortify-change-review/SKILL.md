---
name: fortify-change-review
description: Lightweight, AI-powered security review of code CHANGES (a diff, PR, or in-session edits) using the agent's own analysis — no Fortify scan engine or platform needed. Use when the user asks to "run a Fortify security review" / "Fortify change review", or when adding/modifying code touching authentication, authorization, input handling, output encoding, data access, file operations, outbound HTTP, deserialization, cryptography, transport security, XML parsing, logging, LLM/agent logic, or IaC (AWS, Azure, GCP, Terraform, Kubernetes, Helm). NOT for full SAST/DAST/SCA scans or triaging issues already found by FoD/SSC (use fortify-fod / fortify-ssc).
license: MIT
metadata:
  version: "1.0.0"
---

> **Scope:** Load only checks relevant to the specific change in front of you. Do not audit the full codebase, load references speculatively, or surface security advice outside these steps.

---

## Step 0: Identify the Change Set

Determine which path applies and follow it before proceeding to Step 1.

### Step 0a — In-session change

The skill was triggered because the AI coding agent is actively writing or modifying code in this conversation. The change set is the code being generated or edited right now. Proceed directly to Step 1.

### Step 0b — Explicit review request

A security review has been explicitly requested — by a user, an orchestrator, or a CI/CD pipeline. Determine whether the change set (diff and/or file list) has already been provided in the prompt:

- **Yes — changes are provided:** The change set is identified and in scope. Proceed directly to Step 1.
- **No — changes not provided:** Load `references/identify-changes.md` and follow its discovery workflow to identify and confirm the change set before proceeding to Step 1.

### Step 0 gate

- [ ] One path selected (0a or 0b)
- [ ] Change set is identified and in scope
- [ ] For Step 0b without pre-supplied changes: change set confirmed with user before proceeding

---

## Step 1: Map Change to Checks

Scan the change against the table below — each matched row is a check candidate. Match on intent (e.g., "user asked to add login") as well as code patterns.

| Signal | Check |
|--------|-------|
| New endpoint, route, controller, resolver, or handler | `references/sc-authorization.md` |
| Resource accessed by userId / tenantId / accountId from request | `references/sc-authorization.md` |
| Admin action added, or authorization / validation check bypassed ← high-risk | `references/sc-authorization.md` |
| Database query, ORM filter, or search/sort/filter incorporating external input | `references/sc-injection.md` |
| `eval`, shell execution, dynamic command construction, or deserialized object from external source | `references/sc-deserialization.md` |
| `innerHTML`, `dangerouslySetInnerHTML`, `v-html`, template rendering, or Markdown/HTML rendering of user data | `references/sc-xss-output.md` |
| File upload, download, filename, or path derived from user input | `references/sc-file-path.md` |
| Archive extraction with user-supplied archives | `references/sc-file-path.md` |
| User-controlled URL, webhook, import-from-URL, or outbound fetch by input | `references/sc-ssrf.md` |
| Login, password reset, session management, OAuth/OIDC/SAML, or token generation | `references/sc-authentication.md` |
| Error handler, exception handler, or HTTP error response construction | `references/sc-logging-audit.md` |
| Log statement that includes request data, headers, or user-supplied input | `references/sc-logging-audit.md` |
| LLM system prompt, agent tool definition, RAG ingestion, or function-calling capability | `references/sc-ai-agent-safety.md` |
| TLS disabled, `InsecureSkipVerify`, custom TrustManager, cipher config, or key management logic | `references/sc-crypto-transport.md` |
| XML parsing, SOAP endpoint, XML file upload/import, or DTD processing | `references/sc-xxe.md` |
| AWS CloudFormation template, Ansible AWS task, or Terraform `resource "aws_*"` | `references/sc-iac-aws.md` |
| Azure ARM template, Bicep file, Ansible Azure task, or Terraform `resource "azurerm_*"` | `references/sc-iac-azure.md` |
| GCP Terraform with `resource "google_*"` or `provider "google"` | `references/sc-iac-gcp.md` |
| Kubernetes YAML manifest, Helm chart template, or static Pod spec | `references/sc-iac-kubernetes.md` |

**If no rows matched:** stop. Do not load any reference files or apply checks. Do not produce a Secure Change Review. Resume normal code generation.

### Step 1 gate

- [ ] At least one row matched
- [ ] Up to 3 checks selected; when more than 3 match, prioritize: authorization → injection → others
- [ ] Selected check files loaded
- [ ] `references/apply-checks.md` loaded — proceed to Steps 2 and 3 in that file
