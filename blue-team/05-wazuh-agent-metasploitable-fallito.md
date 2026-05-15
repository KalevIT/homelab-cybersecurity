# 05 — Wazuh Agent su Metasploitable2 — Tentativo Fallito

## Categoria
Blue Team / SIEM / Agent / Compatibilità / Legacy Systems

## Obiettivo
Tentare l'installazione del Wazuh Agent su Metasploitable2 per
monitorare la VM target dal punto di vista della vittima durante
gli exploit. Documentare l'esito e le cause tecniche del fallimento.

## Ambiente

| Ruolo | VM | IP |
|---|---|---|
| SIEM | Ubuntu BlueTeam | 10.10.10.105 |
| Target | Metasploitable2 | 10.10.10.101 |

## Specifiche Metasploitable2

| Parametro | Valore |
|---|---|
| OS | Ubuntu 8.04 LTS (Hardy Heron) — 2008 |
| Kernel | 2.6.24-16-server |
| Architettura | i686 / i386 (32-bit) |
| OpenSSL | 0.9.8g (2007) |
| glibc | 2.7 |
| Scopo | VM intenzionalmente vulnerabile per penetration testing |

## Accesso alla VM

Metasploitable2 non ha interfaccia grafica — solo console testuale.
Credenziali: `msfadmin / msfadmin`

**Nota SSH:** Meta ha SSH sulla porta 22 ma usa algoritmi
di key exchange deprecati. Da Kali moderna è necessario forzarli:

```bash
ssh -o KexAlgorithms=+diffie-hellman-group1-sha1 \
    -o HostKeyAlgorithms=+ssh-rsa \
    msfadmin@10.10.10.101
```

Per questo tentativo si è usata direttamente la console VMware.

## Procedura Tentata

### Step 1 — Verifica architettura

```bash
dpkg --print-architecture
# i386
```

Confermato: sistema 32-bit, richiede pacchetto `i386`.

### Step 2 — Comando generato dalla Dashboard

La Dashboard Wazuh ha generato il comando per `amd64` (errore):

```bash
wget https://packages.wazuh.com/4.x/apt/pool/main/w/wazuh-agent/\
wazuh-agent_4.14.5-1_amd64.deb \
&& sudo WAZUH_MANAGER='10.10.10.105' \
   WAZUH_AGENT_NAME='metasploitable-target' \
   dpkg -i ./wazuh-agent_4.14.5-1_amd64.deb
```

### Step 3 — Errore durante il download

```
Resolving packages.wazuh.com... 108.138.192.47, ...
Connecting to packages.wazuh.com|108.138.192.47|:443... connected.
OpenSSL: error:14077410:SSL routines:SSL23_GET_SERVER_HELLO:sslv3 alert handshake failure
Unable to establish SSL connection.
```

Il download fallisce ancora prima di arrivare al pacchetto.

![Errore SSL su Metasploitable](screenshots/wazuh-agent-meta-01-ssl-error.png)

## Analisi — Cause del Fallimento

### Causa 1 — OpenSSL troppo vecchio (bloccante)

| Sistema | Versione OpenSSL | TLS supportato |
|---|---|---|
| Metasploitable2 | 0.9.8g (2007) | SSLv3, TLS 1.0 |
| packages.wazuh.com | Moderno | TLS 1.2+ obbligatorio |

Il server Wazuh richiede TLS 1.2 minimo. OpenSSL 0.9.8 non lo
supporta → handshake fallisce → download impossibile.

### Causa 2 — Architettura sbagliata nel comando

La Dashboard ha generato il comando per `amd64` ma il sistema
è `i386`. Anche correggendo il pacchetto a `i386`, il problema
TLS bloccherebbe ugualmente il download.

### Causa 3 — Dipendenze di sistema incompatibili (bloccante)

Anche scaricando il pacchetto con un metodo alternativo
(es. trasferimento via SCP da Kali), l'installazione fallirebbe:

| Dipendenza | Richiesta da Wazuh | Disponibile su Ubuntu 8.04 |
|---|---|---|
| glibc | ≥ 2.17 | 2.7 ❌ |
| OpenSSL | ≥ 1.0.2 | 0.9.8 ❌ |
| systemd | richiesto | non presente (usa SysV init) ❌ |

Metasploitable2 è basata su Ubuntu 8.04 del 2008 — è
fondamentalmente incompatibile con qualsiasi software moderno.

## Esito

| Tentativo | Risultato |
|---|---|
| Download pacchetto | ❌ SSL handshake failure |
| Installazione dpkg | ❌ non raggiunta |
| Agent registrato su Wazuh | ❌ impossibile |

**Conclusione:** L'installazione del Wazuh Agent su Metasploitable2
è tecnicamente impossibile con Wazuh 4.x. Non esiste una versione
di Wazuh Agent compatibile con Ubuntu 8.04 / glibc 2.7.

## Alternativa per Monitorare la Vittima

In un lab più avanzato, si userebbe una VM target moderna
(es. Metasploitable3, basata su Ubuntu 14.04/Windows Server)
che supporta agent moderni. In alternativa si può usare:

- **Wireshark su Kali**: cattura il traffico di rete dell'attacco
- **tcpdump su Kali**: analisi del traffico a riga di comando
- **pfSense logs**: il firewall vede tutto il traffico tra le VM

## Snapshot
Nessuno — nessuna modifica persistente alla VM.

## Lezioni Imparate
- Metasploitable2 è intenzionalmente vecchia e vulnerabile —
  questa stessa caratteristica la rende incompatibile con
  software di sicurezza moderni come Wazuh
- La Dashboard Wazuh genera il pacchetto per l'architettura
  selezionata nel form — selezionare sempre `i386` per sistemi
  32-bit, non `amd64`
- OpenSSL 0.9.8 (2007) non supporta TLS 1.2: qualsiasi connessione
  HTTPS moderna fallisce — questo è esattamente il tipo di
  vulnerabilità che rende Metasploitable2 un target facile
- Nel mondo reale i sistemi legacy sono un rischio enorme:
  non possono essere monitorati da agent moderni ma sono
  ugualmente esposti agli attacchi
- La copertura del SIEM ha sempre dei "punti ciechi" sui sistemi
  troppo vecchi — il Blue Team deve esserne consapevole
