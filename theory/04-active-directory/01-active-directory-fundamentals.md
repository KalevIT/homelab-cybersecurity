# 01 — Active Directory: Fundamentals

## What Is Active Directory

**Active Directory (AD)** is Microsoft's directory service — a centralized
system for managing users, computers, groups, and resources in a Windows
network. Introduced in Windows 2000, it runs on virtually every
enterprise network in the world.

If you compromise Active Directory, you compromise the entire organization.
This is why AD attacks are the primary focus of real-world penetration tests.

---

## Core Concepts

### Domain
A **domain** is the fundamental unit of AD — a logical grouping of users,
computers, and resources that share a common directory database.

```
homelab.local          ← our domain name
├── Users/             ← user accounts
├── Computers/         ← joined workstations and servers
├── Groups/            ← collections of users/computers
└── Group Policies/    ← settings applied to users/computers
```

Every domain has a **Domain Controller (DC)** — the server that runs
Active Directory services and holds the directory database.

### Domain Controller (DC)
The DC is the heart of Active Directory. It:
- Authenticates users (verifies username/password)
- Authorizes access (checks permissions)
- Stores the AD database (`NTDS.dit`)
- Replicates changes to other DCs
- Runs DNS, LDAP, Kerberos services

In our Phase 4 lab: **Windows Server 2025** will be the DC.

### NTDS.dit
The **AD database file** stored on the DC at:
```
C:\Windows\NTDS\NTDS.dit
```
Contains:
- All user accounts and attributes
- **Password hashes** (NTLM hashes) for every user
- Group memberships
- Computer accounts
- Domain policies

If an attacker extracts `NTDS.dit`, they have the credentials
for every account in the domain — a complete domain compromise.

---

## Authentication: NTLM vs Kerberos

Active Directory supports two authentication protocols:

### NTLM (NT LAN Manager) — Legacy
Challenge-response protocol. Used when Kerberos isn't available.

```
1. Client sends username
2. Server sends random challenge (nonce)
3. Client responds with: NTLM_hash(password) + challenge → NTLM_response
4. DC verifies the response
```

**Why it's dangerous:** The NTLM hash itself can authenticate — if you
capture a hash, you don't need the plaintext password.
→ **Pass-the-Hash attack**

### Kerberos — Modern (default in AD)
Ticket-based protocol. Much more secure by design, but still attackable.
Full explanation in `04-kerberos-authentication.md`.

---

## Key AD Objects

### Users
Every person/service that needs to log in. Has:
- **sAMAccountName** — the login name (e.g., `jsmith`)
- **UPN** — User Principal Name (`jsmith@homelab.local`)
- **Password hash** — stored in NTDS.dit
- **Group memberships** — defines permissions

**Special users to know:**
| Account | Purpose | Risk |
|---|---|---|
| `Administrator` | Built-in local admin | High-value target |
| `krbtgt` | Kerberos ticket signing | Compromise = Golden Ticket |
| Service accounts | Run services (IIS, SQL) | Often over-privileged |
| Domain Admins | Full domain control | Primary target |

### Groups
Collections of users/computers that receive shared permissions.

**Critical built-in groups:**
| Group | Description | Attack Value |
|---|---|---|
| **Domain Admins** | Full control of the domain | Ultimate target |
| **Enterprise Admins** | Control across all domains | Forest-level control |
| **Schema Admins** | Modify AD schema | Rare but powerful |
| **Backup Operators** | Can backup/restore any file | Can read NTDS.dit |
| **Server Operators** | Log in to DCs | Dangerous privilege |

### Computers
Every machine joined to the domain has a computer account. Computers
have password hashes too — compromising a computer account lets you
move laterally.

### Group Policy Objects (GPOs)
Rules applied to users or computers. Controls:
- Password complexity requirements
- Software installation
- Logon scripts
- Security settings (firewall, BitLocker, etc.)

GPOs are hierarchical: Domain GPO → Organizational Unit GPO → Local.

---

## Organizational Units (OUs)

OUs are containers within a domain for organizing objects:

```
homelab.local
├── OU=IT Department
│   ├── User: Alice (IT Admin)
│   └── Computer: IT-PC-01
├── OU=Sales
│   ├── User: Bob (Salesperson)
│   └── Computer: SALES-PC-01
└── OU=Servers
    └── Computer: WEB-SERVER-01
```

OUs allow applying different GPOs to different departments.
In attacks, OU structure helps identify high-value targets.

---

## Trusts

**Domain trusts** allow users from one domain to access resources in another.

```
homelab.local ←──── trusts ────── partner.local
     │                                   │
  our users                         partner users
  can access                        can access our
  partner resources                 resources
```

Types:
- **One-way trust**: A trusts B (B's users can access A)
- **Two-way trust**: Both domains trust each other
- **Transitive trust**: A trusts B, B trusts C → A trusts C

**Attack relevance:** Trusts are often a lateral movement path between
organizations in a pentest.

---

## LDAP — How AD is Queried

**LDAP (Lightweight Directory Access Protocol)** is the protocol used
to read and write AD data. Every AD tool (BloodHound, ldapdomaindump,
PowerView) queries AD through LDAP.

Port 389 (unencrypted) / Port 636 (LDAPS, encrypted).

Example LDAP query: "Show me all users in Domain Admins group":
```
ldapsearch -H ldap://10.10.10.10 -b "DC=homelab,DC=local" \
  "(memberOf=CN=Domain Admins,CN=Users,DC=homelab,DC=local)"
```

Wireshark can capture LDAP traffic — useful for understanding what
tools are actually doing when they query AD.

---

## DNS in Active Directory

AD relies heavily on DNS. The DC runs a DNS server, and all clients
use it to locate services.

**SRV records** tell clients where to find DC services:
```
_ldap._tcp.homelab.local        → DC's IP (LDAP service)
_kerberos._tcp.homelab.local    → DC's IP (Kerberos)
_gc._tcp.homelab.local          → Global Catalog
```

When a Windows machine joins a domain, it updates its DNS to point
to the DC. This is why in our lab the DNS for VMs points to pfSense
(10.10.10.254) — in Phase 4 the Windows clients will point to the DC.

---

## Our Phase 4 Lab Setup

```
homelab.local Domain
│
├── Windows Server 2025 (DC)     10.10.10.10
│   ├── Active Directory Domain Services
│   ├── DNS Server
│   └── Kerberos KDC
│
├── Windows 11 (Client)          10.10.10.50
│   └── Joined to homelab.local domain
│
└── Kali Linux (Attacker)        10.10.10.100
    ├── BloodHound
    ├── Impacket
    ├── CrackMapExec
    └── Rubeus
```

Attacks we'll perform:
1. **Enumeration** — map the domain with BloodHound
2. **Kerberoasting** — request service tickets, crack offline
3. **AS-REP Roasting** — attack accounts without pre-authentication
4. **Pass-the-Hash** — authenticate with NTLM hash
5. **DCSync** — simulate DC replication to dump all hashes

---

## Why AD Is the Primary Target

In real enterprise environments:
- 90%+ of Fortune 500 companies use Active Directory
- Average enterprise has 500,000+ AD objects
- A single Domain Admin account = complete domain control
- Domain Admin → local admin on every machine in the domain

The goal of most real attacks is:
```
Initial access → Privilege escalation → Domain Admin → Data exfiltration
```

Ransomware groups specifically target Domain Admin because it allows
deploying ransomware to every machine simultaneously via GPO.

---

## Key Terminology Quick Reference

| Term | Definition |
|---|---|
| DC | Domain Controller — server running AD |
| NTDS.dit | AD database with all hashes |
| NTLM | Legacy auth protocol, hash-based |
| Kerberos | Modern auth protocol, ticket-based |
| TGT | Ticket Granting Ticket — Kerberos "master ticket" |
| TGS | Ticket Granting Service — access ticket for specific service |
| SPN | Service Principal Name — identifier for Kerberos service |
| GPO | Group Policy Object — settings applied to users/computers |
| OU | Organizational Unit — container for AD objects |
| Trust | Relationship allowing cross-domain access |
| krbtgt | Account that signs all Kerberos tickets |
| BloodHound | Tool that maps AD and finds attack paths |
| Impacket | Python library for Windows/AD protocol attacks |

---

## Key Takeaways

- Active Directory is the central nervous system of every Windows
  enterprise — compromise it and you own the organization
- The DC holds NTDS.dit with every user's password hash —
  the crown jewel of any AD attack
- NTLM is legacy but still everywhere — the hash alone authenticates
  (Pass-the-Hash), no cracking required
- Kerberos is the default modern protocol but has its own attack
  surface (Kerberoasting, Golden Ticket, Silver Ticket)
- Domain Admins have local admin rights on every machine in the domain
- Understanding the structure (users, groups, OUs, GPOs) is essential
  before attacking — BloodHound automates this mapping
