# 03 — Wireshark: Packet Analysis Guide

## What Is Wireshark

**Wireshark** is the world's most widely used network protocol analyzer.
It captures packets in real time and lets you inspect them at every level —
from raw bytes to decoded application data.

In our lab: used to capture and analyze the vsftpd exploit traffic,
seeing the backdoor trigger and reverse shell at the packet level.

Version used: **Wireshark 4.6.4** (on Kali Linux 2026.1).

---

## How Wireshark Works

Wireshark puts the network interface into **promiscuous mode** — it
captures ALL packets on the wire, not just those addressed to our machine.

```
Normal mode: NIC receives only packets addressed to itself
Promiscuous mode: NIC receives ALL packets on the segment
```

On a switched network (like our VMnet2), you only see:
- Your own traffic
- Broadcast traffic (ARP, DHCP)

On a hub or with port mirroring, you'd see everyone's traffic.
In our lab, Kali captures its own traffic to/from Metasploitable.

---

## Interface — Three Panels

```
┌─────────────────────────────────────────────────────┐
│  Packet List Panel (top)                            │
│  No. | Time | Source | Destination | Protocol | Info│
│  5   | 0.001| Kali   | Meta        | TCP      | SYN │
│  8   | 0.007| Meta   | Kali        | FTP      | 220 │
├─────────────────────────────────────────────────────┤
│  Packet Details Panel (middle)                      │
│  ▶ Frame 8: 86 bytes                                │
│  ▶ Ethernet II: src 00:0c:29... dst 00:0c:29...     │
│  ▶ Internet Protocol: src 10.10.10.101 dst .100     │
│  ▶ Transmission Control Protocol: port 21→39287     │
│  ▼ File Transfer Protocol: "220 vsFTPd 2.3.4\r\n"  │
├─────────────────────────────────────────────────────┤
│  Packet Bytes Panel (bottom)                        │
│  0000  00 0c 29 0e d0 91 00 0c...  ...).....E.      │
│  0050  32 30 30 20 76 73 46 54...  220 vsFTPd 2.3.4 │
└─────────────────────────────────────────────────────┘
```

Each panel shows the same packet at different levels of abstraction:
- **Top:** Summary (one line per packet)
- **Middle:** Decoded fields (click to expand each protocol layer)
- **Bottom:** Raw hexadecimal + ASCII representation

---

## Starting a Capture

### Method 1: GUI
1. Open Wireshark: `sudo wireshark`
2. Select interface: `eth0` (the active network interface)
3. Optional: Set capture filter in the filter box
4. Double-click `eth0` or click the blue shark fin → capture starts

### Method 2: Command Line (tshark)
```bash
# Capture on eth0, write to file
sudo tshark -i eth0 -w capture.pcap

# Capture with filter
sudo tshark -i eth0 -f "host 10.10.10.101" -w capture.pcap

# Read and display a capture file
tshark -r capture.pcap
```

### Saving and Opening Captures
```bash
# Save from GUI: File → Save As → .pcapng (default) or .pcap

# Open saved capture
wireshark capture.pcapng &
# or
tshark -r capture.pcapng
```

---

## Two Types of Filters

This is the most important thing to understand to avoid confusion:

### Capture Filter (BPF — Berkeley Packet Filter)
Set **before** starting a capture. Uses libpcap/BPF syntax.
Only packets matching the filter are captured — others are discarded.

```bash
# BPF syntax examples
host 10.10.10.101           # traffic to/from this IP
port 21                     # traffic on port 21
tcp                         # TCP traffic only
udp                         # UDP traffic only
host 10.10.10.101 and port 21  # to/from this IP on port 21
not port 22                 # exclude SSH traffic
net 10.10.10.0/24           # entire subnet
```

Set in the filter box on the Wireshark start screen (green when valid).
In our lab: `host 10.10.10.101` → captured only Meta traffic.

### Display Filter (Wireshark syntax)
Applied **during or after** capture. Hides/shows packets without
discarding them. All packets are still in the capture file.

```
# Wireshark display filter syntax examples
ip.addr == 10.10.10.101          # traffic to/from this IP
ip.src == 10.10.10.100           # traffic FROM Kali
ip.dst == 10.10.10.101           # traffic TO Meta
tcp.port == 21                   # FTP control channel
tcp.port == 4444                 # reverse shell
ftp                              # FTP protocol packets only
tcp.flags.syn == 1               # SYN packets only
tcp.flags.rst == 1               # RST packets (red in Wireshark)
tcp.flags == 0x002               # SYN only (hex flags)
frame contains "vsFTPd"          # packets containing this string
http.request.method == "GET"     # HTTP GET requests
dns                              # DNS queries/responses
arp                              # ARP packets
icmp                             # ping packets
```

**Common mistake:** Typing BPF syntax in the display filter bar → error (yellow/red background).
`host 10.10.10.101` is BPF. `ip.addr == 10.10.10.101` is display filter.

---

## Color Coding

Wireshark colors packets by protocol/type. Default scheme:

| Color | Meaning |
|---|---|
| Black text, red background | **TCP errors, RST, retransmissions** |
| Green | HTTP traffic |
| Light blue | DNS |
| Light purple | TCP traffic |
| Yellow | ARP, OSPF |
| Dark blue | SMB |
| White | TCP data (no protocol decoding) |

In our capture:
- **Red packets** = RST — Kali reset the FTP connection after backdoor trigger
- **White packets** = TCP stream between Kali:4444 and Meta (reverse shell data)

---

## Key Feature: Follow TCP Stream

The most powerful feature for analyzing exploits. Reconstructs the
entire TCP conversation in readable format.

```
Right-click any packet → Follow → TCP Stream
```

Shows:
- **Blue text** = data sent by the client (Kali)
- **Red text** = data sent by the server (Meta)

For our vsftpd capture, Follow TCP Stream on port 21 shows:
```
220 vsFTPd 2.3.4
USER sjUTx:)
331 Please specify the password.
PASS yrr67T
500 OOPS: priv_sock_get_result
```

For port 4444 (reverse shell), it shows the actual shell session:
```
whoami
root
id
uid=0(root) gid=0(root)
uname -a
Linux metasploitable...
```

**This is why FTP is dangerous:** Follow TCP Stream and you see everything.

---

## Useful Filters for Security Analysis

### Network Discovery
```
arp                          # See ARP requests (who's on the network)
icmp                         # See ping traffic
dns                          # See DNS queries
broadcast                    # See all broadcast traffic
```

### Attack Analysis
```
tcp.flags.syn == 1 and tcp.flags.ack == 0   # SYN scan
tcp.flags.rst == 1                           # RST packets (scan evidence)
ftp                                          # FTP commands (plaintext)
telnet                                       # Telnet traffic (plaintext)
irc                                          # IRC traffic (UnrealIRCd)
tcp.port == 4444                             # Metasploit default listener
tcp.port == 1524                             # Bindshell
tcp.port == 6667                             # IRC/UnrealIRCd
```

### Credential Hunting
```
ftp contains "PASS"          # FTP passwords
http.authbasic               # HTTP Basic Auth (base64 encoded)
frame contains "password"    # Generic password search
frame contains "login"       # Login forms
```

### Wazuh / SIEM Traffic
```
tcp.port == 1514             # Wazuh agent → manager
tcp.port == 9200             # Wazuh indexer (OpenSearch API)
tcp.port == 443              # Wazuh dashboard (HTTPS)
```

---

## Statistics Menu

Wireshark's Statistics menu provides high-level analysis:

```
Statistics → Protocol Hierarchy
→ Shows percentage of traffic by protocol
→ In our capture: TCP 100%, FTP 10%, data 90%

Statistics → Conversations
→ Lists all TCP/UDP conversations
→ Shows bytes exchanged per connection
→ Useful to spot the reverse shell conversation

Statistics → IO Graphs
→ Visualizes traffic over time
→ Spike = exploit activity

Statistics → Follow Stream
→ Alternative to right-click Follow TCP Stream
```

---

## Exporting Data

```
# Export specific packets to new file
File → Export Specified Packets → (apply filter first)

# Export packet bytes as raw binary
File → Export Packet Bytes

# Export objects (files transferred over HTTP, SMB)
File → Export Objects → HTTP (extracts downloaded files)
File → Export Objects → SMB (extracts SMB file transfers)
```

---

## tshark — Command-Line Wireshark

```bash
# Capture and display live
sudo tshark -i eth0

# Capture to file
sudo tshark -i eth0 -w output.pcap

# Read pcap and filter
tshark -r capture.pcap -Y "tcp.port == 21"

# Extract specific fields
tshark -r capture.pcap -Y "ftp" -T fields -e ftp.request.command -e ftp.request.arg

# Count packets by protocol
tshark -r capture.pcap -qz io,phs

# Follow TCP stream (command line)
tshark -r capture.pcap -q -z follow,tcp,ascii,0
```

---

## Our Lab Captures Explained

### Capture 1: vsftpd exploit (`vsftpd-exploit-capture.pcapng`)

```
Filter: host 10.10.10.101
Total packets: 61

Key sequences:
Packets 5-7:   TCP 3-way handshake (Kali→Meta:21)
Packet 8:      FTP banner "220 vsFTPd 2.3.4"
Packet 10:     USER sjUTx:)  ← BACKDOOR TRIGGER
Packet 12:     PASS yrr67T
Packet 56:     500 OOPS (backdoor activated)
Packets 57,59: TCP RST (FTP connection killed)
Packets 22-23: New TCP handshake (Meta→Kali:4444) = reverse shell opens
Packets 24-52: Data exchange (shell session)
Packets 53-54: FIN (shell closed after 'exit' command)
```

### Display Filters to Analyze Our Capture

```
# See full FTP conversation
ftp

# See 3-way handshake for FTP connection
tcp.port == 21 and tcp.flags.syn == 1

# See RST packets (backdoor trigger moment)
tcp.port == 21 and tcp.flags.rst == 1

# See reverse shell traffic
tcp.port == 4444

# See FTP banner (vsFTPd 2.3.4 version disclosure)
ftp contains "vsFTPd"
```

---

## Key Takeaways

- Wireshark captures everything that crosses the network interface —
  unencrypted protocols are completely exposed
- Three panels: list (summary), details (decoded), bytes (raw hex)
- Two filter types with different syntax: BPF (capture) vs display filter
- Follow TCP Stream reconstructs conversations — essential for seeing
  what commands were run in a shell session
- Unencrypted protocols (FTP, Telnet, HTTP, IRC) expose credentials
  and commands to anyone on the same network segment
- tshark provides the same functionality from the command line — useful
  for automation and headless servers
- Wireshark is the Blue Team's best friend for incident analysis —
  it shows exactly what happened, at the packet level
