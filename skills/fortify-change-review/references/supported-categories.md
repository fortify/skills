# Supported Fortify Vulnerability Categories

This table lists all Fortify vulnerability categories covered by the fortify-change-review skill. Categories are sourced from the risk tables in each `sc-*.md` file.  Category coverage represents subset of the comprehensive Fortify SAST & DAST capabilities, both within these categories and across hundreds more.  The intent of the skill is to 'shifting left' and enable AI codign agents to catch a subset of high impact, common and low noise vulnerabilities as code is being generated.

| Fortify Category | CWE | OWASP 2025 | Security Check |
|---|---|---|---|
| **Code / Application Checks** | | | |
| Access Control | CWE-639 | A01 | sc-authorization.md |
| Access Control: Missing Authorization Check | CWE-862 | A01 | sc-authorization.md |
| Mass Assignment: Request Parameters Bound into Persisted Objects | CWE-915 | A01 | sc-authorization.md |
| Expression Language Injection | CWE-917, CWE-95 | A05 | sc-injection.md |
| LDAP Injection | CWE-90 | A05 | sc-injection.md |
| NoSQL Injection | CWE-943 | A05 | sc-injection.md |
| SQL Injection | CWE-89 | A05 | sc-injection.md |
| Command Injection | CWE-78 | A05 | sc-deserialization.md |
| Dynamic Code Evaluation: Unsafe Deserialization | CWE-502 | A08 | sc-deserialization.md |
| Dynamic Code Evaluation: Code Injection | CWE-94 | A05 | sc-deserialization.md |
| Cross-Site Scripting: Reflected | CWE-79 | A05 | sc-xss-output.md |
| Cross-Site Scripting: Persistent | CWE-79 | A05 | sc-xss-output.md |
| Cross-Site Scripting: DOM | CWE-79 | A05 | sc-xss-output.md |
| Insecure Sanitizer Policy | CWE-79, CWE-80 | A05 | sc-xss-output.md |
| Dangerous File Inclusion | CWE-98 | A01 | sc-file-path.md |
| Directory Traversal | CWE-22 | A01 | sc-file-path.md |
| Open Redirect | CWE-601 | A01 | sc-ssrf.md |
| Server-Side Request Forgery | CWE-918 | — | sc-ssrf.md |
| Access Control: Form Authentication Bypass | CWE-287 | A07 | sc-authentication.md |
| Access Control: Missing Authentication | CWE-306 | A07 | sc-authentication.md |
| Password Management: Weak Password Policy | CWE-640 | A07 | sc-authentication.md |
| Log Forging | CWE-117 | A09 | sc-logging-audit.md |
| Privacy Violation | CWE-532 | A09 | sc-logging-audit.md |
| System Information Leak: External | CWE-209 | A09 | sc-logging-audit.md |
| Excessive Agency | CWE-285 | A01 | sc-ai-agent-safety.md |
| Insecure Tool Calling | CWE-285 | A01 | sc-ai-agent-safety.md |
| Prompt Injection | CWE-77 | A01 | sc-ai-agent-safety.md |
| Insecure SSL: Server Identity Verification Disabled | CWE-297 | A04 | sc-crypto-transport.md |
| Insecure Storage: Lacking Data Protection | CWE-312 | A04 | sc-crypto-transport.md |
| Insecure Transport | CWE-319 | A04 | sc-crypto-transport.md |
| Insecure Transport: Weak SSL Cipher | CWE-327 | A04 | sc-crypto-transport.md |
| Insecure Transport: Weak SSL Protocol | CWE-326 | A04 | sc-crypto-transport.md |
| Key Management: Hardcoded Encryption Key | CWE-321 | A04 | sc-crypto-transport.md |
| Weak Cryptographic Hash | CWE-327 | A04 | sc-crypto-transport.md |
| Weak Encryption: Inadequate RSA Padding | CWE-780 | A04 | sc-crypto-transport.md |
| Weak Encryption: Insecure Mode of Operation | CWE-327 | A04 | sc-crypto-transport.md |
| Weak Encryption: Insufficient Key Size | CWE-326 | A04 | sc-crypto-transport.md |
| XML External Entity Injection | CWE-611 | A02 | sc-xxe.md |
| XPath Injection | CWE-643 | A05 | sc-xxe.md |
| **AWS IaC Checks** | | | |
| AWS CloudFormation Misconfiguration: Insecure DocumentDB Transport | CWE-297 | A07 | sc-iac-aws.md |
| AWS CloudFormation Misconfiguration: Insecure ElastiCache Transport | CWE-311 | A02 | sc-iac-aws.md |
| AWS CloudFormation Misconfiguration: Privileged Batch Container | CWE-250 | A01 | sc-iac-aws.md |
| AWS CloudFormation Misconfiguration: Weak Secrets Manager Generated Password | CWE-521 | A07 | sc-iac-aws.md |
| AWS Terraform Misconfiguration: Insecure RDS Proxy Transport | CWE-311 | A04 | sc-iac-aws.md |
| AWS Terraform Misconfiguration: MQ Publicly Accessible | CWE-749 | A01 | sc-iac-aws.md |
| AWS Terraform Misconfiguration: Neptune Publicly Accessible | CWE-749 | A01 | sc-iac-aws.md |
| AWS Terraform Misconfiguration: Redshift Publicly Accessible | CWE-749 | A01 | sc-iac-aws.md |
| AWS CloudFormation Misconfiguration: Insecure API Gateway Transport | CWE-319 | A04 | sc-iac-aws.md |
| AWS CloudFormation Misconfiguration: Insecure CloudFront Transport | CWE-319 | A04 | sc-iac-aws.md |
| AWS CloudFormation Misconfiguration: Insecure ELB Transport | CWE-319 | A04 | sc-iac-aws.md |
| AWS CloudFormation Misconfiguration: Insecure OpenSearch Transport | CWE-319 | A04 | sc-iac-aws.md |
| AWS CloudFormation Misconfiguration: Insecure RDS Transport | CWE-319 | A04 | sc-iac-aws.md |
| AWS CloudFormation Misconfiguration: Insecure S3 Bucket Transport | CWE-319 | A04 | sc-iac-aws.md |
| **Azure IaC Checks** | | | |
| Azure ARM Misconfiguration: HTTPS Not Required | CWE-319 | A04 | sc-iac-azure.md |
| Azure ARM Misconfiguration: Improper AKS Access Control | CWE-287 | A01 | sc-iac-azure.md |
| Azure ARM Misconfiguration: Improper Custom Role Access Control Policy | CWE-250 | A01 | sc-iac-azure.md |
| Azure ARM Misconfiguration: Insecure Application Gateway Transport | CWE-319 | A04 | sc-iac-azure.md |
| Azure ARM Misconfiguration: Insecure MySQL Server Transport | CWE-297 | A07 | sc-iac-azure.md |
| Azure ARM Misconfiguration: Insecure PostgreSQL Server Transport | CWE-297 | A07 | sc-iac-azure.md |
| Azure ARM Misconfiguration: Insecure Service Bus Transport | CWE-319 | A04 | sc-iac-azure.md |
| Azure ARM Misconfiguration: Insecure Storage Account Transport | CWE-311 | A02 | sc-iac-azure.md |
| Azure Terraform Misconfiguration: Insecure Function App Transport | CWE-319 | A04 | sc-iac-azure.md |
| Azure Terraform Misconfiguration: Insecure Redis Cache Transport | CWE-319 | A04 | sc-iac-azure.md |
| Azure Terraform Misconfiguration: Insecure SQL Database Transport | CWE-319 | A04 | sc-iac-azure.md |
| **GCP IaC Checks** | | | |
| GCP Terraform Misconfiguration: Artifact Registry Publicly Accessible | CWE-284 | A01 | sc-iac-gcp.md |
| GCP Terraform Misconfiguration: BigQuery Dataset Publicly Accessible | CWE-284 | A01 | sc-iac-gcp.md |
| GCP Terraform Misconfiguration: Compute Engine Default Service Account | CWE-250 | A01 | sc-iac-gcp.md |
| GCP Terraform Misconfiguration: Compute Engine Project-Wide SSH | CWE-250 | A01 | sc-iac-gcp.md |
| GCP Terraform Misconfiguration: Cloud KMS CryptoKey Publicly Accessible | CWE-284 | A01 | sc-iac-gcp.md |
| GCP Terraform Misconfiguration: Cloud Storage Bucket Publicly Accessible | CWE-284 | A01 | sc-iac-gcp.md |
| GCP Terraform Misconfiguration: GKE Cluster Administrative Interface Access Control | CWE-749 | A01 | sc-iac-gcp.md |
| GCP Terraform Misconfiguration: GKE Cluster Certificate-Based Authentication | CWE-287 | A07 | sc-iac-gcp.md |
| GCP Terraform Misconfiguration: GKE Cluster HTTP Basic Authentication | CWE-287 | A07 | sc-iac-gcp.md |
| GCP Terraform Misconfiguration: GKE Cluster Legacy Authorization | CWE-637 | A01 | sc-iac-gcp.md |
| GCP Terraform Misconfiguration: GKE Cluster Publicly Accessible | CWE-749 | A01 | sc-iac-gcp.md |
| GCP Terraform Misconfiguration: Improper Compute Engine Access Control | CWE-250 | A01 | sc-iac-gcp.md |
| GCP Terraform Misconfiguration: Insecure Cloud SQL Transport | CWE-311 | A04 | sc-iac-gcp.md |
| GCP Terraform Misconfiguration: Overly Permissive IAM Role | CWE-285 | A01 | sc-iac-gcp.md |
| **Kubernetes IaC Checks** | | | |
| Kubernetes Misconfiguration: Host Write Access | CWE-284 | A01 | sc-iac-kubernetes.md |
| Kubernetes Misconfiguration: Insecure etcd Client Transport | CWE-311 | A04 | sc-iac-kubernetes.md |
| Kubernetes Misconfiguration: Missing API Server Identity Verification | CWE-297 | A07 | sc-iac-kubernetes.md |
| Kubernetes Misconfiguration: Missing Kubelet Certificate Authentication | CWE-285 | A01 | sc-iac-kubernetes.md |
| Kubernetes Misconfiguration: Missing Kubelet Identity Verification | CWE-297 | A07, A01 | sc-iac-kubernetes.md |
| Kubernetes Misconfiguration: Privileged Container | CWE-250 | A01 | sc-iac-kubernetes.md |
