# 09 — Active Directory Verification and DNS Configuration

## Date
2026-05-17 / 2026-05-18 / 2026-05-23

---

## Step 1 — Verify Domain (Get-ADDomain)

```powershell
Get-ADDomain
```

### Key Fields Verified
| Field | Value | Meaning |
|-------|-------|---------|
| `DNSRoot` | homelab.local | FQDN of the domain |
| `NetBIOSName` | HOMELAB | Legacy/short domain name |
| `DomainMode` | Windows2025Domain | Functional level |
| `PDCEmulator` | DC01.homelab.local | Primary Domain Controller role holder |
| `ReplicaDirectoryServers` | {DC01.homelab.local} | Only one DC in this lab |
| `DomainSID` | S-1-5-21-... | Unique domain Security Identifier |
| `InfrastructureMaster` | DC01.homelab.local | FSMO role holder |
| `RIDMaster` | DC01.homelab.local | FSMO role holder |

---

## Step 2 — Verify Core AD Services

```powershell
Get-Service adws, dns, kdc, netlogon | Select-Object Name, Status, StartType
```

### Result
```
Name      Status  StartType
----      ------  ---------
adws      Running Automatic
dns       Running Automatic
kdc       Running Automatic
netlogon  Running Automatic
```

All 4 core services are **Running** with **Automatic** startup. ✅

### Services Explained
| Service | Full Name | Role |
|---------|-----------|------|
| `adws` | Active Directory Web Services | Allows PowerShell AD cmdlets to communicate with AD |
| `dns` | DNS Server | Resolves domain names inside homelab.local |
| `kdc` | Kerberos Key Distribution Center | Issues Kerberos tickets for authentication |
| `netlogon` | Net Logon | Handles authentication and replication between DCs |

---

## Step 3 — Verify Domain Controller

```powershell
Get-ADDomainController -Filter * | Select-Object Name, IPv4Address, IsGlobalCatalog
```

### Result
```
Name  IPv4Address  IsGlobalCatalog
----  -----------  ---------------
DC01  10.10.10.10  True
```

- **IsGlobalCatalog: True** → DC01 is both a Domain Controller AND a Global Catalog server ✅
- In a single-DC lab, the GC is always on the only DC

---

## Step 4 — Verify DNS Zones

```powershell
Get-DnsServerZone
```

### Result
| ZoneName | ZoneType | IsDsIntegrated | IsReverseLookupZone |
|----------|----------|----------------|---------------------|
| _msdcs.homelab.local | Primary | True | False |
| 0.in-addr.arpa | Primary | False | True |
| 127.in-addr.arpa | Primary | False | True |
| 255.in-addr.arpa | Primary | False | True |
| homelab.local | Primary | **True** | False |

- `homelab.local` with `IsDsIntegrated: True` → DNS zone stored inside AD database ✅
- `_msdcs.homelab.local` → Microsoft DC Locator SRV records, required for AD to function

---

## Step 5 — Configure DNS Client on DC01

```powershell
Set-DnsClientServerAddress -InterfaceAlias "Ethernet0" `
  -ServerAddresses "127.0.0.1", "10.10.10.254"
```

### DNS Server Priority (Client)
| Priority | Server | Role |
|----------|--------|------|
| 1st | 127.0.0.1 | DC01 itself — resolves homelab.local and all AD records |
| 2nd | 10.10.10.254 | pfSense — emergency fallback only |

> **Why NOT a public DNS here:** The DNS client settings control where Windows
> sends its own queries. A DC must always query itself first. Adding 1.1.1.1 or
> 8.8.8.8 here would cause Windows to bypass its own DNS service for unknown names,
> breaking AD-integrated zones. Internet resolution is handled by the DNS Forwarder (Step 5b).

> **Linux vs Windows DC — key difference:**
> Linux machines (Kali, Ubuntu) are DNS clients only — no local DNS service.
> Pointing them to 1.1.1.1 is correct. DC01 runs a DNS Server role, so its
> client must query itself (127.0.0.1) first. These are architecturally different scenarios.

### Verify DNS Client Configuration

```powershell
Get-DnsClientServerAddress -InterfaceAlias "Ethernet0" | Select-Object -ExpandProperty ServerAddresses
# Result:
# 127.0.0.1
# 10.10.10.254  ✅
```

---

## Step 5b — Configure DNS Forwarder (Internet Resolution)

For internet name resolution (e.g. `google.com`), the correct approach is to configure
a **DNS Forwarder** on the DNS Server role — not to add public IPs to the client settings.

### Why pfSense cannot be the forwarder
pfSense DNS Resolver (Unbound) is not configured to accept recursive queries
from the OPT1/lab interface by default. Pointing the DC01 DNS forwarder to pfSense
(10.10.10.254) results in `RCODE_SERVER_FAILURE` — pfSense receives the DNS query
but does not respond to it as a resolver.

### Correct solution — public DNS as forwarder

```powershell
Set-DnsServerForwarder -IPAddress "8.8.8.8", "1.1.1.1" -PassThru
```

### Resolution flow

```
App on DC01 (or any domain client)
  │
  ▼
DC01 DNS Server (127.0.0.1)
  ├─ "homelab.local"? → answered internally from AD database ✅
  ├─ "DC01.homelab.local"? → answered internally ✅
  └─ "google.com"? → not in local zones
       │
       ▼
  DNS Forwarder → 8.8.8.8 (Google) or 1.1.1.1 (Cloudflare)
       │
       pfSense routes the UDP/53 packet via NAT (VMnet8) ✅
       [pfSense acts as ROUTER here, not as DNS server]
       │
       ▼
  Internet DNS responds → DC01 DNS returns answer to client ✅
```

### Why this works (and why pfSense didn't)
When Kali/Ubuntu used `1.1.1.1` as DNS, pfSense simply routed UDP/53 packets
to the internet — it never had to act as a DNS server. The same applies here:
DC01's DNS forwarder sends packets to `8.8.8.8`, pfSense routes them as plain
UDP traffic. pfSense firewall already allows port 53 from OPT1 to WAN.

### Verify

```powershell
Get-DnsServerForwarder
# UseRootHint    : True
# IPAddress      : {8.8.8.8, 1.1.1.1}
# ReorderedIPAddress : {8.8.8.8, 1.1.1.1}  ✅

Resolve-DnsName google.com
# google.com  A  172.217.x.x  ✅

ping google.com -n 2
# 0% packet loss  ✅

Resolve-DnsName homelab.local
# homelab.local  A  10.10.10.10  ✅ (internal still works)

Resolve-DnsName DC01.homelab.local
# DC01.homelab.local  A  10.10.10.10  ✅
```

### DNS Forwarder vs DNS Client — Key Difference
| Setting | Command | Scope |
|---------|---------|-------|
| **DNS Client** | `Set-DnsClientServerAddress` | Where Windows OS sends its own queries |
| **DNS Forwarder** | `Set-DnsServerForwarder` | Where the DNS Server role forwards unknown queries |

These are two completely independent settings. Confusing them is a common mistake.

---

## Step 6 — Final DNS Architecture

```
DC01 DNS Client settings:
  Primary:   127.0.0.1     (self — resolves AD/internal)
  Secondary: 10.10.10.254  (pfSense — emergency fallback)

DC01 DNS Server forwarder:
  8.8.8.8    (Google — internet resolution)
  1.1.1.1    (Cloudflare — internet resolution fallback)
```

---

## Snapshot
```
Name: 04-dc01-dns-forwarder-8888-internet-ok
Taken: after DNS forwarder fix confirmed working
```

---

## Fix Log

| Date | Issue | Attempted Fix | Result |
|------|-------|---------------|--------|
| 2026-05-18 | `1.1.1.1` added to DNS client settings | Removed `1.1.1.1` from `Set-DnsClientServerAddress`; set `Set-DnsServerForwarder` to pfSense (10.10.10.254) | ❌ Failed — pfSense Unbound not configured to resolve for OPT1 clients |
| 2026-05-23 | DNS forwarder pointing to pfSense fails with `RCODE_SERVER_FAILURE` | Changed `Set-DnsServerForwarder` to `8.8.8.8`, `1.1.1.1` — pfSense routes UDP/53 as plain traffic | ✅ Working — google.com resolves, homelab.local resolves |
