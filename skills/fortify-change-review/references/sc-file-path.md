# File Upload, Download, and Path Traversal

## Risks

| # | Risk | Detection signal | Fortify Category | CWE | OWASP 2025 | Action |
|---|------|-----------------|-----------------|-----|------------|--------|
| R1 | Path traversal (including zip slip) | `../` or `..\ ` in path parameters; `os.path.join(base, userInput)` without subsequent canonicalization and prefix check; archive entry paths not validated against the output directory before extraction | Directory Traversal | CWE-22 | A01 Broken Access Control | Actions 1, 4, 6 |
| R2 | User-controlled path used in file operation | Uploaded filename used directly in `open()`, `save()`, `sendFile()`, or storage path construction; file type validated by extension or `Content-Type` header only; uploaded files served directly from the web root | Dangerous File Inclusion | CWE-98 | A01 Broken Access Control | Actions 2, 3, 5 |

## Required Agent Actions

1. **Canonicalize and validate paths** *(R1)* — resolve the full canonical path (e.g., `Path.resolve()`, `os.path.realpath()`, `Paths.get().normalize().toAbsolutePath()`) and verify it starts with the intended base directory. Reject any path that escapes the base.

2. **Never use raw user-supplied filenames for storage or file operations** *(R2)* — generate a safe filename (UUID or hash-based) for storage. If the original name must be preserved, store it as metadata only, not as the filesystem path.

3. **Validate file type by content, not extension or Content-Type header** *(R2)* — check the file's magic bytes. Allowlist permitted types; reject everything else. Do not trust the `Content-Type` header or extension alone. For endpoints accessible to untrusted or unauthenticated users, or where uploaded files are served publicly, magic-byte validation is required. For endpoints restricted to authenticated internal users with strong access controls, an extension and MIME-type allowlist may be acceptable if the attack surface is clearly bounded.

4. **Secure archive extraction** *(R1)* — before extracting each entry, resolve its target path and verify it falls within the intended output directory. Reject entries containing `../` or absolute paths.

5. **Store uploads outside the web root** *(R2)* — uploaded files must not be directly executable or served without access control. Use a storage directory that is not publicly accessible, or serve files through a controller that applies authorization.

## Completion Evidence

- [ ] *(R1)* Paths canonicalized and confirmed within the intended base directory before any file operation
- [ ] *(R1)* Archive extraction validates each entry path against the target directory; entries with `../` or absolute paths rejected
- [ ] *(R2)* User-supplied filenames not used directly in filesystem paths; server-generated names used for storage
- [ ] *(R2)* File type validated by content (magic bytes); extension and `Content-Type` header not trusted as sole validation
- [ ] *(R2)* Uploaded files stored outside the web root or served through an authorized controller