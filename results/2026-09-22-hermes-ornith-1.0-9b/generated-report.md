# Security Review: `main.bicep` (Contoso Lab Environment)

## Summary

17 findings across CRITICAL → LOW severity levels. The SQL Server path (`publicNetworkAccess: 'Enabled'` + firewall covering 0.0.0.0/0) is the single most dangerous combination — it gives anyone on the internet direct database access to the tenant's analytics workload. The hardcoded password (`P@ssw0rd123!`) is trivially brute-forced and appears in source control by default. The storage account is wide open for anonymous data exfiltration.

---

## CRITICAL (7)

### 1. SQL Server — `publicNetworkAccess: 'Enabled'`
- **Resource:** `sqlServer` (`Microsoft.Sql/servers@2023-05-01`)
- **Property:** Line 182, `properties.publicNetworkAccess`
- **Risk:** SQL is directly reachable from the internet. Any host can attempt brute-force attacks, exploit known CVEs (e.g., MS17-014 EternalBlue variants), or pivot into the tenant's Active Directory using stolen credentials.
- **Fix:** Set to `'Disabled'`. Route access through a private endpoint backed by an NSG rule restricted to the application tier subnet only.

### 2. SQL Firewall — `startIpAddress: '0.0.0.0'` / `endIpAddress: '255.255.255.255'`
- **Resource:** `sqlFirewallAll` (`Microsoft.Sql/servers/firewallRules@2023-05-01-preview`)
- **Property:** Lines 191–192, `properties.startIpAddress` and `properties.endIpAddress`
- **Risk:** Effectively opens SQL port 1433 to every IP address on Earth. Combined with public network access (point #1), the database is fully exposed.
- **Fix:** Remove the firewall rule entirely — a private-only deployment makes this unnecessary. If external connectivity must exist, restrict `startIpAddress`/`endIpAddress` to a specific CIDR (e.g., `10.24.0.0/16`).

### 3. SQL Admin Password — hardcoded default `'P@ssw0rd123!'`
- **Resource:** Parameter block (`main.bicep`, lines 10–14)
- **Property:** Line 14, `param sqlAdminPassword string = 'P@ssw0rd123!'`
- **Risk:** Trivially guessable. Appears in plain text in the template repository (no `secureParameter`). Reused directly as the SQL server's administrator login password on line 181. Anyone with read access to the repo can authenticate and compromise the database.
- **Fix:** Remove the parameter entirely. Inject via ARM Template Parameters (`secureParameter`) at deployment time, or reference from Key Vault using `reference(extensionData.sqlAdminPassword)`.

### 4. Storage Account — `allowBlobPublicAccess: true`
- **Resource:** `storage` (`Microsoft.Storage/storageAccounts@2023-01-01`)
- **Property:** Line 26, `properties.allowBlobPublicAccess`
- **Risk:** Any anonymous user can read, write, and delete blobs in every container. Ransomware exfiltration and data theft are immediate risks — attacker only needs to know the storage account name (which is also exposed as an output on line 211).
- **Fix:** Set to `'false'`. Use private endpoints or shared access signatures (SAS) with expiration for any external readers.

### 5. Container (`exportsContainer`) — `publicAccess: 'Container'`
- **Resource:** `exportsContainer` (`Microsoft.Storage/storageAccounts/blobServices/containers@2023-01-01`)
- **Property:** Line 51, `properties.publicAccess`
- **Risk:** The `customer-exports` container is readable by anyone on the internet. "Customer data" in a publicly accessible bucket creates immediate compliance violations (GDPR/HIPAA) and enables data leak liability at scale.
- **Fix:** Set to `'None'`. If external reads are required, use Azure AD authentication or SAS tokens scoped per request with short expiration.

### 6. NSG — RDP rule: `sourceAddressPrefix: '*'` port 3389
- **Resource:** `nsg` (`Microsoft.Network/networkSecurityGroups@2023-05-01`)
- **Property:** Lines 63–73, rule `'allow-rdp'`, `properties.sourceAddressPrefix: '*'` with `destinationPortRange: '3389'`
- **Risk:** RDP from the entire internet is a top attack vector. Brute-force attacks and credential-reuse exploits (e.g., Pass-the-Hash after initial compromise) target this port relentlessly. One compromised RDP session can lead to full domain compromise via lateral movement into Azure AD.
- **Fix:** Restrict `sourceAddressPrefix` to the corporate VPN subnet CIDR or disable entirely if not required. If RDP must be enabled, enforce MFA and Network Level Authentication (NLA) via Azure Policy.

### 7. NSG — SQL inbound rule: `sourceAddressPrefix: '*'` port 1433
- **Resource:** `nsg` (`Microsoft.Network/networkSecurityGroups@2023-05-01`)
- **Property:** Lines 89–99, rule `'allow-sql-inbound'`, `properties.sourceAddressPrefix: '*'` with `destinationPortRange: '1433'`
- **Risk:** Direct internet-accessible SQL endpoint with no firewall in front of it. Enables SQL injection attacks from outside the network perimeter and bypasses the internal NSG entirely.
- **Fix:** Restrict `sourceAddressPrefix` to the application subnet CIDR, or move SQL behind a private DNS zone + Azure Private Endpoints so external traffic is impossible.

---

## HIGH (5)

### 8. NSG — SSH rule: `sourceAddressPrefix: 'Internet'` port 22
- **Resource:** `nsg` (`Microsoft.Network/networkSecurityGroups@2023-05-01`)
- **Property:** Lines 76–87, rule `'allow-ssh'`, `properties.sourceAddressPrefix: 'Internet'` with `destinationPortRange: '22'`
- **Risk:** SSH from any public IP enables remote code execution if any VM is vulnerable. Credential stuffing and brute-force attacks run against this port daily on the public internet (e.g., Log4Shell, Cobalt Strike beacons).
- **Fix:** Restrict `sourceAddressPrefix` to the corporate VPN subnet CIDR or disable entirely if not required for direct management.

### 9. Storage Account — `supportsHttpsTrafficOnly: false`
- **Resource:** `storage` (`Microsoft.Storage/storageAccounts@2023-01-01`)
- **Property:** Line 27, `properties.supportsHttpsTrafficOnly`
- **Risk:** HTTP traffic is permitted on the data plane. Enables protocol downgrade attacks and man-in-the-middle interception of blob content in transit.
- **Fix:** Set to `'true'`. Azure will enforce HTTPS on all data-plane operations when this flag is true.

### 10. Storage Account — `minimumTlsVersion: 'TLS1_0'`
- **Resource:** `storage` (`Microsoft.Storage/storageAccounts@2023-01-01`)
- **Property:** Line 28, `properties.minimumTlsVersion`
- **Risk:** TLS 1.0 is deprecated by RFC 8996 and supports broken cipher suites vulnerable to POODLE, BEAST, Lucky Thirteen, and SWEET32 attacks. Downgrade from TLS 1.2/1.3 is trivial for an attacker on the wire.
- **Fix:** Set `'TLS1_2'`. Azure recommends `TLS1_2` as minimum; enable `TLS1_3` if available via the newer SKU settings.

### 11. Storage Account — `allowSharedKeyAccess: true`
- **Resource:** `storage` (`Microsoft.Storage/storageAccounts@2023-01-01`)
- **Property:** Line 29, `properties.allowSharedKeyAccess`
- **Risk:** Allows generation of storage account keys with full admin-level SAS tokens. Any holder can impersonate the storage account and access every resource it protects (blobs, files, queues, tables, all containers).
- **Fix:** Set to `'false'`. Use per-resource SAS tokens or Azure AD identity-based access instead — never generate account-wide shared key access.

### 12. SQL Server — `minimalTlsVersion: '1.0'`
- **Resource:** `sqlServer` (`Microsoft.Sql/servers@2023-05-01-preview`)
- **Property:** Line 183, `properties.minimalTlsVersion`
- **Risk:** Same TLS downgrade vulnerability as on storage (point #10). Encrypted database connections can be downgraded to plaintext.
- **Fix:** Set `'1.2'`.

---

## MEDIUM (4)

### 13. Key Vault — `enableSoftDelete: false`
- **Resource:** `keyVault` (`Microsoft.KeyVault/vaults@2023-07-01`)
- **Property:** Line 139, `properties.enableSoftDelete`
- **Risk:** Accidental or malicious deletion results in immediate, unrecoverable loss of secrets, certificates, and keys. No retention window means no chance to recover from a mistake or an attacker who deletes vault contents before detection.
- **Fix:** Set `'true'`. This is a baseline requirement for any production vault (Azure enforces minimum 7-day soft-delete retention when enabled).

### 14. Key Vault — `enablePurgeProtection: false`
- **Resource:** `keyVault` (`Microsoft.KeyVault/vaults@2023-07-01`)
- **Property:** Line 140, `properties.enablePurgeProtection`
- **Risk:** After soft-deletion, the vault can be permanently purged with no warning. Combined with soft-delete off (point #13), deletion is instant and total — secrets are gone forever immediately upon delete operation.
- **Fix:** Set `'true'`. Enables a minimum 7-day purge protection window so accidental deletions can be recovered via the backup vault or portal.

### 15. Key Vault — `enableRbacAuthorization` unspecified
- **Resource:** `keyVault` (`Microsoft.KeyVault/vaults@2023-07-01`)
- **Property:** Line 142, `properties.enableRbacAuthorization`
- **Risk:** Falls back to legacy access policies, which are harder to audit, cannot enforce fine-grained permissions per individual key/secret/certificate, and do not integrate with Azure AD Conditional Access or entitlement management.
- **Fix:** Set `'true'`. Use AAD workloads (for service-to-service) or user-assigned managed identities (for VMs/app services) for all vault access.

### 16. Key Vault — `accessPolicies` uses placeholder UUID `'00000000-...'`
- **Resource:** `keyVault` (`Microsoft.KeyVault/vaults@2023-07-01`)
- **Property:** Line 145, `properties.accessPolicies[0].objectId: '00000000-0000-0000-000000000000'`
- **Risk:** The object ID is the universal non-existent GUID. Whatever identity this resolves to (likely none, but possibly an existing principal on some tenants) gets blanket `keys:all`, `secrets:all`, `certificates:all`. If it accidentally matches a real Azure AD user or service principal, that principal has unlimited vault access with no granular audit trail of *what* they accessed.
- **Fix:** Remove the placeholder. Assign actual Azure AD object IDs and scope permissions to specific keys/secrets/certificates rather than using `'all'`.

---

## LOW (1)

### 17. Output — `storageKey` exposes account key
- **Resource:** Template output block (`main.bicep`, line 212), `output storageKey string = storage.listKeys().keys[0].value`
- **Risk:** Exposes the storage account key as an ARM template output. Anyone with read access to the deployment (via Azure Portal, REST API `/subscriptions/{id}/resourceGroups/.../providers/Microsoft.Resources/deployments/{name}/content`, or `az deployment group show`) can retrieve it in plaintext. Account keys are equivalent to full admin credentials for that storage account — effectively all data is exposed.
- **Fix:** Remove the `storageKey` output entirely. If external access to storage blobs is needed, use a SAS token with appropriate scope and expiration — never expose account keys as template outputs.

---

## Affected Resources Summary

| # | Resource | Severity | Key Property |
|---|----------|----------|-------------|
| 1 | `sqlServer` | CRITICAL | `publicNetworkAccess: 'Enabled'` |
| 2 | `sqlFirewallAll` | CRITICAL | `startIpAddress: '0.0.0.0'` |
| 3 | Parameter block | CRITICAL | `sqlAdminPassword = 'P@ssw0rd123!'` |
| 4 | `storage` | CRITICAL | `allowBlobPublicAccess: true` |
| 5 | `exportsContainer` | CRITICAL | `publicAccess: 'Container'` |
| 6 | `nsg` (RDP) | CRITICAL | `sourceAddressPrefix: '*'`, port 3389 |
| 7 | `nsg` (SQL) | CRITICAL | `sourceAddressPrefix: '*'`, port 1433 |
| 8 | `nsg` (SSH) | HIGH | `sourceAddressPrefix: 'Internet'`, port 22 |
| 9 | `storage` | HIGH | `supportsHttpsTrafficOnly: false` |
| 10 | `storage` | HIGH | `minimumTlsVersion: 'TLS1_0'` |
| 11 | `storage` | HIGH | `allowSharedKeyAccess: true` |
| 12 | `sqlServer` | HIGH | `minimalTlsVersion: '1.0'` |
| 13 | `keyVault` | MEDIUM | `enableSoftDelete: false` |
| 14 | `keyVault` | MEDIUM | `enablePurgeProtection: false` |
| 15 | `keyVault` | MEDIUM | `enableRbacAuthorization` unspecified |
| 16 | `keyVault` | MEDIUM | Placeholder UUID in access policies |
| 17 | Template output | LOW | `storageKey` exposes account key |
