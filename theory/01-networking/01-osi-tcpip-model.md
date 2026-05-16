# 01 — Networking Fundamentals: OSI and TCP/IP Models

## Why This Matters
Everything in cybersecurity happens over networks. Every exploit we
ran, every packet Wireshark captured, every alert Wazuh generated —
all of it follows the rules defined by these models. Understanding
them means understanding *why* attacks work, not just *how* to run them.

---

## The OSI Model — 7 Layers

The **Open Systems Interconnection** model (1984) describes how data
travels from one computer to another, breaking the process into
7 distinct layers.

```
Layer 7 — Application    ← What the user/app sees (HTTP, FTP, SSH, IRC)
Layer 6 — Presentation   ← Data format, encryption (SSL/TLS, encoding)
Layer 5 — Session        ← Managing connections (open, maintain, close)
Layer 4 — Transport      ← Reliability, ports (TCP, UDP)
Layer 3 — Network        ← Routing between networks (IP addresses)
Layer 2 — Data Link      ← Local delivery (MAC addresses, Ethernet)
Layer 1 — Physical       ← Actual cables, signals, bits
```

**Key rule:** Data travels DOWN the stack when sending, UP when receiving.
Each layer adds its own header (encapsulation) when sending and removes
it (decapsulation) when receiving.

### Lab Connection
| What we did | OSI Layer |
|---|---|
| Wireshark capturing raw frames | Layer 2 |
| IP addresses (10.10.10.x) | Layer 3 |
| TCP ports (21, 4444, 6667) | Layer 4 |
| FTP, SSH, IRC protocols | Layer 7 |
| TLS on Wazuh dashboard (port 443) | Layer 6 |

---

## The TCP/IP Model — 4 Layers (Practical)

The real-world implementation collapses the OSI model into 4 layers:

```
Application Layer    → OSI layers 5+6+7 (HTTP, FTP, DNS, SSH, IRC)
Transport Layer      → OSI layer 4     (TCP, UDP)
Internet Layer       → OSI layer 3     (IP, ICMP)
Network Access Layer → OSI layers 1+2  (Ethernet, MAC)
```

This is what actually runs on every device. When you hear "Layer 3
switch" or "Layer 7 firewall" — those are OSI references.

---

## TCP vs UDP

| Feature | TCP | UDP |
|---|---|---|
| Connection | Yes (3-way handshake) | No |
| Reliability | Guaranteed delivery | Best effort |
| Order | Packets arrive in order | May arrive out of order |
| Speed | Slower | Faster |
| Use cases | FTP, SSH, HTTP, Wazuh | DNS, streaming, VoIP |

**In our lab:** Every exploit used TCP — reliability matters when
you're sending shell commands. Wazuh agent uses TCP port 1514.

---

## The TCP 3-Way Handshake

Before any TCP data exchange, two hosts perform this ritual:

```
Client (Kali)              Server (Metasploitable)
      │                           │
      │──── SYN ─────────────────▶│  "I want to connect, seq=X"
      │                           │
      │◀─── SYN-ACK ──────────────│  "OK, seq=Y, ack=X+1"
      │                           │
      │──── ACK ─────────────────▶│  "Got it, ack=Y+1"
      │                           │
      │    [data exchange]        │
      │                           │
      │──── FIN ─────────────────▶│  "Done, closing"
      │◀─── FIN-ACK ──────────────│  "Confirmed"
```

**SYN** = Synchronize (start connection)
**ACK** = Acknowledge (confirm receipt)
**FIN** = Finish (close connection)
**RST** = Reset (abort connection immediately — red in Wireshark)

### What we saw in Wireshark
- Packets 5-7 (tcp.port==21): SYN → SYN-ACK → ACK opening FTP
- Packets 57-59 (red): RST — Kali aborting the FTP connection
  after triggering the vsftpd backdoor
- Packets 22-23 (tcp.port==4444): new SYN → SYN-ACK opening
  the reverse shell from Metasploitable back to Kali

---

## IP Addresses and Subnets

An **IP address** (IPv4) is a 32-bit number written as 4 octets:
```
10.10.10.100 = 00001010.00001010.00001010.01100100
```

A **subnet mask** defines which part is the network and which is
the host:
```
10.10.10.0/24  means:
  - Network: 10.10.10.0  (first 24 bits)
  - Hosts:   10.10.10.1 → 10.10.10.254
  - Broadcast: 10.10.10.255
```

### Our Lab Network
```
VMnet2: 10.10.10.0/24
├── 10.10.10.1   → 10.10.10.99   (reserved/unused)
├── 10.10.10.100 → 10.10.10.200  (DHCP range — our VMs)
│   ├── 10.10.10.100  Kali (attacker)
│   ├── 10.10.10.101  Metasploitable2 (target)
│   └── 10.10.10.105  Ubuntu/Wazuh (SIEM)
└── 10.10.10.254  pfSense LAB (gateway)
```

pfSense acts as the **router** between VMnet2 (lab) and VMnet8
(NAT/internet). Without pfSense, the lab VMs couldn't reach the internet.

---

## Ports — The "Doors" of a Computer

A port is a number (0-65535) that identifies which service/application
should receive a network packet. Think of IP address as a building
address and port as an apartment number.

### Well-Known Ports (0-1023) — Requires root/admin to open
| Port | Protocol | Service |
|---|---|---|
| 21 | TCP | FTP — File Transfer Protocol |
| 22 | TCP | SSH — Secure Shell |
| 23 | TCP | Telnet (unencrypted remote shell) |
| 25 | TCP | SMTP — Email sending |
| 53 | TCP/UDP | DNS — Domain Name System |
| 80 | TCP | HTTP — Web (unencrypted) |
| 443 | TCP | HTTPS — Web (encrypted) |
| 445 | TCP | SMB — Windows file sharing |
| 389 | TCP | LDAP — Directory services |
| 636 | TCP | LDAPS — LDAP over SSL |
| 88 | TCP/UDP | Kerberos — Authentication |
| 3389 | TCP | RDP — Remote Desktop Protocol |

### Ports We Used in This Lab
| Port | Service | Context |
|---|---|---|
| 21 | FTP vsftpd 2.3.4 | vsftpd backdoor exploit |
| 1524 | Bindshell | Direct root shell — no auth |
| 4444 | Metasploit listener | Reverse shell receiver (our choice) |
| 6200 | vsftpd backdoor | Shell spawned by ":)" trigger |
| 6667 | IRC UnrealIRCd | UnrealIRCd backdoor exploit |
| 1514 | Wazuh agent | Agent → Manager communication |
| 9200 | OpenSearch | Wazuh Indexer HTTP API |
| 443 | HTTPS | Wazuh Dashboard |

---

## ICMP — The "Ping" Protocol

**ICMP (Internet Control Message Protocol)** is used for network
diagnostics, not data transfer. `ping` uses ICMP Echo Request/Reply.

```
Kali$ ping 10.10.10.101
ICMP Echo Request  →  Metasploitable
ICMP Echo Reply    ←  Metasploitable
```

**Why it matters:** An active host responds to ping. If ping fails,
either the host is down, or a firewall blocks ICMP.
`nmap -sn` (ping sweep) uses this to discover live hosts.

---

## DNS — Translating Names to IPs

**DNS (Domain Name System)** converts human-readable names to IP addresses.

```
Browser asks: "What is google.com?"
DNS server responds: "172.217.23.174"
Browser connects to: 172.217.23.174
```

In our lab, pfSense acts as DNS server (10.10.10.254) for all lab VMs.
When Ubuntu pinged `google.com` successfully, pfSense resolved the
name and forwarded the traffic through NAT to the internet.

---

## Firewalls — Traffic Control

A firewall inspects packets and allows/blocks them based on rules.

### pfSense Rules in Our Lab
```
Rule 1 (LAB interface):
  Source: LAB subnets (10.10.10.0/24)
  Destination: any
  Action: PASS
  → All lab VMs can send traffic anywhere
```

Without this rule, Kali couldn't reach Metasploitable even though
they're on the same VMnet2.

### NAT — Network Address Translation
pfSense uses NAT to let lab VMs (10.10.10.x — private addresses)
communicate with the internet using the host's public IP:

```
Kali (10.10.10.100) → pfSense → Internet
                    ↑ NAT translates private IP to VMware's NAT IP
```

This also means: **Metasploitable cannot be reached from the internet**.
It's completely isolated — safe for our lab.

---

## Key Takeaways

- The OSI model explains what happens at each step of a network
  communication — attacks target specific layers
- TCP is reliable (handshake + acknowledgment), UDP is fast (fire-and-forget)
- Every service listens on a port — nmap scans ports to discover services
- Subnetting defines network boundaries — pfSense separates our lab
  from the host machine
- ICMP, DNS, and NAT are infrastructure protocols that make everything work
- Understanding these layers explains *why* Metasploit needs LHOST
  (reverse shell must know where to call back), why FTP is dangerous
  (Layer 7 plaintext), and why pfSense is necessary (Layer 3 routing)
