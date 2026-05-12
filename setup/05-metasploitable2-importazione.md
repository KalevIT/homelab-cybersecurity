# 05 — Importazione Metasploitable2 (Target VM)

## Obiettivo
Importare Metasploitable2 come prima VM target del lab.
VM Linux intenzionalmente vulnerabile per la pratica
di penetration testing.

## Cos'è Metasploitable2
- Basata su Ubuntu 8.04
- Decine di servizi vulnerabili preconfigurati
- Progettata specificamente per essere compromessa in lab
- Credenziali default pubbliche: msfadmin/msfadmin

## ⚠️ Regola di Sicurezza Assoluta
**Mai collegare questa VM a NAT o Bridged.**
Solo VMnet2 (LAB isolata). Lo avverte anche la VM stessa
al boot: "Never expose this VM to an untrusted network!"

## Configurazione VM

| Parametro | Valore |
|---|---|
| Importazione | File → Open → Metasploitable2.vmx |
| RAM | 512 MB |
| CPU | 1 core |
| Disco | 8 GB |
| Rete | VMnet2 (LAB) ✅ |

![VMware settings](screenshots/meta-01-vmware-settings.png)

## Avvio

La VM non ha interfaccia grafica — solo terminale testuale.
Al boot mostra il warning di sicurezza e le credenziali.

![Boot screen](screenshots/meta-02-boot-screen.png)

## Verifica Rete da Metasploitable

```bash
ip addr show
# eth0: 10.10.10.101/24 — IP da DHCP pfSense ✅

ping 10.10.10.254 -c 4
# 4/4 pacchetti ricevuti, 0% loss ✅
```

![IP e ping da Metasploitable](screenshots/meta-03-ip-ping-pfSense.png)

## Prima Ricognizione da Kali — nmap -sV

Primo comando offensivo reale: service version scan sul target.

```bash
ping 10.10.10.101 -c 4   # verifica raggiungibilità
nmap -sV 10.10.10.101    # enumeration servizi e versioni
```

![Ping e nmap da Kali](screenshots/meta-04-kali-ping-nmap.png)

## Risultati nmap — Servizi Rilevati

| Porta | Servizio | Versione | Note |
|---|---|---|---|
| 21 | FTP | vsftpd 2.3.4 | Backdoor nota |
| 22 | SSH | OpenSSH 4.7p1 | Versione datata |
| 23 | Telnet | Linux telnetd | Traffico in chiaro |
| 80 | HTTP | Apache 2.2.8 | Web server vulnerabile |
| 139/445 | SMB | Samba 3.X-4.X | Condivisioni di rete |
| 1524 | Bindshell | **Root shell aperta** | Accesso root diretto |
| 3306 | MySQL | 5.0.51a | Database esposto |
| 5432 | PostgreSQL | 8.3.0-8.3.7 | Database esposto |
| 5900 | VNC | Protocol 3.3 | Desktop remoto |
| 6667 | IRC | UnrealIRCd | Versione con backdoor |

**977 porte chiuse** — solo queste aperte e vulnerabili.

## Mappa Rete Completa — Fine Fase 1

| VM | IP | VMnet | Ruolo |
|---|---|---|---|
| Host Windows | 192.168.233.1 | VMnet1 | Host fisico |
| pfSense LAN | 192.168.233.254 | VMnet1 | Firewall mgmt |
| pfSense LAB | 10.10.10.254 | VMnet2 | Gateway lab |
| Kali Linux | 10.10.10.100 | VMnet2 | Attacker |
| Metasploitable2 | 10.10.10.101 | VMnet2 | Target |

## Snapshot
- `00-metasploitable2-pre-avvio`
- `01-metasploitable2-avviato-rete-ok`
- `02-meta-rete-ok-pre-exploit-unrealircd`

## Lezioni Imparate
- VM preconfigurate (.vmx) si importano senza installazione
- Verificare SEMPRE l'adattatore prima di avviare VM vulnerabili
- nmap -sV identifica servizi E versioni — fondamentale per
  trovare CVE (vulnerabilità note) associate a ogni versione
- La porta 1524 aperta su Metasploitable è un esempio estremo:
  una root shell esposta in rete è la vulnerabilità peggiore possibile
