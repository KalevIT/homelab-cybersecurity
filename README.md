# 🔐 HomeLab Cybersecurity

Lab personale di cybersecurity costruito su VMware Workstation Pro 
su Windows 11 Pro. Documentazione progressiva di un principiante assoluto.

## 🖥️ Specifiche Host
- **OS:** Windows 11 Pro
- **CPU:** AMD Ryzen 9 9900X (12c/24t)
- **RAM:** 32 GB
- **Storage lab:** 650 GB SSD dedicato
- **Hypervisor:** VMware Workstation Pro (ultima versione)

## 🌐 Architettura di Rete

| VMnet   | Tipo      | Subnet           | Scopo                  |
|---------|-----------|------------------|------------------------|
| VMnet1  | Host-Only | 192.168.233.0/24 | Gestione pfSense       |
| VMnet2  | Host-Only | 10.10.10.0/24    | Rete lab isolata       |
| VMnet8  | NAT       | (auto VMware)    | WAN pfSense → Internet |

## 🖥️ VM del Lab

| VM | IP | VMnet | Ruolo |
|---|---|---|---|
| pfSense LAN | 192.168.233.254 | VMnet1 | Firewall mgmt |
| pfSense LAB | 10.10.10.254 | VMnet2 | Gateway lab |
| Kali Linux | 10.10.10.100 | VMnet2 | Attacker |
| Metasploitable2 | 10.10.10.101 | VMnet2 | Target |
| Ubuntu Server | 10.10.10.105 | VMnet2 | Blue Team / SIEM |

## 🗺️ Fasi del Progetto

### ✅ Fase 0 — Setup ambiente e documentazione
- VMware Workstation Pro configurato
- Rete lab isolata (VMnet2 / 10.10.10.0/24)
- Repository GitHub con struttura documentazione

### ✅ Fase 1 — Infrastruttura base
- pfSense Firewall (WAN/LAN/LAB, DHCP, regole firewall)
- Kali Linux 2026.1 (attacker VM)
- Metasploitable2 (target VM)
- Ubuntu Server 26.04 LTS (blue team VM)

### ✅ Fase 2 — Red Team base
- Bindshell porta 1524 (netcat, accesso root diretto)
- vsftpd 2.3.4 backdoor via Metasploit (reverse shell)
- UnrealIRCd 3.2.8.1 backdoor CVE-2010-2075 (supply chain attack)

### ✅ Fase 3 — Blue Team base
- Wazuh 4.14.5 installato su Ubuntu (indexer + manager + dashboard)
- Troubleshooting post-reboot (opensearch.hosts, Filebeat, startup ordering)
- Wazuh Agent su Kali — active, 213 eventi registrati
- Replay exploit vsftpd con Wazuh attivo — alert visibili in dashboard
- Agent su Metasploitable2 — ❌ incompatibile (Ubuntu 8.04 / glibc 2.7 / OpenSSL 0.9.8)
- Wireshark — analisi traffico exploit

### ⬜ Fase 4 — Active Directory Lab
- Windows Server + Active Directory
- VM Windows client
- Attacchi AD (Pass-the-Hash, Kerberoasting, ecc.)

## 📁 Struttura Repository

```
homelab-cybersecurity/
├── setup/          # Installazione e configurazione VMs
├── red-team/       # Attività offensive documentate
└── blue-team/      # Attività difensive e monitoring
```

## 📸 Snapshot VMware

| VM              | Ultimo Snapshot                          |
| --------------- | ---------------------------------------- |
| pfSense         | `05-pfsense-lab-gateway-254-internet-ok` |
| Kali            | `07-kali-wireshark-cattura-vsftpd`       |
| Metasploitable2 | `03-meta-post-unrealircd`                |
| Ubuntu Server   | `05-ubuntu-filebeat-attivo-alert-ok`     |
