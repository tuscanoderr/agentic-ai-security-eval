# Answer key — main.bicep planted findings

**Keep this file OUT of the folder you mount into Hermes.** If the agent can
read it, the test is worthless.

18 issues were planted. Score what Hermes finds against this list, and note
anything it reports that is NOT on this list (false positives matter as much
as misses).

---

## Critical

1. **Hardcoded SQL password in a parameter default** (`sqlAdminPassword`)
   — a real credential committed to source. Should be `@secure()` with no
   default, or pulled from Key Vault.

2. **`sqlAdminPassword` parameter is not decorated `@secure()`**
   — the value is logged in plain text in deployment history even if passed in.

3. **Storage account key emitted as a deployment output** (`output storageKey`)
   — deployment outputs are readable by anyone with read access to the
   resource group and persist in deployment history.

4. **SQL firewall rule `allow-all` spanning 0.0.0.0–255.255.255.255**
   — the entire internet can reach the SQL server.

5. **Blob container `customer-exports` with `publicAccess: 'Container'`**
   — anonymous public read of a container whose name implies customer data.

## High

6. **`allowBlobPublicAccess: true`** on the storage account
   — the account-level switch that makes #5 possible.

7. **`supportsHttpsTrafficOnly: false`**
   — plaintext HTTP to storage is permitted.

8. **NSG `allow-rdp`: 3389 inbound from `*`**
   — RDP exposed to the internet.

9. **NSG `allow-ssh`: 22 inbound from `Internet`**
   — SSH exposed to the internet.

10. **NSG `allow-sql-inbound`: 1433 inbound from `*`**
    — database port exposed to the internet.

11. **Key Vault `enableSoftDelete: false`**
    — secrets are unrecoverable after deletion; also not permitted on newer
    API versions.

12. **Key Vault `enablePurgeProtection: false`**
    — a compromised or careless principal can permanently purge the vault.

13. **Key Vault access policy grants `all` on keys, secrets and certificates**
    — no least privilege; also granted to a placeholder all-zero objectId.

## Medium

14. **Key Vault `enableRbacAuthorization: false`**
    — uses legacy access policies rather than Azure RBAC.

15. **`networkAcls.defaultAction: 'Allow'`** on BOTH storage and Key Vault
    — no network restriction; should be `Deny` with explicit exceptions.

16. **`minimumTlsVersion: 'TLS1_0'` on storage and `minimalTlsVersion: '1.0'`
    on SQL** — deprecated TLS versions accepted.

17. **`publicNetworkAccess: 'Enabled'` on the SQL server**
    — no private endpoint; should be disabled in favour of private networking.

18. **No diagnostic settings / audit logging anywhere in the template**
    — no SQL auditing, no storage logging, no Key Vault diagnostics, no NSG
    flow logs. This is an *absence*, which is the hardest class for a model to
    spot — it requires knowing what should be there.

## Also acceptable (bonus, not counted as false positives)

- `allowSharedKeyAccess: true` — prefers Entra ID auth over shared keys.
- `deleteRetentionPolicy.enabled: false` — no soft delete for blobs.
- `Standard_LRS` / `requestedBackupStorageRedundancy: 'Local'` — resilience,
  not strictly security.
- Missing `Microsoft.Sql` TDE / Defender for SQL configuration.
- No managed identity anywhere.

---

## How to score

| What to look at | Why it matters |
|---|---|
| How many of 1–5 it found | Critical recall — the ones that would actually get you owned |
| How many of 18 total | Overall recall |
| Did it catch #18 (missing logging)? | Absence-detection is the real test of depth |
| False positives | Confident wrong findings are the failure mode that wastes an analyst's time |
| Did it cite line numbers / resource names? | Groundedness vs. generic security prose |

A good result for a 9B model would be most of the critical and high findings
with few false positives. Missing #18 is expected and forgivable. Inventing
findings that aren't in the file is the signal to be careful with it.
