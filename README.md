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

| VMnet   | Tipo      | Subnet          | Scopo                    |
|---------|-----------|-----------------|--------------------------|
| VMnet1  | Host-Only | 192.168.233.0/24| Gestione pfSense         |
| VMnet2  | Host-Only | 10.10.10.0/24   | Rete lab isolata         |
| VMnet8  | NAT       | (auto VMware)   | WAN pfSense → Internet   |

## 🗺️ Fasi del Progetto

- [x] Fase 0 — Setup ambiente e documentazione
- [x] Fase 1 — Infrastruttura base (pfSense, Kali, Metasploitable2)
- [~] Fase 2 — Red Team base (Bindshell ✅, vsftpd ✅, UnrealIRCd 🔄)
- [~] Fase 3 — Blue Team base (Wazuh SIEM ✅, Wireshark 🔄)
- [ ] Fase 4 — Active Directory Lab

## 🖥️ VM del Lab

| VM | IP | VMnet | Ruolo |
|---|---|---|---|
| pfSense LAN | 192.168.233.254 | VMnet1 | Firewall mgmt |
| pfSense LAB | 10.10.10.254 | VMnet2 | Gateway lab |
| Kali Linux | 10.10.10.100 | VMnet2 | Attacker |
| Metasploitable2 | 10.10.10.101 | VMnet2 | Target |
| Ubuntu Server | 10.10.10.105 | VMnet2 | Blue Team / SIEM |

## 📁 Struttura Repository
- `/setup` — Installazione e configurazione VMs
- `/red-team` — Attività offensive documentate
- `/blue-team` — Attività difensive e monitoring
