# XSS and Unsafe Output Rendering

## Risks

| # | Risk | Detection signal | Fortify Category | CWE | OWASP 2025 | Action |
|---|------|-----------------|-----------------|-----|------------|--------|
| R1 | Reflected XSS | User input from query parameters, form fields, or HTTP headers directly concatenated into the HTTP response without encoding; `@Html.Raw()`, `| safe`, or unescaped template directives applied to request-derived data | Cross-Site Scripting: Reflected | CWE-79 | A05 Injection | Actions 1, 4 |
| R2 | Persistent (stored) XSS | User-controlled data previously stored (database, file, or cache) rendered in an HTML response without context-appropriate encoding; `@Html.Raw()`, `| safe`, or unescaped template directives on stored user content | Cross-Site Scripting: Persistent | CWE-79 | A05 Injection | Actions 1, 4 |
| R3 | DOM-based XSS | User-controlled value assigned to `innerHTML`, `outerHTML`, `document.write()`, jQuery `.html()`, or `dangerouslySetInnerHTML` without sanitization | Cross-Site Scripting: DOM | CWE-79 | A05 Injection | Actions 2, 4 |
| R4 | Insecure sanitizer policy | HTML/Markdown sanitizer configured without an explicit safe-tag allowlist; denylist-based sanitization; `javascript:` or `data:` URI scheme not blocked in `href`/`src` attributes | Insecure Sanitizer Policy | CWE-79, CWE-80 | A05 Injection | Action 3 |

## Scoped Exclusions

Do not report these patterns even if a detection signal above matches:

- *(R1, R2)* **React, Angular, and Vue default interpolation** — standard template interpolation (`{}` in JSX, `{{ }}` in Angular/Vue) escapes HTML automatically and is safe by default. Do not flag these. Only flag explicit unsafe overrides: `dangerouslySetInnerHTML`, `bypassSecurityTrustHtml`, `bypassSecurityTrustScript`, `[innerHTML]` binding, or `v-html` with user-controlled data.
- *(R3)* **`textContent` and `innerText` assignments** — these properties insert text only and cannot execute HTML or scripts. Do not flag assignments to `textContent` or `innerText` regardless of the input source.

## Required Agent Actions

1. **Apply context-aware output encoding** *(R1, R2)* — encode output based on the context where it appears: HTML body, HTML attribute, JavaScript, CSS, or URL. Use framework-provided encoding (not manual escaping).
2. **Use safe rendering APIs by default** *(R3)* — prefer `textContent`, `innerText`, `{{}}` (escaped interpolation) over `innerHTML`, `{{{}}}`(unescaped), or `dangerouslySetInnerHTML`.
3. **Sanitize rich content with an allowlist** *(R4)* — if HTML/Markdown must be rendered, use a vetted sanitization library (DOMPurify, Bleach, OWASP Java HTML Sanitizer) with an allowlist of safe tags and attributes. Never use a denylist.
4. **Validate URL schemes** *(R1, R2, R3)* — for dynamic `href` or `src`, allowlist schemes (`https:`, `mailto:`) and reject `javascript:`, `data:text/html`, `vbscript:`.

## Completion Evidence

- [ ] *(R1)* All user-controlled request input reflected in server responses uses context-appropriate encoding; no unescaped directives on request-derived data
- [ ] *(R2)* All stored user-controlled data rendered in HTML responses uses context-appropriate encoding; no unescaped directives on stored user content
- [ ] *(R3)* No `innerHTML`, `dangerouslySetInnerHTML`, or equivalent used with unsanitized user input
- [ ] *(R4)* Rich content rendering uses an allowlist-based sanitizer; `javascript:` and `data:` URI schemes rejected
- [ ] *(R1, R2, R3)* Dynamic URLs validate the scheme against an allowlist