# GCP Infrastructure as Code Security

## Risks

| # | Risk | Detection signal | Fortify Category | CWE | OWASP 2025 |
|---|------|-----------------|-----------------|-----|------------|
| R1 | GKE HTTP Basic Authentication enabled | `google_container_cluster` with `master_auth.username` set to a non-empty string, or `master_auth.password` set | GCP Terraform Misconfiguration: GKE Cluster HTTP Basic Authentication | CWE-287 | A07 Authentication Failures |
| R2 | GKE client certificate authentication enabled | `google_container_cluster` with `master_auth.client_certificate_config.issue_client_certificate = true` | GCP Terraform Misconfiguration: GKE Cluster Certificate-Based Authentication | CWE-287 | A07 Authentication Failures |
| R3 | GKE Legacy ABAC authorization enabled | `google_container_cluster` with `enable_legacy_abac = true` | GCP Terraform Misconfiguration: GKE Cluster Legacy Authorization | CWE-637 | A01 Broken Access Control |
| R4 | GKE control plane not restricted to authorized networks | `google_container_cluster` missing a `master_authorized_networks_config` block, or block present with `enabled = false` | GCP Terraform Misconfiguration: GKE Cluster Administrative Interface Access Control | CWE-749 | A01 Broken Access Control |
| R5 | GKE cluster control plane publicly accessible | `google_container_cluster` with `private_cluster_config` block absent, or `private_cluster_config.enable_private_nodes = false` | GCP Terraform Misconfiguration: GKE Cluster Publicly Accessible | CWE-749 | A01 Broken Access Control |
| R6 | Compute Engine instance uses default service account | `google_compute_instance` with no `service_account` block, or `service_account.email` set to the default pattern (`PROJECT_NUMBER-compute@developer.gserviceaccount.com`) | GCP Terraform Misconfiguration: Compute Engine Default Service Account | CWE-250 | A01 Broken Access Control |
| R7 | Compute Engine instance allows project-wide SSH keys | `google_compute_instance` where `metadata` block is absent, or present without `block-project-ssh-keys = "true"` | GCP Terraform Misconfiguration: Compute Engine Project-Wide SSH | CWE-250 | A01 Broken Access Control |
| R8 | Overly broad firewall rule | `google_compute_firewall` with `direction = "INGRESS"`, `source_ranges` containing `"0.0.0.0/0"` or `"::/0"`, and `allow` block with port ranges covering sensitive services (SSH `22`, RDP `3389`, database ports, or `all`) | GCP Terraform Misconfiguration: Improper Compute Engine Access Control | CWE-250 | A01 Broken Access Control |
| R9 | Cloud Storage bucket publicly accessible | `google_storage_bucket_iam_binding` or `google_storage_bucket_iam_member` with `members`/`member` containing `"allUsers"` or `"allAuthenticatedUsers"` | GCP Terraform Misconfiguration: Cloud Storage Bucket Publicly Accessible | CWE-284 | A01 Broken Access Control |
| R10 | Cloud KMS key publicly accessible | `google_kms_crypto_key_iam_binding` or `google_kms_crypto_key_iam_member` with `members`/`member` containing `"allUsers"` or `"allAuthenticatedUsers"` | GCP Terraform Misconfiguration: Cloud KMS CryptoKey Publicly Accessible | CWE-284 | A01 Broken Access Control |
| R11 | Cloud SQL in-transit encryption not required | `google_sql_database_instance` with `settings.ip_configuration.require_ssl = false` or `require_ssl` absent (defaults to false) | GCP Terraform Misconfiguration: Insecure Cloud SQL Transport | CWE-311 | A04 Cryptographic Failures |
| R12 | Overly permissive IAM role binding | `google_project_iam_binding` or `google_project_iam_member` with `role = "roles/owner"` or `role = "roles/editor"` at the project level; or any IAM binding with `member = "allUsers"` or `member = "allAuthenticatedUsers"` | GCP Terraform Misconfiguration: Overly Permissive IAM Role | CWE-285 | A01 Broken Access Control |
| R13 | BigQuery dataset publicly accessible | `google_bigquery_dataset_iam_binding` or `google_bigquery_dataset_iam_member` with `member` containing `"allUsers"` or `"allAuthenticatedUsers"`; `google_bigquery_dataset` with an `access` block where `special_group` is `"allUsers"` or `"allAuthenticatedUsers"` | GCP Terraform Misconfiguration: BigQuery Dataset Publicly Accessible | CWE-284 | A01 Broken Access Control |
| R14 | Artifact Registry repository publicly accessible | `google_artifact_registry_repository_iam_binding` or `google_artifact_registry_repository_iam_member` with `member` containing `"allUsers"` or `"allAuthenticatedUsers"` | GCP Terraform Misconfiguration: Artifact Registry Publicly Accessible | CWE-284 | A01 Broken Access Control |

## Required Agent Actions

1. **Disable GKE HTTP Basic Authentication** *(R1)* — set `master_auth.username = ""` and `master_auth.password = ""` in `google_container_cluster`. When username and password are set to non-empty values, the Kubernetes API server enables HTTP basic authentication, which transmits static credentials with every request. All cluster access should use RBAC with Google-managed credentials.

2. **Disable GKE client certificate authentication** *(R2)* — set `master_auth.client_certificate_config.issue_client_certificate = false` in `google_container_cluster`. Use Google-managed short-lived credentials (Workload Identity, IAM, or OIDC) instead.

3. **Disable GKE Legacy ABAC** *(R3)* — set `enable_legacy_abac = false`. Use Kubernetes RBAC (`ClusterRole`, `RoleBinding`) for fine-grained authorization instead of the Legacy Attribute-Based Access Control system.

4. **Restrict GKE control plane to authorized networks** *(R4)* — add a `master_authorized_networks_config` block with `enabled = true` and a specific `cidr_blocks` list containing only trusted networks.

5. **Enable private nodes on GKE clusters** *(R5)* — set `private_cluster_config.enable_private_nodes = true` so that node VMs receive only private RFC 1918 IP addresses.

6. **Avoid the default Compute Engine service account** *(R6)* — create a dedicated service account with only the IAM roles required for the workload. Set `service_account.email` to this account and restrict `service_account.scopes` to the minimum required (e.g., `["https://www.googleapis.com/auth/cloud-platform"]` combined with IAM roles, or specific API scopes).

7. **Block project-wide SSH keys on compute instances** *(R7)* — add `block-project-ssh-keys = "true"` to the `metadata` block on all `google_compute_instance` resources. This prevents SSH keys set at the project level from being pushed to the instance, ensuring only instance-level or OS Login keys are used.

8. **Restrict ingress firewall rules to specific sources and ports** *(R8)* — replace `source_ranges = ["0.0.0.0/0"]` with the specific IP ranges of authorized clients. Do not create `allow` rules for SSH (22), RDP (3389), database ports, or `all` protocols from the public internet. Use Identity-Aware Proxy (IAP) for administrative SSH/RDP access instead.

9. **Do not grant public access to Cloud Storage buckets** *(R9)* — never use `"allUsers"` or `"allAuthenticatedUsers"` as a member in `google_storage_bucket_iam_binding` or `google_storage_bucket_iam_member`. Restrict bucket access to specific service accounts or user groups.

10. **Do not grant public access to Cloud KMS keys** *(R10)* — never use `"allUsers"` or `"allAuthenticatedUsers"` as a member in `google_kms_crypto_key_iam_binding` or `google_kms_crypto_key_iam_member`.

11. **Require SSL on all Cloud SQL connections** *(R11)* — set `settings.ip_configuration.require_ssl = true` on all `google_sql_database_instance` resources. Configure client connections to use the Cloud SQL Auth Proxy or present a valid client certificate. Do not permit plaintext database connections.

12. **Apply least privilege to project-level IAM bindings** *(R12)* — do not assign `roles/owner` or `roles/editor` at the project level in Terraform unless the binding serves a short-lived bootstrap purpose. Use predefined roles scoped to specific services (e.g., `roles/storage.objectAdmin`) or create custom roles with only the required permissions. Never bind `allUsers` or `allAuthenticatedUsers` to project-level roles.

13. **Do not grant public access to BigQuery datasets** *(R13)* — never use `"allUsers"` or `"allAuthenticatedUsers"` as a member in `google_bigquery_dataset_iam_binding` or `google_bigquery_dataset_iam_member`, or as a `special_group` in a `google_bigquery_dataset` access block.

14. **Do not grant public access to Artifact Registry repositories** *(R14)* — never use `"allUsers"` or `"allAuthenticatedUsers"` as a member in `google_artifact_registry_repository_iam_binding` or `google_artifact_registry_repository_iam_member`.

## Completion Evidence

*(Verify for every resource type present in the change.)*

- [ ] R1: GKE `master_auth.username` and `master_auth.password` set to empty strings
- [ ] R2: `master_auth.client_certificate_config.issue_client_certificate = false` on all GKE clusters
- [ ] R3: `enable_legacy_abac = false` on all GKE clusters
- [ ] R4: `master_authorized_networks_config.enabled = true` with specific CIDR allowlist on all GKE clusters
- [ ] R5: `private_cluster_config.enable_private_nodes = true` on all GKE clusters
- [ ] R6: No `google_compute_instance` uses the default Compute Engine service account
- [ ] R7: `metadata.block-project-ssh-keys = "true"` on all `google_compute_instance` resources
- [ ] R8: No ingress firewall rule opens sensitive ports to `0.0.0.0/0` or `::/0`
- [ ] R9: No IAM binding on Cloud Storage buckets includes `allUsers` or `allAuthenticatedUsers`
- [ ] R10: No IAM binding on Cloud KMS keys includes `allUsers` or `allAuthenticatedUsers`
- [ ] R11: `settings.ip_configuration.require_ssl = true` on all Cloud SQL instances
- [ ] R12: No `roles/owner` or `roles/editor` binding at project level; no `allUsers`/`allAuthenticatedUsers` in project IAM
- [ ] R13: No BigQuery dataset IAM binding grants access to `allUsers` or `allAuthenticatedUsers`
- [ ] R14: No Artifact Registry repository IAM binding grants access to `allUsers` or `allAuthenticatedUsers`
