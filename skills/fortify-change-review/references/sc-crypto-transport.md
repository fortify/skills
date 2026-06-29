# Cryptography, Transport Security, and Key Management

## Risks

| # | Risk | Detection signal | Fortify Category | CWE | OWASP 2025 | Action |
|---|------|-----------------|-----------------|-----|------------|--------|
| R1 | Cleartext data transmission | `http://` scheme on sensitive endpoints; `ssl=false`, `tls=disabled`, or `requireSSL=false` in database, cache, broker, or service connection config | Insecure Transport | CWE-319 | A04 Cryptographic Failures | Action 1 |
| R2 | Weak TLS version | `SSLv3`, `TLSv1`, or `TLSv1.1` in protocol config; `MinProtocol` below TLS 1.2 | Insecure Transport: Weak SSL Protocol | CWE-326 | A04 Cryptographic Failures | Action 1 |
| R3 | Weak cipher suite | RC4, 3DES, NULL, EXPORT-grade, or bare CBC cipher in TLS suite definition | Insecure Transport: Weak SSL Cipher | CWE-327 | A04 Cryptographic Failures | Action 3 |
| R4 | Certificate validation disabled | `ssl_verify=False`, `InsecureSkipVerify: true`, `CERT_NONE`, `checkServerIdentity: () => undefined`; custom `TrustManager` or `HostnameVerifier` that unconditionally returns true | Insecure SSL: Server Identity Verification Disabled | CWE-297 | A04 Cryptographic Failures | Action 2 |
| R5 | Hardcoded encryption key or IV | Encryption key, HMAC secret, or IV assigned from a string or byte array literal in source; same IV constant reused across operations | Key Management: Hardcoded Encryption Key | CWE-321 | A04 Cryptographic Failures | Action 5 |
| R6 | Insecure encryption mode of operation | `AES/ECB` in cipher transformation; `AES/CBC` without explicit MAC; 3DES in any mode | Weak Encryption: Insecure Mode of Operation | CWE-327 | A04 Cryptographic Failures | Action 4 |
| R7 | Insufficient encryption key size | RSA or DSA key below 2048 bits; ECC key below 256 bits; AES key below 128 bits; DES (56-bit effective key) | Weak Encryption: Insufficient Key Size | CWE-326 | A04 Cryptographic Failures | Action 7 |
| R8 | Weak password hashing | `MD5` or `SHA-1` used for password storage; password hashed without a KDF or unique random salt; general-purpose hash used directly for passwords | Weak Cryptographic Hash | CWE-327 | A04 Cryptographic Failures | Action 6 |
| R9 | Inadequate RSA padding | RSA encryption with `PKCS1Padding` or raw (no-padding) scheme; `pkcs1_v15` in Python; `RSA_PKCS1_PADDING` in OpenSSL | Weak Encryption: Inadequate RSA Padding | CWE-780 | A04 Cryptographic Failures | Action 9 |
| R10 | Cleartext storage of sensitive data | Passwords, tokens, or PII written to database columns, files, or object storage without encryption; encryption-at-rest disabled in cloud storage config | Insecure Storage: Lacking Data Protection | CWE-312 | A04 Cryptographic Failures | Action 8 |

## Required Agent Actions

1. **Enforce TLS 1.2 or higher on all connections** *(R1, R2)* — disable SSLv3, TLS 1.0, and TLS 1.1 explicitly. Prefer TLS 1.3. Apply to database, cache, broker, and internal service connections, not just public-facing endpoints. Do not allow protocol negotiation below TLS 1.2.

2. **Do not disable certificate validation** *(R4)* — `ssl_verify=False`, `InsecureSkipVerify: true`, `checkServerIdentity: () => undefined`, and equivalent settings must never appear in production code paths. If needed for testing, guard with an environment flag and fail closed.

3. **Use only strong cipher suites** *(R3)* — prefer ECDHE for key exchange (forward secrecy), AES-GCM or ChaCha20-Poly1305 for encryption. Reject RC4, 3DES, NULL, EXPORT-grade, and CBC-without-MAC cipher suites.

4. **Use authenticated encryption** *(R6)* — for symmetric encryption, use AES-GCM, AES-CCM, or ChaCha20-Poly1305. Do not use AES-ECB (leaks patterns), AES-CBC without a MAC (padding oracle risk), or 3DES.

5. **Never hardcode keys or IVs** *(R5)* — encryption keys, HMACs, salts, and IVs must not be literal string or byte constants in source. Source them from a key management service or environment variable. IVs must be randomly generated per operation, never reused.

6. **Hash passwords with an appropriate KDF** *(R8)* — use bcrypt, Argon2id, or PBKDF2 with a sufficient iteration count and a unique random salt per password. Do not use MD5, SHA-1, SHA-256, or any general-purpose hash directly for password storage.

7. **Use adequate key sizes** *(R7)* — RSA/DSA minimum 2048 bits; ECC minimum 256 bits; AES minimum 128 bits (prefer 256). Do not use DES (56-bit effective key).

8. **Encrypt sensitive data at rest** *(R10)* — sensitive fields (passwords, tokens, PII) stored in database columns, files, or object storage must be encrypted. Enable encryption-at-rest in cloud storage services (S3 SSE, Azure Storage encryption, GCS CMEK). Do not store plaintext secrets or keys in the same location as the data they protect.

9. **Use OAEP padding for RSA encryption** *(R9)* — use `RSA/ECB/OAEPWithSHA-256AndMGF1Padding` in Java, the `OAEP` scheme in Python, or the platform equivalent. Never use PKCS#1 v1.5 padding or raw (no-padding) RSA for encryption operations.

## Completion Evidence

- [ ] *(R1, R2)* TLS 1.2 minimum enforced on all connections (public endpoints, database, cache, broker, internal services); SSLv3, TLS 1.0, and TLS 1.1 disabled
- [ ] *(R3)* Only strong cipher suites in use; no RC4, 3DES, NULL, EXPORT, or bare CBC
- [ ] *(R4)* Certificate validation enabled on all connections; no `InsecureSkipVerify` / `CERT_NONE` equivalents in production paths
- [ ] *(R5)* No hardcoded keys, IVs, or salts in source; IVs randomly generated per operation
- [ ] *(R6)* Authenticated encryption mode used (GCM, CCM, or Poly1305); no AES-ECB, AES-CBC without MAC, or 3DES
- [ ] *(R7)* RSA/DSA ≥ 2048 bits; ECC ≥ 256 bits; AES ≥ 128 bits; no DES
- [ ] *(R8)* Password storage uses bcrypt, Argon2id, or PBKDF2 with a unique random salt
- [ ] *(R9)* RSA encryption uses OAEP padding; no PKCS#1 v1.5 or raw-padding RSA in production code
- [ ] *(R10)* Sensitive data at rest encrypted; encryption-at-rest enabled in cloud storage configuration
