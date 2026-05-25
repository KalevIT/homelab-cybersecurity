# 04 — Windows AD Administration: Commands Reference for Linux Users

> This document is a practical reference for the PowerShell commands and
> Windows-specific concepts used during the AD setup phase (setup/08→10).
> It is written for someone with a Linux/networking background learning
> Windows Server administration for the first time.
>
> For conceptual AD theory → see `01-active-directory-fundamentals.md`
> For Kerberos attacks → see `02-kerberos-authentication.md`
> For AD attack methodology → see `03-ad-attacks-overview.md`

---

## Table of Contents
1. [PowerShell vs Linux Commands](#1-powershell-vs-linux-commands)
2. [Install-WindowsFeature Explained](#2-install-windowsfeature-explained)
3. [Install-ADDSForest Explained](#3-install-addsforest-explained)
4. [AD Verification Commands Explained](#4-ad-verification-commands-explained)
5. [DNS in Active Directory — Client vs Forwarder](#5-dns-in-active-directory--client-vs-forwarder)
6. [User and OU Management Explained](#6-user-and-ou-management-explained)
7. [FSMO Roles](#7-fsmo-roles)

---

## 1. PowerShell vs Linux Commands

PowerShell uses a **Verb-Noun** syntax — consistent across all cmdlets.

| PowerShell | Linux Equivalent | Example |
|-----------|-----------------|---------|
| `Get-` | read / show / cat / ls | `Get-Service` = `systemctl status` |
| `Set-` | modify / configure | `Set-DnsClientServerAddress` = edit `/etc/resolv.conf` |
| `New-` | create | `New-ADUser` = `useradd` |
| `Add-` | add to existing | `Add-ADGroupMember` = `usermod -aG` |
| `Install-` | install | `Install-WindowsFeature` = `apt install` |
| `Remove-` | delete | `Remove-ADUser` = `userdel` |
| `Resolve-` | query / lookup | `Resolve-DnsName` = `nslookup` / `dig` |

### Pipeline
PowerShell uses `|` just like Linux:
```powershell
Get-Service | Select-Object Name, Status   # show only Name and Status columns
Get-ADUser -Filter * | Format-Table        # show in table format
```

### Variables
```powershell
$usersOU = "OU=LabUsers,DC=homelab,DC=local"   # like: export VAR=value in bash
```

### ConvertTo-SecureString
PowerShell cmdlets that accept passwords require a **SecureString** type.
```powershell
ConvertTo-SecureString "MyPassword" -AsPlainText -Force
```
`-AsPlainText -Force` → "I know I'm passing plain text, do it anyway."

### Backtick `` ` `` — Line Continuation
In PowerShell, the backtick is the line continuation character (like `\` in bash):
```powershell
New-ADUser -Name "Alice" `
  -SamAccountName "alice" `    # ← continues on next line
  -Enabled $true
```

---

## 2. Install-WindowsFeature Explained

```powershell
Install-WindowsFeature -Name AD-Domain-Services -IncludeManagementTools -Verbose
```

| Parameter | Meaning |
|-----------|---------|
| `-Name AD-Domain-Services` | The Windows Server role to install |
| `-IncludeManagementTools` | Also install GUI tools (ADUC, DNS Manager, etc.) |
| `-Verbose` | Show detailed progress output |

**Linux analogy:** `apt install slapd ldap-utils` (rough OpenLDAP equivalent)

This only installs the **software binaries**. AD is not configured yet.
The actual domain creation happens in the next step.

---

## 3. Install-ADDSForest Explained

```powershell
Install-ADDSForest `
  -DomainName "homelab.local" `
  -DomainNetbiosName "HOMELAB" `
  -ForestMode "WinThreshold" `
  -DomainMode "WinThreshold" `
  -InstallDns:$true `
  -SafeModeAdministratorPassword (ConvertTo-SecureString "..." -AsPlainText -Force) `
  -Force:$true
```

### What This Command Actually Does (in order)
1. Creates the AD forest and domain structure
2. Creates the AD database (`NTDS.dit`) at `C:\Windows\NTDS\`
3. Creates `SYSVOL` share (used for GPO replication)
4. Installs and configures the DNS Server role
5. Creates DNS zones for `homelab.local`
6. Configures Kerberos (KDC service)
7. Reboots the server automatically

### Parameter Reference
| Parameter | Value | Purpose |
|-----------|-------|---------|
| `-DomainName` | homelab.local | FQDN of the new domain |
| `-DomainNetbiosName` | HOMELAB | Legacy short name — used in `DOMAIN\username` login |
| `-ForestMode` | WinThreshold | Windows Server 2016+ functional level |
| `-DomainMode` | WinThreshold | Same — Microsoft didn't update the constant for 2019/2022/2025 |
| `-InstallDns` | $true | Auto-install DNS role on this DC |
| `-SafeModeAdministratorPassword` | (secure) | DSRM recovery password — store safely |
| `-Force` | $true | Skip interactive "Continue? [Y/N]" prompt |

### Expected Warning (safe to ignore)
```
WARNING: Unable to create a delegation for this DNS server because the
authoritative parent zone was not found...
```
Normal in isolated labs — `.local` doesn't exist in public DNS. No action needed.

---

## 4. AD Verification Commands Explained

### Get-ADDomain
```powershell
Get-ADDomain
```
**Linux analogy:** Reading `/etc/ldap/slapd.conf` + running `slapcat`

Key output fields:
| Field | Meaning |
|-------|---------|
| `DNSRoot` | FQDN of the domain (homelab.local) |
| `NetBIOSName` | Legacy name used in `DOMAIN\user` login format |
| `DomainMode` | Functional level (Windows2025Domain) |
| `PDCEmulator` | DC holding the "master time" FSMO role |
| `DomainSID` | Unique domain identifier — all user SIDs derive from this |

### Get-Service
```powershell
Get-Service adws, dns, kdc, netlogon | Select-Object Name, Status, StartType
```
**Linux analogy:** `systemctl status adws dns kdc netlogon`

| Service | Why It Must Run |
|---------|----------------|
| `adws` | Without this, PowerShell AD cmdlets fail entirely |
| `dns` | Without DNS, machines can't find the DC → no logins |
| `kdc` | Without Kerberos, authentication is impossible |
| `netlogon` | Without Netlogon, computers can't authenticate |

### Get-ADDomainController
```powershell
Get-ADDomainController -Filter * | Select-Object Name, IPv4Address, IsGlobalCatalog
```
`IsGlobalCatalog: True` → this DC stores a searchable copy of all forest objects.
In a single-DC lab, the GC is always on the only DC.

### Get-DnsServerZone
```powershell
Get-DnsServerZone
```
**Linux analogy:** Listing zone files in `/etc/bind/`

`IsDsIntegrated: True` on `homelab.local` → the DNS zone is stored inside the AD
database and replicates automatically with AD. This is the preferred mode.

---

## 5. DNS in Active Directory — Client vs Forwarder

This is the most misunderstood configuration point when coming from Linux.
There are **two completely independent** DNS settings on a Windows Server DC.

```
┌─────────────────────────────────────────────────────────────────┐
│  SETTING 1: DNS CLIENT (Set-DnsClientServerAddress)             │
│  → Controls where Windows OS sends its own DNS queries          │
│  → Linux equivalent: /etc/resolv.conf                           │
│  → On a DC: MUST be 127.0.0.1 first                            │
│                                                                  │
│  SETTING 2: DNS FORWARDER (Set-DnsServerForwarder)              │
│  → Controls where the DNS Server ROLE forwards unknown queries  │
│  → Linux equivalent: "forwarders { };" in BIND9 named.conf     │
│  → This is where upstream DNS goes (8.8.8.8, 1.1.1.1, etc.)   │
└─────────────────────────────────────────────────────────────────┘
```

### Set-DnsClientServerAddress — DNS Client
```powershell
Set-DnsClientServerAddress -InterfaceAlias "Ethernet0" `
  -ServerAddresses "127.0.0.1", "10.10.10.254"
```

**Linux analogy** (`/etc/resolv.conf`):
```
nameserver 127.0.0.1    # DC's own DNS service
nameserver 10.10.10.254 # pfSense fallback
```

**Why `127.0.0.1` first and NEVER a public IP here:**
The DC must query its own DNS service to resolve `homelab.local` and all AD
SRV records. If you add `1.1.1.1` here, Windows bypasses its own DNS for
unknown names → AD-integrated zones break.

> **Common Mistake:** Adding `1.1.1.1` to DNS client settings for internet access.
> This works for pinging google.com but is architecturally wrong for a DC.
> Internet resolution belongs in the DNS Forwarder (below).

### Set-DnsServerForwarder — DNS Forwarder
```powershell
Set-DnsServerForwarder -IPAddress "8.8.8.8", "1.1.1.1" -PassThru
```

**Linux analogy** (BIND9 `/etc/bind/named.conf.options`):
```
options {
    forwarders { 8.8.8.8; 1.1.1.1; };
};
```

### Why pfSense Cannot Be the Forwarder in This Lab
pfSense DNS Resolver (Unbound) is not configured to accept recursive queries
from the OPT1/lab network by default. Setting the forwarder to `10.10.10.254`
results in `RCODE_SERVER_FAILURE`. Setting it to `8.8.8.8`/`1.1.1.1` works
because pfSense routes UDP/53 packets as plain traffic — proven by the fact
that Kali/Ubuntu could already reach `1.1.1.1` on port 53.

### Full Resolution Flow (Working Configuration)
```
App on DC01 asks: "what is google.com?"
    │
    ▼
DC01 DNS Server (127.0.0.1)
    ├─ "homelab.local"? → answered from AD database locally ✅
    └─ "google.com"? → not in local zones
         │
         ▼ DNS Forwarder
    8.8.8.8 or 1.1.1.1 (external)
         │
    pfSense routes UDP/53 via NAT (VMnet8) — acts as ROUTER, not DNS server ✅
         │
         ▼
    Internet DNS responds → DC01 returns answer to client ✅
```

### Linux vs DC DNS — Why They're Different
| Machine | Has local DNS service? | DNS client points to | Internet via |
|---------|----------------------|---------------------|--------------|
| Kali/Ubuntu | ❌ No | 1.1.1.1 directly | pfSense routing UDP/53 |
| DC01 | ✅ Yes (DNS Server role) | 127.0.0.1 (itself) | DNS Forwarder → 8.8.8.8 |

Linux machines are pure DNS clients — they ask external servers directly.
DC01 is a DNS server — its client asks itself, then it forwards externally.

---

## 6. User and OU Management Explained

### New-ADOrganizationalUnit
```powershell
New-ADOrganizationalUnit -Name "LabUsers" -Path "DC=homelab,DC=local"
```
**Linux analogy:** `mkdir` — creating a folder inside the AD directory tree.

### New-ADUser
```powershell
New-ADUser -Name "Alice Rossi" -SamAccountName "alice.rossi" `
  -UserPrincipalName "alice.rossi@homelab.local" `
  -AccountPassword (ConvertTo-SecureString "Password123!" -AsPlainText -Force) `
  -Enabled $true `
  -Path "OU=LabUsers,DC=homelab,DC=local" `
  -PasswordNeverExpires $true
```

| Parameter | Linux Equivalent | What It Does |
|-----------|-----------------|--------------|
| `-Name` | GECOS field | Display name shown in GUI |
| `-SamAccountName` | username in `/etc/passwd` | Login name for `DOMAIN\username` |
| `-UserPrincipalName` | N/A | Modern email-format login |
| `-AccountPassword` | `passwd username` | Sets the password |
| `-Enabled $true` | `usermod -U` | Activates the account |
| `-Path` | Home directory location | Which OU to place the user in |
| `-PasswordNeverExpires` | `chage -M -1` | Disables password expiration |

### Set-ADUser with -ServicePrincipalNames
```powershell
Set-ADUser "svc-sql" -ServicePrincipalNames @{Add="MSSQLSvc/dc01.homelab.local:1433"}
```
An SPN (Service Principal Name) is a unique identifier for a service on a host.
Format: `ServiceClass/Host:Port`

Setting an SPN makes the account **Kerberoastable** — any domain user can request
a Kerberos service ticket encrypted with this account's password hash, then crack it offline.

### Add-ADGroupMember
```powershell
Add-ADGroupMember -Identity "Domain Admins" -Members "labadmin"
```
**Linux analogy:** `usermod -aG sudo labadmin`

### Get-ADUser
```powershell
Get-ADUser -Filter * -SearchBase "DC=homelab,DC=local" | Select-Object Name, SamAccountName, Enabled
```
**Linux analogy:** `cat /etc/passwd | awk -F: '{print $1}'`

`-Filter *` → return ALL users (equivalent to `WHERE 1=1` in SQL).
`-SearchBase` → start searching from this point in the AD directory tree.

---

## 7. FSMO Roles

FSMO = Flexible Single Master Operations.
Some AD operations can only be performed by ONE DC to avoid conflicts.

In a single-DC lab, DC01 holds ALL 5 FSMO roles — normal.

| FSMO Role | Holder | Purpose |
|-----------|--------|---------|
| PDC Emulator | DC01 | Master time source; handles password changes |
| RID Master | DC01 | Allocates pools of unique IDs for new objects |
| Infrastructure Master | DC01 | Updates cross-domain object references |
| Schema Master | DC01 | Only DC that can modify the AD schema |
| Domain Naming Master | DC01 | Controls adding/removing domains from forest |

> **Security note:** Compromising the PDC Emulator gives control over domain time.
> Kerberos is time-sensitive (±5 minutes) — time manipulation can bypass authentication.

---

## Final Lab DNS Configuration Summary

```
DC01 DNS Client (Set-DnsClientServerAddress):
  Primary:   127.0.0.1     → DC01 itself (resolves homelab.local + all AD records)
  Secondary: 10.10.10.254  → pfSense (emergency fallback only)

DC01 DNS Server Forwarder (Set-DnsServerForwarder):
  8.8.8.8   → Google DNS (internet resolution)
  1.1.1.1   → Cloudflare DNS (fallback)
```
