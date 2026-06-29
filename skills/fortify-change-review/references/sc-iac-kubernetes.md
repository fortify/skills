# Kubernetes Infrastructure as Code Security

## Risks

| # | Risk | Detection signal | Fortify Category | CWE | OWASP 2025 | Action |
|---|------|-----------------|-----------------|-----|------------|--------|
| R1 | Privileged container | `securityContext.privileged: true` on a container or init container in any Pod spec | Kubernetes Misconfiguration: Privileged Container | CWE-250 | A01 Broken Access Control | Action 1 |
| R2 | Writable host path volume mount | `volumes[].hostPath` present in a Pod spec, and the corresponding `volumeMounts[].readOnly` is absent or `false` | Kubernetes Misconfiguration: Host Write Access | CWE-284 | A01 Broken Access Control | Action 2 |
| R3 | kube-apiserver etcd transport not secured | Static Pod manifest for `kube-apiserver` (command args) missing `--etcd-cafile`, `--etcd-certfile`, or `--etcd-keyfile`; or any of these set to an empty value | Kubernetes Misconfiguration: Insecure etcd Client Transport | CWE-311 | A04 Cryptographic Failures | Action 3 |
| R4 | kube-apiserver TLS not configured | Static Pod manifest for `kube-apiserver` missing `--tls-cert-file` or `--tls-private-key-file` in command args | Kubernetes Misconfiguration: Missing API Server Identity Verification | CWE-297 | A07 Authentication Failures | Action 4 |
| R5 | Kubelet anonymous authentication enabled | Kubelet `ConfigMap` or config file with `authentication.anonymous.enabled: true`; or `--anonymous-auth=true` in kubelet command args | Kubernetes Misconfiguration: Missing Kubelet Identity Verification | CWE-297 | A07 Authentication Failures | Action 5 |
| R6 | Kubelet authorization set to AlwaysAllow | Kubelet `ConfigMap` or config file with `authorization.mode: AlwaysAllow`; or `--authorization-mode=AlwaysAllow` in kubelet command args | Kubernetes Misconfiguration: Missing Kubelet Identity Verification | CWE-297 | A01 Broken Access Control | Action 5 |
| R7 | Kubelet client CA not configured | Kubelet `ConfigMap` or config file missing `authentication.x509.clientCAFile`; or `--client-ca-file` absent from kubelet command args | Kubernetes Misconfiguration: Missing Kubelet Certificate Authentication | CWE-285 | A01 Broken Access Control | Action 5 |

## Required Agent Actions

1. **Remove privileged mode from all containers** *(R1)* — set `securityContext.privileged: false` or remove the field (defaults to false). Privileged containers have full access to the host kernel and all devices. If a container needs elevated Linux capabilities, use `securityContext.capabilities.add` to grant only the specific capability required (e.g., `NET_ADMIN`, `SYS_PTRACE`). Add `securityContext.allowPrivilegeEscalation: false` as a defense-in-depth measure.

2. **Mount host paths read-only when required** *(R2)* — if a `hostPath` volume is necessary (e.g., to read `/var/log` or `/proc`), set `volumeMounts[].readOnly: true` on the corresponding mount. Prefer `emptyDir`, `configMap`, or `secret` volume types over `hostPath`. Never mount sensitive host directories (`/`, `/etc`, `/var/run/docker.sock`, `/proc`) with write access.

3. **Secure kube-apiserver to etcd communication with mutual TLS** *(R3)* — ensure the `kube-apiserver` static Pod manifest includes:
   - `--etcd-cafile` — path to the etcd CA certificate
   - `--etcd-certfile` — path to the client certificate used by the API server
   - `--etcd-keyfile` — path to the private key for that certificate

   All values must be non-empty paths to valid files. Without these, communication between the API server and etcd is unencrypted.

4. **Configure TLS on the API server endpoint** *(R4)* — ensure the `kube-apiserver` static Pod manifest includes both `--tls-cert-file` and `--tls-private-key-file` pointing to a valid certificate and key pair. This is required for encrypted, authenticated communication between clients (kubectl, controllers) and the API server.

5. **Harden kubelet authentication and authorization** *(R5, R6, R7)* — in the kubelet configuration:
   - Set `authentication.anonymous.enabled: false` to prevent unauthenticated requests
   - Set `authorization.mode: Webhook` (not `AlwaysAllow`) to enforce RBAC-based access control
   - Set `authentication.x509.clientCAFile` to the path of the cluster CA certificate to enable certificate-based client authentication

   These three settings together prevent the kubelet API from accepting anonymous or unrestricted requests.

## Completion Evidence

- [ ] *(R1)* No container or init container has `securityContext.privileged: true`; elevated capabilities limited to specific named capabilities only
- [ ] *(R2)* All `hostPath` volume mounts have `readOnly: true`; sensitive host directories are not mounted
- [ ] *(R3)* `kube-apiserver` manifest includes `--etcd-cafile`, `--etcd-certfile`, and `--etcd-keyfile` with non-empty paths
- [ ] *(R4)* `kube-apiserver` manifest includes `--tls-cert-file` and `--tls-private-key-file` with non-empty paths
- [ ] *(R5)* Kubelet `authentication.anonymous.enabled: false`; no `--anonymous-auth=true` in kubelet args
- [ ] *(R6)* Kubelet `authorization.mode: Webhook`; no `--authorization-mode=AlwaysAllow` in kubelet args
- [ ] *(R7)* Kubelet `authentication.x509.clientCAFile` set; `--client-ca-file` present in kubelet args
