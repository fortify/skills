# Azure Infrastructure as Code Security

## Risks

| # | Risk | Detection signal | Fortify Category | CWE | OWASP 2025 |
|---|------|-----------------|-----------------|-----|------------|
| R1 | App Service HTTPS not enforced | `Microsoft.Web/sites` or `Microsoft.Web/sites/config` with `httpsOnly: false` or `httpsOnly` property absent; Terraform `azurerm_app_service`, `azurerm_linux_web_app`, or `azurerm_windows_web_app` with `https_only = false`; Ansible `azure_rm_webapp` with `https_only: false` or property absent | Azure ARM Misconfiguration: HTTPS Not Required | CWE-319 | A04 Cryptographic Failures |
| R2 | MySQL SSL enforcement disabled | `Microsoft.DBforMySQL/servers` with `sslEnforcement: "Disabled"`; Terraform `azurerm_mysql_server` with `ssl_enforcement_enabled = false`; Ansible `azure_rm_mysqlserver` with `enforce_ssl: false` | Azure ARM Misconfiguration: Insecure MySQL Server Transport | CWE-297 | A07 Authentication Failures |
| R3 | PostgreSQL SSL enforcement disabled | `Microsoft.DBforPostgreSQL/servers` with `sslEnforcement: "Disabled"`; Terraform `azurerm_postgresql_server` with `ssl_enforcement_enabled = false`; Ansible `azure_rm_postgresqlserver` with `enforce_ssl: false` | Azure ARM Misconfiguration: Insecure PostgreSQL Server Transport | CWE-297 | A07 Authentication Failures |
| R4 | Storage account permits HTTP traffic | `Microsoft.Storage/storageAccounts` with `supportsHttpsTrafficOnly: false` or property absent; Terraform `azurerm_storage_account` with `enable_https_traffic_only = false` or `https_traffic_only_enabled = false`; Ansible `azure_rm_storageaccount` with `https_only: false` | Azure ARM Misconfiguration: Insecure Storage Account Transport | CWE-311 | A02 Security Misconfiguration |
| R5 | Custom role with wildcard actions | ARM `Microsoft.Authorization/roleDefinitions` resource where `properties.permissions[].actions` contains `"*"` or `"*/*"`; Bicep `roleDefinition` resource with `actions: ['*']` | Azure ARM Misconfiguration: Improper Custom Role Access Control Policy | CWE-250 | A01 Broken Access Control |
| R6 | SQL Database server allows weak TLS | Terraform `azurerm_mssql_server` with `minimum_tls_version` set to `"1.0"` or `"1.1"`, or property absent (defaults to permitting TLS 1.0); ARM `Microsoft.Sql/servers` with `properties.minimalTlsVersion` absent or below `"1.2"` | Azure Terraform Misconfiguration: Insecure SQL Database Transport | CWE-319 | A04 Cryptographic Failures |
| R7 | Function App HTTPS not enforced | `Microsoft.Web/sites` of `kind: functionapp` with `httpsOnly: false` or property absent; Terraform `azurerm_function_app`, `azurerm_linux_function_app`, or `azurerm_windows_function_app` with `https_only = false` or attribute absent | Azure Terraform Misconfiguration: Insecure Function App Transport | CWE-319 | A04 Cryptographic Failures |
| R8 | AKS cluster has RBAC disabled | `Microsoft.ContainerService/managedClusters` with `properties.enableRBAC: false`; Terraform `azurerm_kubernetes_cluster` with `role_based_access_control_enabled = false` | Azure ARM Misconfiguration: Improper AKS Access Control | CWE-287 | A01 Broken Access Control |
| R9 | Redis Cache non-SSL port enabled or weak TLS | `Microsoft.Cache/Redis` with `properties.enableNonSslPort: true`; Terraform `azurerm_redis_cache` with `enable_non_ssl_port = true` (provider < 3.x) or `non_ssl_port_enabled = true` (provider ≥ 3.x); or `minimum_tls_version` set below `"1.2"` | Azure Terraform Misconfiguration: Insecure Redis Cache Transport | CWE-319 | A04 Cryptographic Failures |
| R10 | Application Gateway HTTP listener without redirect | `Microsoft.Network/applicationGateways` with an HTTP listener that has no associated redirect configuration to HTTPS; Terraform `azurerm_application_gateway` with an `http_listener` block using `protocol = "Http"` and no corresponding redirect routing rule | Azure ARM Misconfiguration: Insecure Application Gateway Transport | CWE-319 | A04 Cryptographic Failures |
| R11 | Service Bus namespace minimum TLS below 1.2 | `Microsoft.ServiceBus/namespaces` with `properties.minimumTlsVersion: "1.0"` or `"1.1"`, or property absent (defaults to permitting TLS 1.0); Terraform `azurerm_servicebus_namespace` with `minimum_tls_version = "1.0"` or `"1.1"` | Azure ARM Misconfiguration: Insecure Service Bus Transport | CWE-319 | A04 Cryptographic Failures |

## Required Agent Actions

1. **Enforce HTTPS on all App Service resources** *(R1)* — ARM `Microsoft.Web/sites`: `httpsOnly: true`; TF (`azurerm_app_service`, `azurerm_linux_web_app`, `azurerm_windows_web_app`): `https_only = true`; Ansible `azure_rm_webapp`: `https_only: true`.

2. **Enforce SSL on MySQL server connections** *(R2)* — ARM `Microsoft.DBforMySQL/servers`: `sslEnforcement: "Enabled"`; TF `azurerm_mysql_server`: `ssl_enforcement_enabled = true` (or `ssl_minimal_tls_version_enforced = "TLS1_2"` for newer provider versions); Ansible `azure_rm_mysqlserver`: `enforce_ssl: true`.

3. **Enforce SSL on PostgreSQL server connections** *(R3)* — ARM `Microsoft.DBforPostgreSQL/servers`: `sslEnforcement: "Enabled"`; TF `azurerm_postgresql_server`: `ssl_enforcement_enabled = true` (or `ssl_minimal_tls_version_enforced = "TLS1_2"` for newer provider versions); Ansible `azure_rm_postgresqlserver`: `enforce_ssl: true`.

4. **Restrict storage accounts to HTTPS only** *(R4)* — ARM `Microsoft.Storage/storageAccounts`: `supportsHttpsTrafficOnly: true`; TF `azurerm_storage_account`: `enable_https_traffic_only = true` (provider < 3.x) or `https_traffic_only_enabled = true` (≥ 3.x).

5. **Do not create custom roles with wildcard actions** *(R5)* — never define a `Microsoft.Authorization/roleDefinitions` resource with `"*"` or `"*/*"` in the `actions` array. Enumerate only the specific resource actions required (e.g., `"Microsoft.Storage/storageAccounts/read"`). If broad access is genuinely needed, assign the built-in `Owner` or `Contributor` role rather than recreating it as a custom role.

6. **Enforce minimum TLS 1.2 on SQL Database servers** *(R6)* — TF (`azurerm_mssql_server`, `azurerm_sql_server`): `minimum_tls_version = "1.2"`; ARM `Microsoft.Sql/servers`: `properties.minimalTlsVersion: "1.2"`.

7. **Enforce HTTPS on all Function App resources** *(R7)* — TF (`azurerm_function_app`, `azurerm_linux_function_app`, `azurerm_windows_function_app`): `https_only = true`; ARM `Microsoft.Web/sites` (kind `functionapp`): `httpsOnly: true`.

8. **Enable RBAC on all AKS clusters** *(R8)* — TF `azurerm_kubernetes_cluster`: `role_based_access_control_enabled = true`; ARM `Microsoft.ContainerService/managedClusters`: `properties.enableRBAC: true`.

9. **Disable non-SSL port and enforce TLS 1.2 on Redis Cache** *(R9)* — TF `azurerm_redis_cache`: `non_ssl_port_enabled = false` (provider ≥ 3.x) or `enable_non_ssl_port = false` (< 3.x), `minimum_tls_version = "1.2"`; ARM `Microsoft.Cache/Redis`: `properties.enableNonSslPort: false`, `properties.minimumTlsVersion: "1.2"`.

10. **Redirect HTTP to HTTPS on Application Gateway** *(R10)* — TF `azurerm_application_gateway`: add a redirect configuration (type `Permanent`) from each HTTP listener to the corresponding HTTPS listener. ARM `Microsoft.Network/applicationGateways`: add a redirect configuration of type `Permanent` targeting the HTTPS frontend. Do not route application traffic through HTTP listeners.

11. **Enforce minimum TLS 1.2 on Service Bus namespaces** *(R11)* — TF `azurerm_servicebus_namespace`: `minimum_tls_version = "1.2"`; ARM `Microsoft.ServiceBus/namespaces`: `properties.minimumTlsVersion: "1.2"`.

## Completion Evidence

*(Verify for every resource type present in the change.)*

- [ ] R1: `httpsOnly: true` on all App Service resources; `https_only = true` in Terraform
- [ ] R2: `sslEnforcement: "Enabled"` on all MySQL server resources; `ssl_enforcement_enabled = true` in Terraform
- [ ] R3: `sslEnforcement: "Enabled"` on all PostgreSQL server resources; `ssl_enforcement_enabled = true` in Terraform
- [ ] R4: `supportsHttpsTrafficOnly: true` on all storage account resources
- [ ] R5: No `Microsoft.Authorization/roleDefinitions` contains `"*"` or `"*/*"` in its `actions` array
- [ ] R6: `minimum_tls_version = "1.2"` on all SQL server resources; `minimalTlsVersion: "1.2"` in ARM
- [ ] R7: `https_only = true` on all Function App resources
- [ ] R8: `role_based_access_control_enabled = true` on all AKS clusters; RBAC not explicitly disabled
- [ ] R9: Non-SSL port disabled on all Redis Cache instances; `minimum_tls_version = "1.2"` applied
- [ ] R10: All Application Gateway HTTP listeners have a permanent redirect to HTTPS
- [ ] R11: `minimum_tls_version = "1.2"` on all Service Bus namespaces
