# Session output, verbatim

Terminal output from the Hermes session, transcribed exactly as displayed.
This is the first of two artefacts from the same run. The second,
`generated-report.md`, is what the agent wrote to a file when asked to save
the same review. Compare them.

Model: `ornith-1.0-9b` via LM Studio
Harness: Hermes Agent v0.21.3
Prompt: *Read /workspace/main.bicep and perform a security review. For every
issue, give the resource name, the specific property, why it's a risk, and
the fix. Rank by severity. Don't summarize the file, I want findings.*

---

```
Security Review — main.bicep

Ranked by severity. Resource → Property → Risk → Fix.


CRITICAL (7)

1. SQL Server (sqlServer) — publicNetworkAccess: 'Enabled'
   SQL is directly reachable from the internet. Any host can attempt brute-force,
   exploit known CVEs, or pivot into the tenant's Active Directory.
   Fix: Set to 'Disabled' and route access through a private endpoint + NSG rule
   restricted to the app tier subnet.

2. SQL Firewall (sqlFirewallAll) — startIpAddress: '0.0.0.0' / endIpAddress: '255.255.255.255'
   Effectively opens SQL to every IP on Earth. Combined with public network access,
   the entire database is exposed.
   Fix: Remove the firewall rule entirely (private-only deployment) or restrict
   startIpAddress/endIpAddress to a specific subnet CIDR (e.g., 10.24.0.0/16).

3. SQL Admin Password (sqlAdminPassword) — hardcoded default 'P@ssw0rd123!'
   Trivially guessable, appears in plain text in the template repo, and is reused as
   the SQL server's administrator password (line 181).
   Fix: Remove the parameter entirely. Inject via ARM Template Parameters
   (secureParameter) or a Key Vault reference (reference(extensionData.sqlAdminPassword)).

4. Storage Account — allowBlobPublicAccess: true
   Any anonymous user can read, write, and delete blobs in every container including
   sensitive data. Ransomware exfil and data theft are immediate risks.
   Fix: Set to 'false'. Use private endpoints or shared access signatures (SAS) with
   expiration for any external readers.

5. Container (exportsContainer) — publicAccess: 'Container'
   The customer-exports container is readable by anyone on the internet. "Customer data"
   in a public-accessible bucket is a compliance and leak liability.
   Fix: Set to 'None'. If external reads are needed, use Azure AD authentication or SAS
   tokens scoped per request.

6. NSG — allow-rdp rule: sourceAddressPrefix: '*' port 3389
   RDP from the entire internet is a top attack vector (brute force, pass-the-hash,
   ZeroLog0-style exploits). One compromised RDP session = full domain compromise.
   Fix: Restrict to the corporate VPN subnet CIDR or disable if not required. If needed,
   enforce MFA and network level authentication via Azure Policy.

7. NSG — allow-sql-inbound rule: sourceAddressPrefix: '*' port 1433
   Same pattern as RDP but on the database tier — direct internet-accessible SQL endpoint
   with no firewall in front of it.
   Fix: Restrict to the app subnet CIDR, or move SQL behind a private DNS zone + private
   endpoints.


HIGH (5)

8. NSG — allow-ssh rule: sourceAddressPrefix: 'Internet' port 22
   SSH from anywhere enables remote code execution if any VM is vulnerable. Brute-force
   credential stuffing runs against it daily on the public internet.
   Fix: Restrict to VPN subnet or disable if not required for direct management.

9. Storage Account — supportsHttpsTrafficOnly: false
   HTTP traffic is permitted, enabling protocol downgrade attacks and man-in-the-middle
   interception of blob content.
   Fix: Set to 'true'. Azure will enforce HTTPS on all data plane operations.

10. Storage Account — minimumTlsVersion: 'TLS1_0'
    TLS 1.0 is deprecated (RFC 8996) and supports broken cipher suites vulnerable to
    POODLE, BEAST, and Lucky Thirteen.
    Fix: Set 'TLS1_2'. Azure recommends TLS1_2 as minimum; TLS1_3 if available.

11. Storage Account — allowSharedKeyAccess: true
    Allows generation of storage account keys with full admin-level SAS tokens. Any holder
    can impersonate the storage account and access everything it protects.
    Fix: Set to 'false'. Use per-resource SAS tokens or Azure AD identity-based access instead.

12. SQL Server — minimalTlsVersion: '1.0'
    Same vulnerability as on storage (point 10) but on the database tier — TLS 1.0
    connections can downgrade encrypted traffic to plaintext.
    Fix: Set '1.2'.


MEDIUM (4)

13. Key Vault (keyVault) — enableSoftDelete: false
    Accidental deletion results in immediate, unrecoverable loss of secrets, certificates,
    and keys. No purge protection window means no chance to recover.
    Fix: Set 'true'. This is a baseline requirement for any production vault
    (default 7-day retention).

14. Key Vault (keyVault) — enablePurgeProtection: false
    After soft-deletion, the vault can be permanently purged with no warning. Combined with
    soft-delete off (point 13), deletion is instant and total.
    Fix: Set 'true'. Enables a minimum 7-day purge protection window.

15. Key Vault — enableRbacAuthorization: false
    Falls back to legacy access policies, which are harder to audit, can't enforce
    fine-grained permissions per key/secret, and don't integrate with Azure AD
    Conditional Access.
    Fix: Set 'true'. Use AAD workloads or user-assigned managed identities for all
    vault access.

16. Key Vault — accessPolicies uses placeholder UUID '00000000-...'
    The object ID is the universal non-existent GUID. Whatever identity this resolves to
    (likely none) gets blanket keys:all, secrets:all, certificates:all. If it accidentally
    matches a real principal, that principal has unlimited vault access with no audit trail
    of what they accessed.
    Fix: Remove the placeholder. Assign actual AAD object IDs and scope permissions to
    specific keys/secrets/certificates rather than 'all'.


LOW (1)

17. Output — output storageKey string = storage.listKeys().keys[0].value (line 212)
    Exposes the storage account key as an ARM template output. Anyone with read access to
    the deployment can retrieve it via Azure Portal, REST API
    (/subscriptions/{id}/resourceGroups/.../providers/Microsoft.Resources/deployments/{name}/content),
    or az deployment group show. Keys in outputs are essentially plaintext credentials.
    Fix: Remove the storageKey output entirely. If external access is needed, use a SAS
    token with appropriate scope and expiration — never expose account keys as template outputs.


Summary: 17 findings across CRITICAL → LOW. The SQL Server path (public network +
allow-all firewall) is the single most dangerous combination in this file — it gives
anyone on the internet direct database access to a tenant's analytics workload. The
hardcoded password (P@ssw0rd123!) is trivially brute-forced and appears in source
control by default. The storage account is wide open for anonymous data exfiltration.
```

---

## Differences against `generated-report.md`

Both artefacts come from the same session, the same model, and the same input
file. The file version was produced immediately afterwards by asking the agent
to write the same review to disk. It regenerated rather than transcribed.

**Finding 1.** Session output ends at "pivot into the tenant's Active Directory."
The file adds "(e.g., MS17-014 EternalBlue variants)". EternalBlue is MS17-010
and it is an SMB vulnerability, unrelated to SQL Server.

**Finding 6.** Session output reads "brute force, pass-the-hash, ZeroLog0-style
exploits". The file drops the ZeroLogon reference and keeps pass-the-hash.
ZeroLogon is a Netlogon flaw against domain controllers, not an RDP exploit.

**Finding 7.** Session output reads "Same pattern as RDP but on the database
tier". The file adds "Enables SQL injection attacks from outside the network
perimeter". SQL injection is an application layer flaw with no relationship to
network exposure of port 1433.

**Finding 8.** Session output ends at "runs against it daily on the public
internet." The file adds "(e.g., Log4Shell, Cobalt Strike beacons)". Log4Shell
is a Java logging vulnerability and Cobalt Strike is a command and control
framework. Neither is an SSH brute force risk.

**Finding 15.** Session output correctly reads `enableRbacAuthorization: false`.
The file changes this to "unspecified". The template sets the value explicitly
at line 141, so the written report misdescribes its own source.

Every error present in the session output also persisted into the file. The
finding list itself was identical across both: same 17 items, same severity
ranking, same ordering.
