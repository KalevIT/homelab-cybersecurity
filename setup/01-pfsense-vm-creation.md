# 01 — Creazione VM pfSense

## Obiettivo
Creare la VM pfSense-Firewall in VMware Workstation Pro con i 3 adattatori
di rete necessari per isolare il lab dall'host.

## Configurazione VM

| Parametro | Valore |
|---|---|
| Nome | pfSense-Firewall |
| OS Guest | FreeBSD 13 64-bit |
| vCPU | 1 core |
| RAM | 1024 MB |
| Disco | 20 GB (single file) |
| Percorso | D:\VM\MACHINES\PFSENSE\ |

## Adattatori di Rete

| Adattatore | VMnet | Ruolo |
|---|---|---|
| Adapter 1 | VMnet8 (NAT) | WAN |
| Adapter 2 | VMnet1 (Host-Only) | LAN/MGMT |
| Adapter 3 | VMnet2 (Host-Only) | OPT1/Lab |

![Configurazione adattatori VM](screenshots/pfsense-vm-adapters.png)

## Installazione — Netgate Installer v1.2

Installer ufficiale scaricato da shop.netgate.com. Richiede connessione
Internet durante l'installazione (usa VMnet8/NAT).

### Step WAN
Interfaccia em0 configurata in DHCP client su VMnet8.

![WAN Setup](screenshots/pfsense-netgate-installer-wan-setup.png)

### Step LAN
Interfaccia em1 configurata statica. IP impostato a 192.168.233.1/24
(poi corretto dopo il conflitto con l'host — vedi sezione Fix IP).

![LAN Setup](screenshots/pfsense-netgate-installer-lan-setup.png)

### Scelta pfSense CE
Nessun abbonamento Plus attivo — selezionato Install CE (gratuito).

![Install CE](screenshots/pfsense-install-ce-selection.png)

### Opzioni Installazione
File system: UFS — partizionamento: GPT.

![Installation Options](screenshots/pfsense-installation-options.png)

### Assegnazione Interfacce
LAN = em1 (attivo), WAN = em0 (attivo). em2 non visibile
perché senza traffico — configurata come OPT1 dopo.

![Interface Assignment](screenshots/pfsense-interface-assignment.png)

## Primo Avvio — Console pfSense

![Console primo avvio](screenshots/pfsense-first-boot-console.png)

| Interfaccia | NIC | IP |
|---|---|---|
| WAN | em0 | 192.168.47.128/24 (DHCP da VMnet8) |
| LAN | em1 | 192.168.233.254/24 (statico, post-fix) |
| OPT1 | em2 | Non ancora configurata |

**Versione installata:** pfSense 2.8.1-RELEASE amd64 (20251215-1731)

## Fix IP — Conflitto Host/pfSense

**Problema:** VMware assegna automaticamente 192.168.233.1 all'adattatore
host VMnet1. pfSense LAN era configurata sullo stesso IP — conflitto diretto.

**Soluzione:** LAN pfSense cambiata a 192.168.233.254/24 da console
(opzione 2 — Set interface IP address).

| Dispositivo | IP | Rete |
|---|---|---|
| Host Windows (VMnet1) | 192.168.233.1 | VMnet1 |
| pfSense LAN | 192.168.233.254 | VMnet1 |
| pfSense WAN | 192.168.47.128 | VMnet8 NAT |

**Lezione imparata:** Verificare sempre i conflitti IP tra adattatori
virtuali host VMware e VM prima di configurare le interfacce.

## Snapshot
- `00-pre-installazione` — VM creata, non avviata
- `01-pfsense-installato-e-avviato` — pfSense operativo, pre-webGUI
- `02-pfsense-lan-ip-corretto` — LAN corretta a .254, NordVPN disattivata