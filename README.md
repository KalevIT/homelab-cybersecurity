# 🔐 HomeLab Cybersecurity

Personal cybersecurity lab built on VMware Workstation Pro
on Windows 11 Pro. Progressive documentation by an absolute beginner.

## 🖥️ Host Specs
- **OS:** Windows 11 Pro
- **CPU:** AMD Ryzen 9 9900X (12c/24t)
- **RAM:** 32 GB
- **Lab storage:** 650 GB dedicated SSD
- **Hypervisor:** VMware Workstation Pro (latest version)

## 🌐 Network Architecture

| VMnet   | Type      | Subnet           | Purpose                |
|---------|-----------|------------------|------------------------|
| VMnet1  | Host-Only | 192.168.233.0/24 | pfSense management     |
| VMnet2  | Host-Only | 10.10.10.0/24    | Isolated lab network   |
| VMnet8  | NAT       | (VMware auto)    | pfSense WAN → Internet |

## 🖥️ Lab VMs

| VM                  | IP              | VMnet  | Role              |
| ------------------- | --------------- | ------ | ----------------- |
| pfSense LAN         | 192.168.233.254 | VMnet1 | Firewall mgmt     |
| pfSense LAB         | 10.10.10.254    | VMnet2 | Lab gateway       |
| Kali Linux          | 10.10.10.100    | VMnet2 | Attacker          |
| Metasploitable2     | 10.10.10.101    | VMnet2 | Target            |
| Ubuntu Server       | 10.10.10.105    | VMnet2 | Blue Team / SIEM  |
| Windows Server 2025 | 10.10.10.10     | VMnet2 | Domain Controller |

## 🗺️ Project Phases

### ✅ Phase 0 — Environment setup and documentation
- VMware Workstation Pro configured
- VMware Virtual Network Editor: VMnet1 (Host-Only/mgmt), VMnet2 (Host-Only/lab), VMnet8 (NAT)
- Isolated lab network (VMnet2 / 10.10.10.0/24)
- GitHub repository with documentation structure

### ✅ Phase 1 — Base infrastructure
- pfSense Firewall (WAN/LAN/LAB, DHCP, firewall rules)
- Kali Linux 2026.1 (attacker VM)
- Metasploitable2 (target VM)
- Ubuntu Server 26.04 LTS (blue team VM)

### ✅ Phase 2 — Base Red Team
- Bindshell port 1524 (netcat, direct root access)
- vsftpd 2.3.4 backdoor via Metasploit (reverse shell)
- UnrealIRCd 3.2.8.1 backdoor CVE-2010-2075 (supply chain attack)

### ✅ Phase 3 — Base Blue Team
- Wazuh 4.14.5 installed on Ubuntu (indexer + manager + dashboard)
- Post-reboot troubleshooting (opensearch.hosts, Filebeat, startup ordering)
- Wazuh Agent on Kali — active, 213 events recorded
- vsftpd exploit replay with Wazuh active — alerts visible in dashboard
- Agent on Metasploitable2 — ❌ incompatible (Ubuntu 8.04 / glibc 2.7 / OpenSSL 0.9.8)
- Wireshark — exploit traffic capture and analysis (vsftpd backdoor trigger visible)

### 🔄 Phase 4 — Active Directory Lab
- ✅ Windows Server 2025 installed — DC01 (10.10.10.10), static IP, VMnet2
- ✅ AD DS feature installed (`Install-WindowsFeature AD-Domain-Services`)
- ✅ Domain created — `homelab.local` / NetBIOS: `HOMELAB` / DomainMode: Windows2025
- ✅ AD verified — all services running (adws, dns, kdc, netlogon), DC01 is Global Catalog
- ✅ DNS configured — client: 127.0.0.1, forwarder: 8.8.8.8 / 1.1.1.1
- ✅ AD structure — OUs: LabUsers, LabAdmins, ServiceAccounts, Workstations
- ✅ AD users — alice.rossi, bob.verdi, charlie.bianchi, svc-sql (SPN), svc-backup (SPN), labadmin (Domain Admin)
- ⬜ Windows 11 CLIENT01 — domain join pending
- ⬜ AD attacks: BloodHound, Kerberoasting, AS-REP Roasting, Pass-the-Hash, DCSync

## 📁 Repository Structure

```
homelab-cybersecurity/
├── setup/          # VM installation and configuration
├── red-team/       # Documented offensive activities
├── blue-team/      # Defensive activities and monitoring
└── theory/         # Concepts and theory behind every tool and attack
    ├── 01-networking/      # OSI model, TCP/IP, protocols
    ├── 02-tools/           # nmap, Metasploit, Wireshark
    ├── 03-exploitation/    # Shell types, payloads
    └── 04-active-directory/ # AD fundamentals, Kerberos, attacks
```

## 📸 VMware Snapshots

| VM | Latest Snapshot |
|---|---|
| pfSense | `05-pfsense-lab-gateway-254-internet-ok` |
| Kali | `07-kali-wireshark-cattura-vsftpd` |
| Metasploitable2 | `03-meta-post-unrealircd` |
| Ubuntu Server | `05-ubuntu-filebeat-attivo-alert-ok` |
| Windows Server 2025 | `04-dc01-dns-forwarder-8888-internet-ok` |

### DC01 Snapshot History
| # | Name | Description |
|---|------|-------------|
| 00 | `00-winserver2025-installato-pre-ad` | Base OS, renamed DC01, static IP |
| 01 | `01-dc01-addomain-installed-homelab-local` | AD DS installed, domain created |
| 02 | `02-dc01-verified-dns-forwarder-ok` | AD verified, DNS client configured |
| 03 | `03-dc01-ad-users-created` | OUs and users created |
| 04 | `04-dc01-dns-forwarder-8888-internet-ok` | DNS forwarder fixed, internet working |

## ⚠️ Note on Theory Files

The `theory/` folder contains security research documentation including
descriptions of offensive techniques and tool syntax. Some files may
trigger antivirus false positives on Windows systems.

**Recommended:** Add the repository folder to your AV exclusions list.
All files are Markdown (.md) — plain text, not executable.
GitHub does not flag or restrict security research documentation.
