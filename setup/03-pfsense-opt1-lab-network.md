# 03 — Configurazione Interfaccia LAB (OPT1)

## Obiettivo
Abilitare em2 come interfaccia LAB su pfSense collegata a VMnet2
(10.10.10.0/24). Questa rete ospiterà tutte le VM attacker e target.

## Step 1 — Interface Assignments
Aggiunta em2 dalla pagina Interfaces > Assignments.

![Interface Assignments](screenshots/05-interface-assignments-em2.png)

## Step 2 — Configurazione OPT1/LAB
Interfaccia abilitata, rinominata LAB, IP statico 10.10.10.1/24.

![Configurazione OPT1](screenshots/06-opt1-lab-configurazione.png)

## Step 3 — DHCP Server LAB
DHCP abilitato per range 10.10.10.100 - 10.10.10.200.

![DHCP Server LAB](screenshots/07-dhcp-server-lab.png)

## Step 4 — Regola Firewall
Regola Pass su interfaccia LAB: sorgente LAB subnets,
destinazione any, protocollo any.

![Firewall Rule LAB](screenshots/08-firewall-rule-lab.png)

## Risultato Finale — Dashboard
Tutte e 3 le interfacce attive e operative.

![Dashboard 3 interfacce](screenshots/04-dashboard-3-interfacce-attive.png)

## Configurazione Completa

| Interfaccia | Fisica | IP | VMnet | Scopo |
|---|---|---|---|---|
| WAN | em0 | 192.168.47.128/24 (DHCP) | VMnet8 | Internet |
| LAN | em1 | 192.168.233.254/24 | VMnet1 | Gestione |
| LAB | em2 | 10.10.10.1/24 | VMnet2 | Rete VM lab |

## DHCP LAB
- Range: 10.10.10.100 → 10.10.10.200
- Gateway: 10.10.10.1
- DNS: 10.10.10.1

## Snapshot
- `04-pfsense-opt1-lab-configurata`

## Note
Banner giallo nel DHCP: ISC DHCP sarà rimosso in versioni future
di pfSense. Per il lab attuale non è un problema — ignorabile.