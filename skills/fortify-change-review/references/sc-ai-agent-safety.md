# AI-Agent and LLM Application Safety

## Risks

| # | Risk | Detection signal | Fortify Category | CWE | OWASP 2025 | Action |
|---|------|-----------------|-----------------|-----|------------|--------|
| R1 | Excessive agency — agent can perform destructive, irreversible, or high-impact actions without scope restriction or confirmation | Tool definition includes `bash`, `shell`, `eval`, file system write, send/messaging, financial, or code execution capabilities; no confirmation requirement before destructive or irreversible operations; tool list not limited to minimum needed for the task | Excessive Agency | CWE-285 | A01 Broken Access Control | Actions 1–3, 6, 8 |
| R2 | Insecure tool calling — user-controlled or LLM-generated content passed to tool invocations without validation | User input or LLM output used as a tool argument, API payload, database query, or shell command without schema validation or sanitization; tool invocation bypasses the injection and deserialization checks appropriate for the downstream use | Insecure Tool Calling | CWE-285 | A01 Broken Access Control | Actions 1, 4 |
| R3 | Prompt injection (direct or indirect) | User-controlled input or content retrieved from external sources (web, documents, vector store) is incorporated into an LLM prompt in a way that can override system instructions, alter agent behavior, or exfiltrate the system prompt | Prompt Injection | CWE-77 | A01 Broken Access Control | Actions 5, 7 |

## Scoped Exclusions

Do not report these patterns even if a detection signal above matches:

- *(R3)* **User content in the user role** — including user-controlled input in the user-role portion of an LLM prompt is not a prompt injection finding. Only flag when user input is placed directly in the system prompt, or when the application design provides no structural boundary between user-supplied content and trusted agent instructions, enabling user content to override agent behavior or cause system prompt exfiltration.

## Required Agent Actions

1. **Define an explicit tool allowlist** *(R1, R2)* — the agent must only have access to tools it genuinely needs for the task. Enumerate permitted tools. Do not grant broad filesystem, network, or shell access unless the use case specifically requires it and the scope is tightly bounded.

2. **Apply least-privilege tool scopes** *(R1)* — for each tool, grant the minimum scope:
   - Read-only access unless write is required
   - Scoped to specific resources/paths, not global access
   - No ability to modify agent configuration, system prompts, or tool definitions

3. **Require human confirmation for high-impact actions** *(R1)* — any agent action that is destructive, irreversible, financially significant, or sends external communications (email, webhook, API mutation) must pause and request explicit human approval before execution.

4. **Validate and sanitize LLM output before use** *(R2)* — if LLM output is used as code, a database query, a shell command, or an API payload:
   - Do not execute it directly
   - Parse and validate it against a strict schema or AST
   - Run it in a sandbox or with limited permissions
   - Apply the appropriate injection check (`sc-injection.md`, `sc-deserialization.md`) to the downstream use

5. **Protect system prompt confidentiality** *(R3)* — do not instruct the model to "never reveal the system prompt" as a sole control (models can be instructed to ignore it). Instead, design the application so the system prompt is never returned in a user-visible response path.

6. **Enforce RAG data boundaries** *(R1)* — retrieval results must be scoped to the requesting user's authorized data. A user's query must not retrieve documents belonging to other users or tenants. Apply the same access control model as the underlying data store.

7. **Treat retrieved content as untrusted input** *(R3)* — text retrieved from external sources (web, documents, databases, vector stores) may contain prompt injection payloads. Do not pass retrieved content into the system role. Mark retrieved content clearly as external data in the prompt structure.

8. **Set token and action chain limits** *(R1)* — enforce `max_tokens` on model calls. For agentic loops, set a maximum iteration/step count to prevent runaway agent execution.

## Completion Evidence

- [ ] *(R1, R2)* Tool allowlist is explicit — only tools required for the task are granted
- [ ] *(R1)* Each tool scope is minimized (read-only where possible, scoped to specific resources/paths)
- [ ] *(R1)* Destructive, irreversible, or high-impact actions require explicit human confirmation before execution
- [ ] *(R2)* LLM output validated against a schema or AST before use as code, query, command, or API payload; appropriate downstream injection check applied
- [ ] *(R3)* System prompt is not returned in any user-visible response path
- [ ] *(R1)* RAG retrieval is scoped to the requesting user's authorized data
- [ ] *(R1)* Token limits and agent step/iteration limits are set