# 08 — Active Directory Domain Services Installation

## Environment
| Property | Value |
|----------|-------|
| VM | WinServer2025 |
| Hostname | DC01 |
| IP | 10.10.10.10 |
| Network | VMnet2 (10.10.10.0/24) |
| OS | Windows Server 2025 Standard (Desktop Experience) |
| Date | 2026-05-17 |

---

## Step 1 — Install AD DS Feature

```powershell
Install-WindowsFeature -Name AD-Domain-Services -IncludeManagementTools -Verbose
```

### Result
```
Success  Restart Needed  Exit Code  Feature Result
-------  --------------  ---------  --------------
True     No              Success    {Active Directory Domain Services, Management Tools...}
```

**No reboot required** after feature installation. Reboot happens after `Install-ADDSForest`.

---

## Step 2 — Promote Server to Domain Controller

```powershell
Install-ADDSForest `
  -DomainName "homelab.local" `
  -DomainNetbiosName "HOMELAB" `
  -ForestMode "WinThreshold" `
  -DomainMode "WinThreshold" `
  -InstallDns:$true `
  -SafeModeAdministratorPassword (ConvertTo-SecureString "Adm1n!str4tor" -AsPlainText -Force) `
  -Force:$true
```

### Parameters Explained
| Parameter | Value | Purpose |
|-----------|-------|---------|
| `-DomainName` | homelab.local | FQDN of the new domain |
| `-DomainNetbiosName` | HOMELAB | Legacy short name for the domain |
| `-ForestMode` | WinThreshold | Forest functional level (Windows Server 2016+) |
| `-DomainMode` | WinThreshold | Domain functional level (Windows Server 2016+) |
| `-InstallDns` | $true | Automatically install DNS Server role on this DC |
| `-SafeModeAdministratorPassword` | (secure) | DSRM (Directory Services Restore Mode) password |
| `-Force` | $true | Suppress interactive confirmation prompts |

### Warnings During Installation (Expected)
```
WARNING: Unable to create a delegation for this DNS server because the authoritative
parent zone was not found or does not run Windows DNS Server.
```
**This warning is normal** in an isolated lab environment. It means there is no parent
DNS zone (`.local` is not delegated from any public DNS root). Safe to ignore.

### Installation Progress
1. Validating environment and user input → All tests passed ✅
2. Installing new forest → In progress
3. Completing DNS installation → In progress
4. Operation completed: DCPromo.General.3 | Status: **Success** ✅
5. Server reboot initiated automatically

### Post-Reboot Credentials
| Field | Value |
|-------|-------|
| Username | `HOMELAB\Administrator` |
| Password | `Adm1n!str4tor` |
| DSRM Password | `Adm1n!str4tor` |

---

## Snapshot
```
Name: 01-dc01-addomain-installed-homelab-local
Taken: after successful reboot and domain login
```
