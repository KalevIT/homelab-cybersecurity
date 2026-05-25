# 10 — AD Users and Organizational Unit Structure

## Date
2026-05-18

---

## Overview

A realistic AD environment was created with intentionally vulnerable accounts
to support red team exercises (BloodHound, Kerberoasting, Pass-the-Hash, etc.)

---

## Step 1 — Create Organizational Units (OUs)

```powershell
New-ADOrganizationalUnit -Name "LabUsers"        -Path "DC=homelab,DC=local"
New-ADOrganizationalUnit -Name "LabAdmins"       -Path "DC=homelab,DC=local"
New-ADOrganizationalUnit -Name "ServiceAccounts" -Path "DC=homelab,DC=local"
New-ADOrganizationalUnit -Name "Workstations"    -Path "DC=homelab,DC=local"
```

### OU Structure
```
DC=homelab,DC=local
├── OU=LabUsers          → Standard domain users
├── OU=LabAdmins         → Privileged accounts
├── OU=ServiceAccounts   → Kerberoastable service accounts
├── OU=Workstations      → Computer objects (CLIENT01 will go here)
├── CN=Users             → Default container (built-in accounts)
└── OU=Domain Controllers → DC01
```

---

## Step 2 — Create Standard Users

```powershell
$usersOU = "OU=LabUsers,DC=homelab,DC=local"

New-ADUser -Name "Alice Rossi" -SamAccountName "alice.rossi" `
  -UserPrincipalName "alice.rossi@homelab.local" `
  -AccountPassword (ConvertTo-SecureString "Password123!" -AsPlainText -Force) `
  -Enabled $true -Path $usersOU -PasswordNeverExpires $true

New-ADUser -Name "Bob Verdi" -SamAccountName "bob.verdi" `
  -UserPrincipalName "bob.verdi@homelab.local" `
  -AccountPassword (ConvertTo-SecureString "Summer2024!" -AsPlainText -Force) `
  -Enabled $true -Path $usersOU -PasswordNeverExpires $true

New-ADUser -Name "Charlie Bianchi" -SamAccountName "charlie.bianchi" `
  -UserPrincipalName "charlie.bianchi@homelab.local" `
  -AccountPassword (ConvertTo-SecureString "Welcome1" -AsPlainText -Force) `
  -Enabled $true -Path $usersOU -PasswordNeverExpires $true
```

---

## Step 3 — Create Service Accounts (Kerberoastable)

```powershell
$svcOU = "OU=ServiceAccounts,DC=homelab,DC=local"

New-ADUser -Name "svc-sql" -SamAccountName "svc-sql" `
  -UserPrincipalName "svc-sql@homelab.local" `
  -AccountPassword (ConvertTo-SecureString "SqlService2024!" -AsPlainText -Force) `
  -Enabled $true -Path $svcOU -PasswordNeverExpires $true

# Set SPN → makes account Kerberoastable ⚠️
Set-ADUser "svc-sql" -ServicePrincipalNames @{Add="MSSQLSvc/dc01.homelab.local:1433"}

New-ADUser -Name "svc-backup" -SamAccountName "svc-backup" `
  -UserPrincipalName "svc-backup@homelab.local" `
  -AccountPassword (ConvertTo-SecureString "Backup!2024" -AsPlainText -Force) `
  -Enabled $true -Path $svcOU -PasswordNeverExpires $true

Set-ADUser "svc-backup" -ServicePrincipalNames @{Add="BackupSvc/dc01.homelab.local"}
```

---

## Step 4 — Create Domain Admin Account

```powershell
$admOU = "OU=LabAdmins,DC=homelab,DC=local"

New-ADUser -Name "Lab Admin" -SamAccountName "labadmin" `
  -UserPrincipalName "labadmin@homelab.local" `
  -AccountPassword (ConvertTo-SecureString "L4bAdm1n!" -AsPlainText -Force) `
  -Enabled $true -Path $admOU -PasswordNeverExpires $true

Add-ADGroupMember -Identity "Domain Admins" -Members "labadmin"
```

---

## Step 5 — Verify All Users

```powershell
Get-ADUser -Filter * -SearchBase "DC=homelab,DC=local" |
  Select-Object Name, SamAccountName, Enabled |
  Format-Table -AutoSize
```

### Result
```
Name            SamAccountName   Enabled
----            --------------   -------
Administrator   Administrator    True
Guest           Guest            False
krbtgt          krbtgt           False
Alice Rossi     alice.rossi      True
Bob Verdi       bob.verdi        True
Charlie Bianchi charlie.bianchi  True
svc-sql         svc-sql          True
svc-backup      svc-backup       True
Lab Admin       labadmin         True
```

---

## User Summary Table

| Username | Password | OU | Role | Attack Target |
|----------|----------|----|------|---------------|
| alice.rossi | Password123! | LabUsers | Standard user | Password spray |
| bob.verdi | Summer2024! | LabUsers | Standard user | Password spray |
| charlie.bianchi | Welcome1 | LabUsers | Standard user | Weak password |
| svc-sql | SqlService2024! | ServiceAccounts | Service account | **Kerberoasting** |
| svc-backup | Backup!2024 | ServiceAccounts | Service account | **Kerberoasting** |
| labadmin | L4bAdm1n! | LabAdmins | Domain Admin | Privilege escalation target |

### Built-in Accounts (Default)
| Account | Status | Notes |
|---------|--------|-------|
| Administrator | Enabled | Domain built-in admin |
| Guest | Disabled | Always disabled by default (good) |
| krbtgt | Disabled | Kerberos ticket-granting account — never enable manually |

---

## Why These Accounts Are Vulnerable

### svc-sql and svc-backup — Kerberoasting Targets
These accounts have a **Service Principal Name (SPN)** set.
In AD, any authenticated domain user can request a Kerberos service ticket for any SPN.
That ticket is encrypted with the service account's NTLM hash.
An attacker can capture that ticket and crack it offline → password recovered.

### charlie.bianchi — Weak Password
`Welcome1` is a common password found in wordlists like rockyou.txt.
Vulnerable to **password spray** attacks.

### labadmin — Privilege Escalation Target
Once a low-privilege account is compromised, attackers look for paths to Domain Admin.
BloodHound will map the shortest privilege escalation path to this account.

---

## Snapshot
```
Name: 03-dc01-ad-users-created
Taken: after all OUs and users verified
```

## Next Steps
- [ ] Install CLIENT01 (Windows 11) — IP: 10.10.10.50
- [ ] Join CLIENT01 to homelab.local
- [ ] Log in as alice.rossi on CLIENT01
- [ ] Run BloodHound from Kali to enumerate AD
- [ ] Kerberoasting: GetUserSPNs.py against svc-sql and svc-backup
