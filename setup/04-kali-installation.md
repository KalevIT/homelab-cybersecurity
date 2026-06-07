# 04 — Kali Linux Installation (Attacker VM)

## Objective
Install Kali Linux as the attacker VM on the LAB network (VMnet2/10.10.10.0/24).
This VM is the starting point for all Red Team activities in the lab.

## VM Configuration

| Parameter   | Value                            |
| ----------- | -------------------------------- |
| VMware name | Kali-Attacker                    |
| OS          | Kali Linux 2026.1 (Debian-based) |
| vCPU        | 2 cores                          |
| RAM         | 8192 MB                          |
| Disk        | 100 GB (single file, SCSI)       |
| Network     | VMnet2 (LAB — 10.10.10.0/24)     |
| Path        | F:\VM\DISKS\KALI-ATTACKER\       |

## Installation — Chosen Parameters

### Network
Hostname and domain configured during the wizard.

![Hostname](screenshots/kali-01-hostname.png)

![Domain](screenshots/kali-02-dominio.png)

### User
Modern Kali does not use root as the main user.
Standard user created with sudo.

![Full name](screenshots/kali-03-utente-nome-completo.png)

![Username](screenshots/kali-04-utente-username.png)

### Partitioning
Simple scheme — everything in one partition, suitable for a lab VM.

![Guided method](screenshots/kali-05-partizionamento-metodo.png)

![Partition scheme](screenshots/kali-06-partizionamento-schema.png)

Result: 103.1 GB ext4 (/) + 4.3 GB swap.

![Partition preview](screenshots/kali-07-partizionamento-anteprima.png)

![Write confirmation](screenshots/kali-08-partizionamento-conferma.png)

### Installed Software

| Component | Status |
|---|---|
| XFCE Desktop | ✅ |
| Top 10 tools | ✅ |
| Default tools | ✅ |

![Software selection](screenshots/kali-09-selezione-software.png)

### GRUB and Installation End

![GRUB bootloader](screenshots/kali-10-grub-bootloader.png)

![Installation completed](screenshots/kali-11-installazione-completata.png)

## First Boot

![Login screen](screenshots/kali-12-login-screen.jpg)

## Network Verification — Results

```bash
ip addr show
# eth0: 10.10.10.100/24 — IP assigned by pfSense DHCP ✅

ip route show
# default via 10.10.10.254 dev eth0 proto dhcp src 10.10.10.100 metric 100

ping 10.10.10.254 -c 4
# 4 packets sent, 4 received, 0% packet loss ✅
```

![Terminal IP and ping](screenshots/kali-13-terminale-ip-ping.png)

## Internet Verification — Post DNS Fix

```bash
ping 8.8.8.8 -c 2
# 2/2 packets received ✅ (IP connectivity)

ping google.com -c 2
# 2/2 packets received ✅ (DNS resolution working)
```

![Internet and DNS verified](screenshots/kali-14-internet-ok-post-dns-fix.png)

## Network Map Post-Installation

| VM | IP | Network | Role |
|---|---|---|---|
| pfSense (LAN) | 192.168.233.254 | VMnet1 | Firewall/Gateway mgmt |
| pfSense (LAB) | 10.10.10.254 | VMnet2 | Lab network gateway |
| Kali Linux | 10.10.10.100 | VMnet2 | Attacker |
| Metasploitable2 | (to configure) | VMnet2 | Target |

## Snapshots
- `00-kali-pre-installazione` — VM created, ISO mounted
- `01-kali-installato-desktop-ok` — Kali operational, IP confirmed
- `02-kali-prima-exploitation-bindshell` — After bindshell exploit
- `03-kali-vsftpd-exploit-metasploit` — After vsftpd exploit
- `04-kali-internet-ok-post-dns-fix` — Internet verified, DNS working
- `05-kali-unrealircd-exploit` — After UnrealIRCd exploit
- `06-kali-wazuh-agent-attivo` — Wazuh Agent installed and active
- `07-kali-wireshark-cattura-vsftpd` — Wireshark capture of vsftpd exploit
- `08-kali-bloodhound-sharphound-installed` — BloodHound CE 9.1.0 + SharpHound installed, tools verified

## Lessons Learned
- 100 GB disk shows as 107.4 GB in VMware
  (difference between decimal and binary GB — normal)
- Kali automatically gets IP .100 as first DHCP lease
- Ping to pfSense gateway (10.10.10.254) is the minimum test to
  confirm the lab network is working correctly
- Always also verify ping to 8.8.8.8 (connectivity) and to
  google.com (DNS) — network can work without internet if
  pfSense does not have the correct NAT rule
