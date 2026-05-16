# 02 — Common Network Protocols

## Overview

A **protocol** is a set of rules that defines how two systems communicate.
Every service we attacked in this lab speaks a specific protocol.
Understanding the protocol means understanding both how to use it
legitimately and how to exploit it offensively.

---

## FTP — File Transfer Protocol (Port 21)

**Purpose:** Transfer files between client and server.
**Year designed:** 1971. Older than the internet as we know it.
**Encryption:** None. Everything in plaintext.

### How FTP Works
```
Client connects to port 21 (control channel)
Client sends: USER username\r\n
Server sends: 331 Password required
Client sends: PASS password\r\n
Server sends: 230 Login successful
Client sends: LIST (list directory)
Server opens port 20 (data channel) and sends directory listing
```

Two channels: **control** (port 21, commands) and **data** (port 20,
file transfer). This split design is why FTP is complex and often
causes firewall headaches.

### Why FTP Is Dangerous
1. **No encryption** — username and password visible to anyone sniffing the network
2. **Old implementations have backdoors** — vsftpd 2.3.4 (our exploit)
3. **Anonymous FTP** — many servers allow login without credentials
4. **Active vs Passive mode** — often misconfigured, causes firewall bypasses

### What We Did
Wireshark filter `ftp` showed the entire FTP conversation in plaintext:
- `220 vsFTPd 2.3.4` — server announces version (information disclosure)
- `USER sjUTx:)` — Metasploit sent username containing `:)` trigger
- `PASS yrr67T` — password (irrelevant, backdoor already triggered)
- `500 OOPS` — backdoor activated

**Defense:** Replace FTP with SFTP (SSH File Transfer Protocol) or FTPS.
Never expose FTP to the internet. Disable anonymous login.

---

## SSH — Secure Shell (Port 22)

**Purpose:** Encrypted remote shell access, file transfer (SCP/SFTP).
**Year designed:** 1995 (replacing Telnet).
**Encryption:** Yes — asymmetric key exchange + symmetric session encryption.

### How SSH Works
```
1. Client connects to port 22
2. Server sends its public key (host key)
3. Client verifies: "Is this the right server?" (prevents MITM)
4. Key exchange (Diffie-Hellman): establish shared session key
5. All subsequent traffic is encrypted with AES (or similar)
6. Client authenticates: password OR public key
7. Encrypted shell session begins
```

### SSH Key-Based Authentication
More secure than passwords — uses asymmetric cryptography:

```bash
# Generate key pair on client
ssh-keygen -t ed25519 -C "kali@homelab"
# Creates: ~/.ssh/id_ed25519 (private) + id_ed25519.pub (public)

# Copy public key to server
ssh-copy-id user@10.10.10.105
# Server stores it in: ~/.ssh/authorized_keys

# Now connect without password
ssh user@10.10.10.105
```

**Private key = identity. Never share it. Losing it or having it stolen
means anyone can authenticate as you.**

### Old SSH Algorithms (Metasploitable)
Metasploitable2 uses deprecated key exchange algorithms. Modern SSH
clients reject them by default:
```bash
# Force old algorithms to connect to Metasploitable
ssh -o KexAlgorithms=+diffie-hellman-group1-sha1 \
    -o HostKeyAlgorithms=+ssh-rsa \
    msfadmin@10.10.10.101
```

**Why deprecated?** `diffie-hellman-group1-sha1` uses 1024-bit keys,
now considered breakable with sufficient compute power.

### The First-Connection Warning
```
The authenticity of host '10.10.10.105' can't be established.
ECDSA key fingerprint is SHA256:xxxx...
Are you sure you want to continue connecting (yes/no)?
```
This is SSH showing the server's **host key fingerprint**. Saying "yes"
stores it in `~/.ssh/known_hosts`. If the fingerprint changes on a
subsequent connection, SSH warns you — possible MITM attack.

**Defense:** SSH is already secure by design. Use key-based auth,
disable root login, disable password auth, change default port (minor).

---

## Telnet (Port 23)

**Purpose:** Remote shell — the predecessor of SSH.
**Encryption:** None. Username, password, and all commands in plaintext.

Telnet should never be used in production. It exists on Metasploitable2
because it's a legacy system from 2008. In Wireshark, you can see every
keystroke of a Telnet session in real time.

---

## SMB — Server Message Block (Ports 139, 445)

**Purpose:** Windows file sharing, printer sharing, inter-process communication.
**Year designed:** 1983. Evolved into CIFS, then SMB2, SMB3.
**Encryption:** SMB3 supports encryption. Older versions don't.

### How SMB Works
```
Client ──▶ Server:445
SMB Negotiate Protocol (agree on SMB version)
SMB Session Setup (authenticate: NTLM or Kerberos)
SMB Tree Connect (connect to share: \\server\share)
SMB Create/Read/Write (file operations)
```

### Why SMB Is Critical in AD Attacks
1. **NTLM authentication** — if you can intercept SMB auth, you get NTLM hashes
2. **Relay attacks** — captured NTLM hash can be relayed to authenticate elsewhere
3. **SMB Signing disabled** — required for relay attacks to work
4. **Named pipes** — used for inter-process communication, basis for many attacks
5. **Lateral movement** — `psexec`, `wmiexec`, `smbexec` all use SMB

### EternalBlue (CVE-2017-0144) — Famous SMB Vulnerability
The NSA-developed exploit leaked by Shadow Brokers in 2017.
Exploits a buffer overflow in SMBv1.
Used by WannaCry ransomware (May 2017) to spread globally.
Metasploit module: `exploit/windows/smb/ms17_010_eternalblue`

Metasploitable2 (Samba 3.x) has SMB vulnerabilities we haven't
explored yet — they'll be more relevant in Phase 4.

**Defense:** Disable SMBv1, enable SMB signing, block ports 139/445
at network perimeter, patch regularly.

---

## HTTP/HTTPS (Ports 80/443)

**HTTP (HyperText Transfer Protocol):** Transfers web content. Plaintext.
**HTTPS:** HTTP over TLS. Encrypted. The `S` means Secure.

### HTTP Request/Response
```
Client request:
GET /index.html HTTP/1.1
Host: 10.10.10.101
User-Agent: Mozilla/5.0

Server response:
HTTP/1.1 200 OK
Content-Type: text/html
<html>...</html>
```

### In Our Lab
Metasploitable2 runs Apache 2.2.8 on port 80 with multiple vulnerable
web applications (DVWA, Mutillidae, phpMyAdmin).

Wazuh Dashboard runs HTTPS on port 443 with a self-signed certificate
(hence the browser warning — no trusted CA signed it).

**Defense:** Always use HTTPS. Implement proper certificate management.
Use a WAF (Web Application Firewall) for production web servers.

---

## DNS — Domain Name System (Port 53)

**Purpose:** Translate domain names to IP addresses.
**Protocol:** UDP (for queries), TCP (for large responses and zone transfers).
**Encryption:** None by default. DNS over HTTPS (DoH) is the modern alternative.

### DNS Resolution Process
```
Browser: "What is google.com?"
         ↓
Local DNS cache: miss
         ↓
Resolver (pfSense/ISP): query
         ↓
Root server: "Ask .com TLD server"
         ↓
.com TLD server: "Ask google's nameserver"
         ↓
Google's nameserver: "172.217.23.174"
         ↓
Browser connects to 172.217.23.174
```

### DNS Record Types
| Record | Purpose | Example |
|---|---|---|
| A | IPv4 address | `google.com → 172.217.23.174` |
| AAAA | IPv6 address | `google.com → 2607:f8b0:...` |
| MX | Mail server | `@gmail.com → smtp.gmail.com` |
| CNAME | Alias | `www.google.com → google.com` |
| PTR | Reverse lookup (IP→name) | `172.217.23.174 → google.com` |
| SRV | Service location | `_ldap._tcp.homelab.local → dc.homelab.local` |
| TXT | Arbitrary text | SPF, DKIM records |

### DNS in Active Directory
AD is entirely dependent on DNS. SRV records tell clients:
- Where to find the Domain Controller
- Where Kerberos service is
- Where the Global Catalog is

In Phase 4: Windows clients will use the DC's IP as their DNS server.
Without correct DNS → cannot join domain, cannot authenticate.

### DNS Attacks
- **DNS Spoofing/Cache Poisoning** — inject false DNS records
- **DNS Zone Transfer** — dump all DNS records (often misconfigured)
- **DNS Enumeration** — discover infrastructure from DNS records

```bash
# Zone transfer attempt (often reveals internal infrastructure)
dig axfr homelab.local @10.20.20.10
```

---

## LDAP — Lightweight Directory Access Protocol (Port 389/636)

**Purpose:** Query and modify directory services. Used by Active Directory.
**Port 389:** Unencrypted. **Port 636:** LDAPS (LDAP over SSL).

### How LDAP Structures Data
LDAP uses a hierarchical tree structure called a **DIT (Directory Information Tree)**:

```
DC=homelab,DC=local                    ← domain root
├── CN=Users                           ← users container
│   ├── CN=Administrator               ← user object
│   └── CN=kaliuser                    ← user object
├── CN=Computers                       ← computers container
├── OU=IT Department                   ← organizational unit
│   └── CN=alice,OU=IT Department      ← user in OU
└── CN=Domain Admins,CN=Users          ← group object
```

### Common LDAP Queries in Pentesting
```bash
# Enumerate all users (unauthenticated on misconfigured AD)
ldapsearch -H ldap://10.20.20.10 -x -b "DC=homelab,DC=local" \
  "(objectClass=user)" sAMAccountName

# Find Domain Admins
ldapsearch -H ldap://10.20.20.10 -x -b "DC=homelab,DC=local" \
  "(memberOf=CN=Domain Admins,CN=Users,DC=homelab,DC=local)"

# Find computers
ldapsearch -H ldap://10.20.20.10 -x -b "DC=homelab,DC=local" \
  "(objectClass=computer)" name
```

BloodHound (our Phase 4 tool) uses LDAP to map the entire AD structure
and find privilege escalation paths.

---

## Kerberos — Authentication Protocol (Port 88)

Full detail in `04-active-directory/02-kerberos-authentication.md`.
Brief overview here:

**Purpose:** Ticket-based authentication in Active Directory.
**Port:** 88 TCP/UDP.

### Key Concept: Tickets Instead of Passwords
```
Traditional auth: "Here's my password" → server verifies
Kerberos: "Here's my ticket" → server verifies ticket's signature
```

Password never leaves the client. Tickets have limited lifetime (10 hours).

### Three Actors
- **KDC (Key Distribution Center)** — runs on the DC, issues tickets
- **Client** — the user or computer requesting access
- **Service** — the resource the client wants to access (file share, SQL, etc.)

### Why It's Attackable
1. **Kerberoasting** — request tickets for services, crack offline
2. **AS-REP Roasting** — accounts without pre-auth leak encrypted data
3. **Golden Ticket** — forge TGTs using the `krbtgt` hash
4. **Silver Ticket** — forge TGS tickets for specific services
5. **Pass-the-Ticket** — steal and reuse valid tickets

---

## Protocol Summary — Our Lab

| Protocol | Port | Encryption | Used In Lab | Attack |
|---|---|---|---|---|
| FTP | 21 | ❌ None | vsftpd exploit | Backdoor, plaintext creds |
| SSH | 22 | ✅ Full | Kali → Ubuntu | Deprecated algorithms on Meta |
| Telnet | 23 | ❌ None | Available on Meta | Cleartext session |
| HTTP | 80 | ❌ None | Apache on Meta | Web vulns (Phase future) |
| SMB | 445 | Partial | Samba on Meta | Relay, EternalBlue (Phase 4) |
| HTTPS | 443 | ✅ Full | Wazuh dashboard | Self-signed cert warning |
| IRC | 6667 | ❌ None | UnrealIRCd exploit | Backdoor trigger |
| LDAP | 389 | ❌ None | AD (Phase 4) | Enumeration, relay |
| Kerberos | 88 | Partial | AD (Phase 4) | Roasting, ticket attacks |
| DNS | 53 | ❌ None | pfSense resolver | Spoofing, zone transfer |

---

## Key Takeaways

- Every protocol has a specific purpose and design era — older protocols
  (FTP, Telnet) were designed when security wasn't a priority
- "No encryption" means Wireshark can read everything, including passwords
- SSH replaced Telnet/rsh — always use SSH for remote access
- SMB is the backbone of Windows networking and the most exploited
  protocol in corporate environments
- DNS is the phonebook of the internet — if an attacker controls DNS,
  they can redirect traffic anywhere
- LDAP is how Active Directory is queried — every AD attack tool uses it
- Understanding a protocol means understanding its attack surface
