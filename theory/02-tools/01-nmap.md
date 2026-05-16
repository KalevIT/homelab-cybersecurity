# 03 — Nmap: Network Mapper Complete Guide

## What Is Nmap

**Nmap (Network Mapper)** is the industry-standard tool for network
discovery and security auditing. Created by Gordon Lyon (Fyodor) in 1997,
it's the first tool used in virtually every penetration test.

In our lab: `nmap -sV 10.10.10.101` was our first offensive command —
it revealed all vulnerable services on Metasploitable2.

---

## Why Nmap First

Before exploiting anything, you need to know:
1. **Which hosts are alive** (host discovery)
2. **Which ports are open** (port scanning)
3. **What services are running** (service detection)
4. **What OS is running** (OS detection)
5. **What vulnerabilities exist** (script scanning)

Nmap answers all five questions. The output directly feeds into
the choice of exploits.

---

## How Port Scanning Works

### The Core Idea
A port is either:
- **Open** — a service is listening, it responds to connection attempts
- **Closed** — no service listening, the OS sends RST (reset)
- **Filtered** — a firewall drops packets, no response

Nmap sends crafted packets and interprets responses.

### TCP Connect Scan (-sT) — Full 3-Way Handshake
```
Nmap ──SYN──▶ Target:22
Nmap ◀──SYN-ACK── Target    → Port OPEN (service responding)
Nmap ──ACK──▶ Target        (complete handshake)
Nmap ──RST──▶ Target        (close — we don't actually want to connect)

Nmap ──SYN──▶ Target:9999
Nmap ◀──RST── Target        → Port CLOSED (nothing listening)

Nmap ──SYN──▶ Target:1234
[no response / timeout]     → Port FILTERED (firewall dropping packets)
```

Slow, noisy, detectable. Used when SYN scan requires root.

### SYN Scan (-sS) — Half-Open, Stealthy
```
Nmap ──SYN──▶ Target:22
Nmap ◀──SYN-ACK── Target    → Port OPEN
Nmap ──RST──▶ Target        (never completes handshake)
```
Never completes the TCP connection → doesn't appear in application logs.
Requires root/admin. Default scan type for nmap when run as root.

### UDP Scan (-sU)
UDP has no handshake. Nmap sends a UDP packet:
```
Nmap ──UDP──▶ Target:53
[no response]               → Open or Filtered (UDP is ambiguous)
Nmap ◀──ICMP Port Unreachable── Target  → Closed
```
UDP scanning is slow because closed ports require ICMP responses,
and firewalls often rate-limit ICMP.

---

## Essential Nmap Commands

### Host Discovery
```bash
# Ping sweep — find live hosts (ICMP echo)
nmap -sn 10.10.10.0/24

# Scan without ping (assume all hosts alive)
nmap -Pn 10.10.10.101

# ARP ping (more reliable on local network)
nmap -PR 10.10.10.0/24
```

### Port Scanning
```bash
# Default scan (top 1000 ports, SYN if root)
nmap 10.10.10.101

# All 65535 ports
nmap -p- 10.10.10.101

# Specific ports
nmap -p 21,22,80,443 10.10.10.101

# Port range
nmap -p 1-1000 10.10.10.101

# Top 100 most common ports
nmap --top-ports 100 10.10.10.101
```

### Service and Version Detection
```bash
# Service version detection (-sV) — what we used
nmap -sV 10.10.10.101

# Aggressive version detection (more probes, slower)
nmap -sV --version-intensity 9 10.10.10.101
```

### OS Detection
```bash
# OS fingerprinting
nmap -O 10.10.10.101

# Combined service + OS (requires root)
nmap -sV -O 10.10.10.101
```

### Script Scanning (NSE)
```bash
# Default scripts (safe, informational)
nmap -sC 10.10.10.101

# Specific script
nmap --script ftp-anon 10.10.10.101

# Script category
nmap --script vuln 10.10.10.101    # vulnerability detection
nmap --script auth 10.10.10.101    # authentication tests
nmap --script exploit 10.10.10.101 # exploitation (use carefully)

# Multiple scripts
nmap --script "ftp-* and not ftp-brute" 10.10.10.101
```

### Combined Scans
```bash
# -A: aggressive (OS + version + scripts + traceroute)
nmap -A 10.10.10.101

# Full recommended scan for lab
nmap -sV -sC -O -p- 10.10.10.101

# Quick version scan (what we used)
nmap -sV 10.10.10.101
```

---

## Output Formats

```bash
# Normal output (default, human readable)
nmap -sV 10.10.10.101

# Save to file — normal format
nmap -sV 10.10.10.101 -oN scan.txt

# Save to file — XML format (parseable by tools)
nmap -sV 10.10.10.101 -oX scan.xml

# Save to file — grepable format
nmap -sV 10.10.10.101 -oG scan.gnmap

# Save all formats simultaneously
nmap -sV 10.10.10.101 -oA scan_results
# Creates: scan_results.nmap, scan_results.xml, scan_results.gnmap

# Import XML into Metasploit
msf > db_import scan.xml
msf > hosts
msf > services
```

---

## Timing Templates

Nmap has 6 timing templates (-T0 to -T5):

| Template | Name | Use Case |
|---|---|---|
| -T0 | Paranoid | IDS evasion, very slow |
| -T1 | Sneaky | IDS evasion, slow |
| -T2 | Polite | Don't disrupt network |
| -T3 | Normal | Default |
| -T4 | Aggressive | Fast, reliable network |
| -T5 | Insane | Very fast, may miss ports |

```bash
nmap -T4 10.10.10.101    # Fast scan, good for lab
nmap -T1 10.10.10.101    # Slow/stealthy
```

In our lab: no IDS concern, use -T4 for speed.
In real engagements: -T2 or -T3 to avoid detection.

---

## Reading Nmap Output — Our Scan Explained

```bash
nmap -sV 10.10.10.101
```

Output we got:
```
PORT     STATE SERVICE     VERSION
21/tcp   open  ftp         vsftpd 2.3.4       ← version! backdoor known
22/tcp   open  ssh         OpenSSH 4.7p1      ← very old version
23/tcp   open  telnet      Linux telnetd      ← plaintext remote shell
80/tcp   open  http        Apache httpd 2.2.8 ← vulnerable web server
139/tcp  open  netbios-ssn Samba smbd 3.X     ← SMB share
445/tcp  open  netbios-ssn Samba smbd 3.X
1524/tcp open  bindshell   Metasploitable root shell  ← ROOT SHELL OPEN
3306/tcp open  mysql       MySQL 5.0.51a      ← database exposed
5432/tcp open  postgresql  PostgreSQL 8.3     ← database exposed
5900/tcp open  vnc         VNC (protocol 3.3) ← desktop access
6667/tcp open  irc         UnrealIRCd         ← backdoor version
```

### How We Used This Information
Each line is a potential attack vector:

| Port | Version Detected | Action Taken |
|---|---|---|
| 21 vsftpd 2.3.4 | Known backdoor | Exploit #2: vsftpd |
| 1524 bindshell | Root shell open | Exploit #1: netcat |
| 6667 UnrealIRCd | Known backdoor | Exploit #3: UnrealIRCd |
| 22 OpenSSH 4.7p1 | Outdated, but not exploited | Left for later |
| 3306 MySQL 5.0.51a | Old version | Left for later |

**The nmap output IS the attack plan.** Every version number is a
search query: `vsftpd 2.3.4 exploit` → CVE/Metasploit module.

---

## NSE — Nmap Scripting Engine

Nmap includes hundreds of Lua scripts for advanced scanning:

```bash
# List all scripts
ls /usr/share/nmap/scripts/

# FTP scripts
nmap --script ftp-anon 10.10.10.101          # Check anonymous FTP
nmap --script ftp-bounce 10.10.10.101        # FTP bounce attack
nmap --script ftp-vsftpd-backdoor 10.10.10.101  # Check vsftpd backdoor!

# SMB scripts
nmap --script smb-vuln-ms17-010 10.10.10.101  # EternalBlue check
nmap --script smb-enum-shares 10.10.10.101    # List shares
nmap --script smb-enum-users 10.10.10.101     # List users

# HTTP scripts
nmap --script http-title 10.10.10.101          # Get page title
nmap --script http-robots.txt 10.10.10.101     # Get robots.txt

# Vulnerability scanning
nmap --script vuln 10.10.10.101               # Run all vuln scripts
```

### The vsftpd Backdoor Script
```bash
nmap --script ftp-vsftpd-backdoor 10.10.10.101
# Output: VULNERABLE: vsFTPd version 2.3.4 backdoor
```

This would have confirmed the vulnerability before we even opened Metasploit.

---

## Nmap in the Pentest Workflow

```
Phase 1: Discovery — Who is alive?
  nmap -sn 10.10.10.0/24

Phase 2: Port scan — What ports are open?
  nmap -p- -T4 10.10.10.101

Phase 3: Service detection — What is running?
  nmap -sV -p [open ports] 10.10.10.101

Phase 4: Script scan — What vulnerabilities exist?
  nmap -sC --script vuln -p [open ports] 10.10.10.101

Phase 5: OS detection — What OS is it?
  nmap -O 10.10.10.101

→ Feed results to Metasploit, choose exploits
```

---

## Stealth Considerations

In real engagements, nmap scans generate logs. Defensive tools detect:
- Large number of SYN packets from one IP
- Port scanning patterns
- OS fingerprinting probes

Evasion techniques (educational):
```bash
# Decoy scan (spoof source IPs)
nmap -D 192.168.1.5,192.168.1.6,ME 10.10.10.101

# Fragment packets (evade simple packet inspection)
nmap -f 10.10.10.101

# Randomize port order
nmap --randomize-hosts 10.10.10.0/24

# Slow timing (avoid rate-based detection)
nmap -T1 10.10.10.101
```

In our lab: no need for evasion. In real engagements: stealth matters.

---

## Key Takeaways

- Nmap is always the first step — you cannot exploit what you don't know exists
- `-sV` (version detection) is the most important flag: versions reveal CVEs
- SYN scan (-sS) is stealthy, TCP connect (-sT) is noisy
- NSE scripts extend nmap from scanner to vulnerability detector
- Save output with `-oA` — you'll want to reference it later
- The version string `vsftpd 2.3.4` directly maps to a Metasploit module
- In real engagements, scan timing and stealth matter — in lab, use -T4
