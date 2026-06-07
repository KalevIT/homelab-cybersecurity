# 05 - BloodHound Graph Analysis & Kerberoasting Theory

## What BloodHound Actually Does

BloodHound does not find vulnerabilities. It finds **attack paths** — sequences of relationships between objects (users, groups, computers, GPOs) that an attacker can chain together to gain elevated privileges.

The underlying model treats Active Directory as a **directed graph**:
- **Nodes:** users, computers, groups, OUs, GPOs, domains
- **Edges:** relationships like `MemberOf`, `GenericAll`, `Owns`, `HasSession`, `CanRCDP`, `AdminTo`

An attack path is any traversal from a low-privilege node to a high-value target (typically `Domain Admins` or `Domain Controllers`).

---

## Core Concepts

### Nodes

| Node Type | Description |
|---|---|
| `User` | Domain user account |
| `Computer` | Domain-joined machine |
| `Group` | Security or distribution group |
| `OU` | Organizational Unit |
| `GPO` | Group Policy Object |
| `Domain` | The AD domain itself |

### Edges (Attack-Relevant)

| Edge | Meaning | Abuse |
|---|---|---|
| `GenericAll` | Full control over the target object | Reset password, add to group, write SPNs |
| `GenericWrite` | Write arbitrary attributes | Set SPN → Kerberoast; write script path → execution |
| `Owns` | Object owner has implicit `WriteDacl` | Grant yourself any permission |
| `WriteDacl` | Modify the object's ACL | Grant `GenericAll` to self |
| `ForceChangePassword` | Reset password without knowing current | Account takeover |
| `AdminTo` | Local admin on a machine | Code execution, credential dump |
| `HasSession` | User has active session on a machine | Token impersonation if you have local admin |
| `CanRDP` | RDP access | Interactive session |
| `MemberOf` | Group membership | Inherit all edges of the group |
| `AllExtendedRights` | All extended rights on object | Includes `ForceChangePassword`, `GetChangesAll` (DCSync) |
| `GetChangesAll` | DCSync right | Dump all domain password hashes |

---

## SharpHound Collection Methods

SharpHound collects data by querying LDAP and making RPC/SMB calls from a domain-joined context.

```
SharpHound.exe -c All
```

Collection flags under `-c All`:

| Method | What it collects |
|---|---|
| `Session` | Active user sessions on machines (via NetSessionEnum) |
| `ACL` | Access control entries on all objects |
| `ObjectProps` | Object attributes (adminCount, enabled, lastLogon, SPN…) |
| `Trusts` | Domain and forest trust relationships |
| `Container` | OU and container structure |
| `GPOLocalGroup` | Local group membership pushed via GPO |
| `LocalAdmin` | Local administrators on domain computers |
| `RDP` | Who can RDP to which machines |
| `DCOM` | DCOM access rights |
| `PSRemote` | WinRM/PSRemote access |

### Why Run from a Domain-Joined Host

SharpHound requires an authenticated domain context. Running from a non-joined machine (e.g., Kali with `bloodhound-python`) fails when:
- LDAP signing is enforced (Windows Server 2025 default)
- Channel binding is required

SharpHound on a domain-joined Windows host uses the logged-in user's Kerberos ticket and bypasses these restrictions.

---

## Cypher Query Language

BloodHound uses **Neo4j's Cypher** query language to traverse the graph.

### Basic Structure

```cypher
MATCH (start)-[relationship]->(end) RETURN start, relationship, end
```

### Useful Queries

**All users with SPNs (Kerberoastable):**
```cypher
MATCH (u:User {hasspn:true}) RETURN u.name, u.serviceprincipalnames
```

**All GenericAll relationships:**
```cypher
MATCH p=(u)-[:GenericAll]->(t) RETURN p
```

**Shortest path from any user to Domain Admins:**
```cypher
MATCH p=shortestPath(
  (u:User)-[*1..]->(g:Group {name:"DOMAIN ADMINS@HOMELAB.LOCAL"})
) RETURN p
```

**Find computers where Domain Admins have sessions:**
```cypher
MATCH p=(g:Group {name:"DOMAIN ADMINS@HOMELAB.LOCAL"})-[:HasSession]->(c:Computer)
RETURN p
```

**Find all paths exploiting ACL edges:**
```cypher
MATCH p=(u:User)-[:GenericAll|GenericWrite|Owns|WriteDacl|ForceChangePassword*1..]->(t)
WHERE t.name CONTAINS "DOMAIN ADMINS"
RETURN p
```

**Kerberoastable users with path to DA:**
```cypher
MATCH p=shortestPath(
  (u:User {hasspn:true})-[*1..]->(g:Group {name:"DOMAIN ADMINS@HOMELAB.LOCAL"})
) RETURN p
```

---

## Kerberoasting — Mechanics

### Prerequisites

- Valid domain credentials (any user)
- One or more accounts with registered SPNs

### How It Works

1. The attacker requests a **TGS (Ticket Granting Service)** ticket for a service account SPN using their own TGT.
2. The KDC encrypts the TGS with the **service account's password hash**.
3. The attacker receives the encrypted TGS — offline cracking is possible without further interaction with the DC.

```
Attacker → KDC: "Give me a TGS for MSSQLSvc/dc01.homelab.local:1433"
KDC → Attacker: TGS encrypted with svc-sql's password hash
Attacker → hashcat: crack offline
```

The attack is **entirely passive from the DC's perspective** — TGS requests are normal Kerberos operations and generate only standard Windows event logs (Event ID 4769).

### Etype Matters

| Etype | Value | Algorithm | Hashcat Mode |
|---|---|---|---|
| RC4-HMAC | 23 | RC4 | 13100 |
| AES-128 | 17 | AES-128-CTS-HMAC-SHA1 | 19600 |
| AES-256 | 18 | AES-256-CTS-HMAC-SHA1 | 19700 |

**Windows Server 2025 defaults to AES-256 (etype 18).** Using mode `13100` on an etype 18 hash will silently fail — hashcat will show `Exhausted` with 0 recovered.

The hash prefix identifies the etype:
- `$krb5tgs$23$` → mode 13100
- `$krb5tgs$17$` → mode 19600
- `$krb5tgs$18$` → mode 19700

### Detection (Blue Team Perspective)

| Event ID | Description | Significance |
|---|---|---|
| 4769 | Kerberos Service Ticket Request | Normal, but bulk requests for multiple SPNs is anomalous |
| 4770 | Kerberos Service Ticket Renewed | Less relevant |

Detection indicators:
- Multiple TGS requests from a single user in a short window
- TGS requests for service accounts that are rarely accessed
- RC4 TGS requests (etype 23) in an environment that enforces AES — downgrade attack indicator

---

## ACL Abuse Paths (Relevant to homelab.local)

The lab domain was configured with intentionally weak ACLs to simulate common real-world misconfigurations.

### GenericAll on User → Password Reset

If principal A has `GenericAll` on user B:

```powershell
# On a machine where A is authenticated
Set-ADAccountPassword -Identity B `
  -NewPassword (ConvertTo-SecureString "NewP@ss2024!" -AsPlainText -Force) -Reset
```

### GenericWrite on User → Targeted Kerberoasting

If principal A has `GenericWrite` on user B (who does not already have an SPN):

```powershell
# Set a fake SPN on B
Set-ADUser -Identity B -ServicePrincipalNames @{Add="fake/spn"}
# Request TGS for the fake SPN → crack → remove SPN
```

### Owns → WriteDacl → GenericAll

If A `Owns` object B:
1. A grants themselves `WriteDacl` on B
2. A modifies B's DACL to grant themselves `GenericAll`
3. A now has full control over B

This three-step chain is common in misconfigured domains and is one of the paths BloodHound's `shortestPath` query will find.

---

## Hashcat Operational Notes

### Wordlist Size Warning

```
The wordlist or mask that you are using is too small.
Hashcat is expecting at least 2048 base words but only got 0.0% of that.
```

This warning appears when the wordlist has fewer than 2048 entries. It does not prevent cracking — use `--force` to suppress and proceed. The warning is about GPU utilization efficiency, not correctness.

### Potfile

hashcat stores cracked hashes in `~/.local/share/hashcat/hashcat.potfile`. On subsequent runs against the same hash, it will output:

```
INFO: Removed hash found as potfile entry.
```

This means the hash was already cracked in a previous session. Use `--show` to display stored results:

```bash
hashcat -m 19700 /tmp/kerberoast.txt --show
```

### `-w 3` Workload Profile

| Level | Profile | Description |
|---|---|---|
| 1 | Low | Reduced GPU load, system stays responsive |
| 2 | Default | Balanced |
| 3 | High | Maximum GPU utilization |
| 4 | Nightmare | May cause system instability |

For a lab homelab with integrated CPU cracking (no dedicated GPU), `-w 3` is safe.

---

## References

- BloodHound CE documentation: https://support.bloodhoundenterprise.io/
- impacket GetUserSPNs: https://github.com/fortra/impacket
- hashcat modes: https://hashcat.net/wiki/doku.php?id=hashcat
- Kerberoasting — The Hacker Recipes: https://www.thehacker.recipes/ad/movement/kerberos/kerberoast
- Event ID 4769 — Microsoft Docs: https://learn.microsoft.com/en-us/windows/security/threat-protection/auditing/event-4769
