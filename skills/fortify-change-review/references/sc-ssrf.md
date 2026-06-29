# SSRF and Outbound Request Safety

## Risks

| # | Risk | Detection signal | Fortify Category | CWE | OWASP 2025 | Action |
|---|------|-----------------|-----------------|-----|------------|--------|
| R1 | Server-side request forgery (SSRF) | User-controlled string passed directly to `fetch()`, `HttpClient`, `requests.get()`, `urllib.request`, or equivalent without hostname/IP validation or scheme restriction | Server-Side Request Forgery | CWE-918 | — | Actions 1–5 |
| R2 | Open redirect | `?next=`, `?redirect=`, `?returnUrl=`, or equivalent parameter passed to redirect response without destination validation; absolute URL to external domain accepted without host allowlist check | Open Redirect | CWE-601 | A01 Broken Access Control | Action 6 |

## Scoped Exclusions

Do not report these patterns even if a detection signal above matches:

- *(R1)* **Path-only control** — if the attacker can control only the URL path (the segment after a fixed host and scheme), this is not an SSRF finding. SSRF requires the ability to reach an unintended destination, which requires control of the host or scheme.
- *(R2)* **Relative redirect destinations** — redirect destinations that are relative paths (start with `/`, contain no `://` and no hostname) are inherently safe and do not require host validation. Only flag when an absolute URL with a user-controlled or external host can be supplied.

## Required Agent Actions

1. **Validate the URL before making the request** *(R1)*:
   - Parse the URL and resolve the hostname to an IP address.
   - Reject requests to private IP ranges: `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`, `127.0.0.0/8`, `169.254.0.0/16` (link-local / metadata), `::1`, `fc00::/7`.
   - Reject non-HTTP/HTTPS schemes (`file://`, `gopher://`, `ftp://`, `dict://`, etc.).
   - Block cloud provider metadata endpoints by hostname: `metadata.google.internal`, `169.254.169.254`, `fd00:ec2::254`.

2. **Use an allowlist where possible** *(R1, R2)* — if the set of target hosts is known and finite, maintain an explicit allowlist. Reject any host not on the list. Prefer this over a blocklist.

3. **Prevent redirect-chain bypasses** *(R1)* — disable automatic redirect following, or re-validate the resolved destination URL after each redirect before following it.

4. **Set timeouts and response size limits** *(R1)* — enforce connection timeout (e.g., ≤5 seconds) and maximum response body size to prevent denial-of-service and data exfiltration via slow responses.

5. **Do not return raw response bodies to the requester** *(R1)* — if the response must be returned to the user (e.g., proxied content), ensure the content type is validated and the response does not expose internal system data.

6. **Validate open redirect destinations** *(R2)* — if the application redirects to a user-supplied URL (login `?next=`, post-auth `?returnUrl=`, etc.), validate the destination is a relative path or belongs to an allowlisted host. Reject absolute URLs pointing to external domains. Use a server-side allowlist, not a client-side check. Redirects to relative paths (starting with `/` and containing no protocol or host) are inherently safe and do not require further validation.

## Completion Evidence

- [ ] *(R1)* URL is parsed and resolved IP validated against private/reserved ranges before any outbound request
- [ ] *(R1)* Only `http://` and `https://` schemes permitted; cloud metadata endpoints explicitly blocked
- [ ] *(R1)* Redirect following disabled or each redirect target re-validated before following
- [ ] *(R2)* Open redirect destinations validated against a relative-path or host allowlist; absolute external URLs rejected