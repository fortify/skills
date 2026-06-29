# AWS Infrastructure as Code Security

## Risks

| # | Risk | Detection signal | Fortify Category | CWE | OWASP 2025 |
|---|------|-----------------|-----------------|-----|------------|
| R1 | Privileged Batch container | `AWS::Batch::JobDefinition` container properties with `Privileged: true`; Terraform `aws_batch_job_definition` with `"privileged": true` in `container_properties` JSON | AWS CloudFormation Misconfiguration: Privileged Batch Container | CWE-250 | A01 Broken Access Control |
| R2 | ElastiCache in-transit encryption disabled | `AWS::ElastiCache::ReplicationGroup` with `TransitEncryptionEnabled: false` or property absent (defaults to disabled); Terraform `aws_elasticache_replication_group` with `transit_encryption_enabled = false` or attribute absent | AWS CloudFormation Misconfiguration: Insecure ElastiCache Transport | CWE-311 | A02 Security Misconfiguration |
| R3 | DocumentDB TLS disabled | `AWS::DocDB::DBClusterParameterGroup` containing a parameter named `tls` with value `disabled`; connection string or cluster config permitting unencrypted connections | AWS CloudFormation Misconfiguration: Insecure DocumentDB Transport | CWE-297 | A07 Authentication Failures |
| R4 | Weak Secrets Manager generated password | `AWS::SecretsManager::Secret` with `GenerateSecretString.ExcludePunctuation: true` and no compensating character classes; `PasswordLength` below 20; or `ExcludeCharacters` string that eliminates numbers, uppercase, or lowercase | AWS CloudFormation Misconfiguration: Weak Secrets Manager Generated Password | CWE-521 | A07 Authentication Failures |
| R5 | Managed database or broker publicly accessible | Terraform `aws_db_instance`, `aws_redshift_cluster`, `aws_neptune_cluster_instance`, or `aws_mq_broker` with `publicly_accessible = true` | AWS Terraform Misconfiguration: *Publicly Accessible | CWE-749 | A01 Broken Access Control |
| R6 | RDS Proxy TLS not required | Terraform `aws_db_proxy` resource with `require_tls = false` or `require_tls` attribute absent (defaults to `false`) | AWS Terraform Misconfiguration: Insecure RDS Proxy Transport | CWE-311 | A04 Cryptographic Failures |
| R7 | S3 bucket allows unencrypted transport | `AWS::S3::BucketPolicy` without a `Deny` statement that conditions on `aws:SecureTransport: false`; `AWS::S3::Bucket` with no associated bucket policy enforcing HTTPS; Terraform `aws_s3_bucket_policy` whose policy document lacks a `Deny` effect with condition `aws:SecureTransport = false` | AWS CloudFormation Misconfiguration: Insecure S3 Bucket Transport | CWE-319 | A04 Cryptographic Failures |
| R8 | Load balancer accepts plaintext HTTP | `AWS::ElasticLoadBalancingV2::Listener` with `Protocol: HTTP` and no `redirect` action to HTTPS; `AWS::ElasticLoadBalancing::LoadBalancer` with a listener using `Protocol: HTTP`; Terraform `aws_lb_listener` with `protocol = "HTTP"` and no `redirect` default action | AWS CloudFormation Misconfiguration: Insecure ELB Transport | CWE-319 | A04 Cryptographic Failures |
| R9 | RDS instance does not enforce SSL connections | `AWS::RDS::DBParameterGroup` without `rds.force_ssl = 1` (PostgreSQL) or `require_secure_transport = ON` (MySQL); Terraform `aws_db_parameter_group` missing the SSL enforcement parameter for the engine type; `aws_rds_cluster_parameter_group` without SSL enforcement | AWS CloudFormation Misconfiguration: Insecure RDS Transport | CWE-319 | A04 Cryptographic Failures |
| R10 | CloudFront distribution permits HTTP viewer requests | `AWS::CloudFront::Distribution` with `DefaultCacheBehavior.ViewerProtocolPolicy: allow-all` or any cache behavior using `ViewerProtocolPolicy: allow-all`; Terraform `aws_cloudfront_distribution` with `default_cache_behavior.viewer_protocol_policy = "allow-all"` or any `ordered_cache_behavior.viewer_protocol_policy = "allow-all"` | AWS CloudFormation Misconfiguration: Insecure CloudFront Transport | CWE-319 | A04 Cryptographic Failures |
| R11 | API Gateway custom domain uses weak TLS policy | `AWS::ApiGateway::DomainName` with `SecurityPolicy: TLS_1_0`; `AWS::ApiGatewayV2::DomainName` with `DomainNameConfigurations[].SecurityPolicy: TLS_1_0`; Terraform `aws_api_gateway_domain_name` or `aws_apigatewayv2_domain_name` with `security_policy = "TLS_1_0"` | AWS CloudFormation Misconfiguration: Insecure API Gateway Transport | CWE-319 | A04 Cryptographic Failures |
| R12 | OpenSearch/Elasticsearch domain does not enforce HTTPS | `AWS::OpenSearchService::Domain` or `AWS::Elasticsearch::Domain` with `DomainEndpointOptions.EnforceHTTPS: false` or property absent (defaults to false); Terraform `aws_opensearch_domain` or `aws_elasticsearch_domain` with `domain_endpoint_options.enforce_https = false` or the block absent | AWS CloudFormation Misconfiguration: Insecure OpenSearch Transport | CWE-319 | A04 Cryptographic Failures |

## Required Agent Actions

1. **Never run Batch jobs in privileged mode** *(R1)* — set `Privileged: false` in `AWS::Batch::JobDefinition` container properties. If the workload requires elevated Linux capabilities, use `LinuxParameters.Capabilities.Add` to grant only the specific capability needed (e.g., `NET_ADMIN`) rather than full privilege.

2. **Enforce in-transit encryption on ElastiCache** *(R2)* — CFN `AWS::ElastiCache::ReplicationGroup`: `TransitEncryptionEnabled: true`; TF `aws_elasticache_replication_group`: `transit_encryption_enabled = true`.

3. **Enforce TLS on DocumentDB clusters** *(R3)* — do not create an `AWS::DocDB::DBClusterParameterGroup` that sets the `tls` parameter to `disabled`. When no parameter group explicitly disables TLS, DocumentDB defaults to requiring TLS. If a parameter group is required for other settings, ensure it does not include `tls: disabled`. All client connection strings must specify TLS.

4. **Generate strong Secrets Manager passwords** *(R4)* — `PasswordLength` must be at least 20 characters. Do not set `ExcludePunctuation: true` unless the consuming application strictly cannot accept special characters — and in that case, ensure the character space still provides ≥80 bits of entropy. Do not specify `ExcludeCharacters` in a way that removes most character classes.

5. **Do not expose managed services to the public internet** *(R5)* — set `publicly_accessible = false` on all `aws_db_instance`, `aws_redshift_cluster`, `aws_neptune_cluster_instance`, and `aws_mq_broker` resources. Control access through VPC security groups and private subnets. If external access is required, use a VPN or bastion host rather than direct public exposure.

6. **Require TLS on RDS Proxy connections** *(R6)* — set `require_tls = true` on all `aws_db_proxy` resources. Omitting `require_tls` defaults to false; even when the underlying RDS instance enforces TLS, client applications can connect to the proxy without TLS.

7. **Enforce HTTPS-only access on S3 buckets** *(R7)* — attach a bucket policy `Deny` conditioned on `"Bool": {"aws:SecureTransport": "false"}` to every `AWS::S3::Bucket`. TF: `aws_s3_bucket_policy` with the same condition.

8. **Terminate TLS at the load balancer** *(R8)* — CFN `AWS::ElasticLoadBalancingV2::Listener`: `Protocol: HTTPS`; any HTTP listener must have a `redirect` action to HTTPS (301). TF `aws_lb_listener`: `protocol = "HTTPS"` with an ACM cert ARN, or a `redirect` default action on any HTTP listener. Do not serve application traffic through HTTP listeners.

9. **Enforce SSL connections to RDS instances** *(R9)* — CFN: `AWS::RDS::DBParameterGroup` with `rds.force_ssl = 1` (PostgreSQL) or `require_secure_transport = ON` (MySQL), associated with the `AWS::RDS::DBInstance`. TF: `aws_db_parameter_group` with the engine-appropriate parameter, referenced via `parameter_group_name`.

10. **Enforce HTTPS on CloudFront distributions** *(R10)* — CFN `AWS::CloudFront::Distribution`: `ViewerProtocolPolicy: redirect-to-https` or `https-only` on all cache behaviors; never `allow-all`. TF: `viewer_protocol_policy = "redirect-to-https"` on all `default_cache_behavior` and `ordered_cache_behavior` blocks.

11. **Enforce TLS 1.2 on API Gateway custom domains** *(R11)* — CFN: `SecurityPolicy: TLS_1_2_2021` on `AWS::ApiGateway::DomainName` and `AWS::ApiGatewayV2::DomainName`. TF: `security_policy = "TLS_1_2"` on `aws_api_gateway_domain_name` and `aws_apigatewayv2_domain_name`.

12. **Enforce HTTPS and minimum TLS 1.2 on OpenSearch domains** *(R12)* — CFN (`AWS::OpenSearchService::Domain`, `AWS::Elasticsearch::Domain`): `DomainEndpointOptions.EnforceHTTPS: true`, `TLSSecurityPolicy: Policy-Min-TLS-1-2-2019-07`. TF: `domain_endpoint_options { enforce_https = true; tls_security_policy = "Policy-Min-TLS-1-2-2019-07" }`.

## Completion Evidence

*(Verify for every resource type present in the change.)*

- [ ] R1: No `Privileged: true` in any Batch job container definition
- [ ] R2: `TransitEncryptionEnabled: true` on all ElastiCache replication groups; Terraform `transit_encryption_enabled = true`
- [ ] R3: No DocDB cluster parameter group sets `tls` to `disabled`
- [ ] R4: Secrets Manager password length ≥ 20; `ExcludePunctuation: true` not used without compensating charset
- [ ] R5: `publicly_accessible = false` on all RDS, Redshift, Neptune, and MQ resources
- [ ] R6: `require_tls = true` on all `aws_db_proxy` resources; attribute not absent
- [ ] R7: Every S3 bucket has a bucket policy `Deny` conditioned on `aws:SecureTransport = false`
- [ ] R8: All ALB/NLB listeners use `Protocol: HTTPS`; any HTTP listener is a 301 redirect only
- [ ] R9: PostgreSQL RDS parameter groups have `rds.force_ssl = 1`; MySQL have `require_secure_transport = ON`
- [ ] R10: All CloudFront `ViewerProtocolPolicy` values are `redirect-to-https` or `https-only`
- [ ] R11: All API Gateway custom domain names use `SecurityPolicy: TLS_1_2_2021`
- [ ] R12: All OpenSearch/Elasticsearch domains have `EnforceHTTPS: true` and TLS 1.2 policy
