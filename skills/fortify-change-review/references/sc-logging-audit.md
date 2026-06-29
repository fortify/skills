# Security Logging, Auditing, and Error Disclosure

## Risks

| # | Risk | Detection signal | Fortify Category | CWE | OWASP 2025 | Action |
|---|------|-----------------|-----------------|-----|------------|--------|
| R1 | Sensitive information in error responses | Stack trace, exception class name, internal file path, database connection string, or server config returned in HTTP error response body | System Information Leak: External | CWE-209 | A09 Logging and Alerting Failures | Action 3 |
| R2 | Sensitive values written to log files | Password, token, session ID, API key, or PII field written to log output; `Authorization`, `Cookie`, or `X-Api-Key` headers logged without redaction | Privacy Violation | CWE-532 | A09 Logging and Alerting Failures | Action 1 |
| R3 | Log injection / log forging | User-controlled string written to log output without newline sanitization; newline or Unicode line terminator characters reachable in log message | Log Forging | CWE-117 | A09 Logging and Alerting Failures | Action 2 |

## Scoped Exclusions

Do not report these patterns even if a detection signal above matches:

- *(R2)* **Non-sensitive data in logs** — logging non-PII, non-credential data is not a privacy violation. URLs, request paths, HTTP methods, status codes, and timestamps are safe to log even if user-influenced. Only flag when the logged value is a credential (password, token, API key, session ID) or PII (name, email, SSN, payment data).
- *(R3)* **Structured JSON loggers with typed value fields** — when a structured logging framework (e.g., Logrus, Winston, structlog, Log4j with JSON layout) is in use and user input flows only into a typed field value, log injection via newline characters is not a concern. Only flag when user input is concatenated into an unstructured plaintext message string, or flows into the message key or log level field.

## Required Agent Actions

1. **Exclude sensitive values from logs** *(R2)* — never log passwords, secrets, tokens, session IDs, credit card numbers, SSNs, or other PII. If logging a request or response, redact `Authorization`, `Cookie`, `X-Api-Key` headers and sensitive body fields before writing.

2. **Sanitize log inputs** *(R3)* — before writing user-controlled strings to log output, strip or encode newline characters (`\n`, `\r`, `\u2028`, `\u2029`) to prevent log injection/forging. If the application uses a structured (JSON) logging framework, user input assigned to a typed field value is inherently protected from newline-based log forging. Manual sanitization is most critical when user input is concatenated into unstructured (plaintext) log message strings; for structured loggers, verify that user input does not flow into the message key or log level fields.

3. **Return generic error messages to users** *(R1)* — error responses to end users must not contain stack traces, exception class names, internal file paths, database connection details, or server configuration. Log the full detail server-side; return a generic message and a correlation ID client-side.

## Completion Evidence

- [ ] *(R2)* No passwords, secrets, tokens, or PII written to log output; `Authorization`, `Cookie`, and `X-Api-Key` headers redacted before logging
- [ ] *(R3)* User-controlled strings sanitized (newlines stripped) before being written to log output
- [ ] *(R1)* User-facing error responses are generic; no stack traces, file paths, or connection details returned
