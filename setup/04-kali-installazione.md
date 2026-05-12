# 04 — Installazione Kali Linux (Attacker VM)

## Obiettivo
Installare Kali Linux come VM attacker sulla rete LAB (VMnet2/10.10.10.0/24).
Questa VM è il punto di partenza per tutte le attività Red Team del lab.

## Configurazione VM

| Parametro   | Valore                          |
| ----------- | ------------------------------- |
| Nome VMware | Kali-Attacker                   |
| OS          | Kali Linux 2026.1 (base Debian) |
| vCPU        | 2 core                          |
| RAM         | 4096 MB                         |
| Disco       | 100 GB (single file, SCSI)      |
| Rete        | VMnet2 (LAB — 10.10.10.0/24)    |
| Percorso    | D:\VM\MACHINES\KALI-ATTACKER\   |

## Installazione — Parametri Scelti

### Rete
Hostname e dominio configurati durante il wizard.

![Hostname](screenshots/kali-01-hostname.png)

![Dominio](screenshots/kali-02-dominio.png)

### Utente
Kali moderno non usa root come utente principale.
Creato utente standard con sudo.

![Nome completo](screenshots/kali-03-utente-nome-completo.png)

![Username](screenshots/kali-04-utente-username.png)

### Partizionamento
Schema semplice — tutto in una partizione, adatto a VM lab.

![Metodo guidato](screenshots/kali-05-partizionamento-metodo.png)

![Schema partizione](screenshots/kali-06-partizionamento-schema.png)

Risultato: 103.1 GB ext4 (/) + 4.3 GB swap.

![Anteprima partizioni](screenshots/kali-07-partizionamento-anteprima.png)

![Conferma scrittura](screenshots/kali-08-partizionamento-conferma.png)

### Software Installato

| Componente | Stato |
|---|---|
| Desktop XFCE | ✅ |
| Top 10 tools | ✅ |
| Default tools | ✅ |

![Selezione software](screenshots/kali-09-selezione-software.png)

### GRUB e Fine Installazione

![GRUB bootloader](screenshots/kali-10-grub-bootloader.png)

![Installazione completata](screenshots/kali-11-installazione-completata.png)

## Primo Avvio

![Login screen](screenshots/kali-12-login-screen.jpg)

## Verifica Rete — Risultati

```bash
ip addr show
# eth0: 10.10.10.100/24 — IP assegnato da DHCP pfSense ✅

ip route show
# default via 10.10.10.254 dev eth0 proto dhcp src 10.10.10.100 metric 100

ping 10.10.10.254 -c 4
# 4 pacchetti inviati, 4 ricevuti, 0% packet loss ✅
```

![Terminale IP e ping](screenshots/kali-13-terminale-ip-ping.png)

## Verifica Internet — Post Fix DNS

```bash
ping 8.8.8.8 -c 2
# 2/2 pacchetti ricevuti ✅ (connettività IP)

ping google.com -c 2
# 2/2 pacchetti ricevuti ✅ (risoluzione DNS funzionante)
```

![Internet e DNS verificati](screenshots/kali-14-internet-ok-post-dns-fix.png)

## Mappa Rete Post-Installazione

| VM | IP | Rete | Ruolo |
|---|---|---|---|
| pfSense (LAN) | 192.168.233.254 | VMnet1 | Firewall/Gateway mgmt |
| pfSense (LAB) | 10.10.10.254 | VMnet2 | Gateway rete lab |
| Kali Linux | 10.10.10.100 | VMnet2 | Attacker |
| Metasploitable2 | (da configurare) | VMnet2 | Target |

## Snapshot
- `00-kali-pre-installazione` — VM creata, ISO montata
- `01-kali-installato-desktop-ok` — Kali operativo, IP confermato
- `02-kali-prima-exploitation-bindshell`
- `03-kali-vsftpd-exploit-metasploit`
- `04-kali-internet-ok-post-dns-fix`

## Lezioni Imparate
- Il disco da 100GB viene mostrato come 107.4GB da VMware
  (differenza tra GB decimali e binari — normale)
- Kali assegna automaticamente IP .100 come primo lease DHCP
- Il ping al gateway pfSense (10.10.10.254) è il test minimo per
  confermare che la rete lab funziona correttamente
- Verificare sempre anche il ping a 8.8.8.8 (connettività) e a
  google.com (DNS) — la rete può funzionare senza internet se
  pfSense non ha la regola NAT corretta