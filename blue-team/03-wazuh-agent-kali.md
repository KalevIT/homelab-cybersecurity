# 03 — Wazuh Agent su Kali (Attacker VM)

## Categoria
Blue Team / SIEM / Agent / Monitoring

## Obiettivo
Installare il Wazuh Agent su Kali Linux (10.10.10.100) e registrarlo
al Manager su Ubuntu (10.10.10.105). Da questo momento Wazuh monitora
in tempo reale tutto quello che accade sulla macchina attacker.

## Ambiente

| Ruolo | VM | IP |
|---|---|---|
| SIEM / Manager | Ubuntu BlueTeam | 10.10.10.105 |
| Attacker / Agent | Kali Linux | 10.10.10.100 |

## Architettura Post-Installazione

```
Kali (10.10.10.100)              Ubuntu (10.10.10.105)
┌──────────────────┐             ┌────────────────────────┐
│  Wazuh Agent     │────────────▶│  Wazuh Manager         │
│  v4.14.5         │  porta 1514 │  raccoglie log/eventi  │
│  active/running  │             │                        │
└──────────────────┘             │  Dashboard: alert ✅   │
                                 └────────────────────────┘
```

## Procedura

### Step 1 — Generazione comando dalla Dashboard

Dashboard Wazuh (`https://10.10.10.105`) →
**Deploy new agent** → form compilato:

| Campo | Valore |
|---|---|
| OS | Linux — DEB amd64 |
| Server address | 10.10.10.105 |
| Agent name | kali-attacker |
| Group | default |

La dashboard genera automaticamente il comando di installazione.

![Deploy new agent form](screenshots/wazuh-agent-01-deploy-form.png)

### Step 2 — Installazione su Kali

Comando generato ed eseguito su Kali:

```bash
wget https://packages.wazuh.com/4.x/apt/pool/main/w/wazuh-agent/wazuh-agent_4.14.5-1_amd64.deb \
  && sudo WAZUH_MANAGER='10.10.10.105' \
     WAZUH_AGENT_NAME='kali-attacker' \
     dpkg -i ./wazuh-agent_4.14.5-1_amd64.deb
```

Output:
```
wazuh-agent_4.14.5-1_amd64.deb  100% [====] 12,60M 5,22MB/s in 2,4s
Selezionato il pacchetto wazuh-agent non precedentemente selezionato.
Configurazione di wazuh-agent (4.14.5-1)...
```

### Step 3 — Avvio e abilitazione servizio

```bash
sudo systemctl daemon-reload
sudo systemctl enable wazuh-agent
sudo systemctl start wazuh-agent
sudo systemctl status wazuh-agent
```

Output status:
```
● wazuh-agent.service - Wazuh agent
   Loaded: loaded (/usr/lib/systemd/system/wazuh-agent.service; enabled)
   Active: active (running) since Wed 2026-05-13 21:34:21 CEST; 6s ago

Processi attivi:
├─ wazuh-execd
├─ wazuh-agentd
├─ wazuh-syscheckd
├─ wazuh-logcollector
└─ wazuh-modulesd
```

![Kali - installazione e status agent](screenshots/wazuh-agent-02-kali-installazione.png)

## Verifica sulla Dashboard

Dopo ~30 secondi dall'avvio dell'agent, nella Dashboard:
**Endpoints → kali-attacker → active** ✅

| Campo | Valore |
|---|---|
| ID | 001 |
| Status | active |
| IP | 10.10.10.100 |
| Versione | Wazuh v4.14.5 |
| OS | Kali GNU/Linux 2026.1 |
| Cluster node | node01 |
| Registration | May 13, 2026 @ 21:34:16 |

![Dashboard - agent active](screenshots/wazuh-agent-03-dashboard-active.png)

## Analisi Iniziale Automatica

Wazuh ha eseguito automaticamente una **Security Configuration
Assessment (SCA)** su Kali usando il benchmark
**CIS Distribution Independent Linux v2.0.0**:

| Metrica | Valore |
|---|---|
| Passed | 83 |
| Failed | 99 |
| Not applicable | 8 |
| Score | 45% |

Questo è normale per una macchina attacker — Kali non è pensata
per essere hardened ma per fare penetration testing.
Il punteggio basso non è un problema in questo contesto di lab.

## Snapshot
- `06-kali-wazuh-agent-attivo`

## Lezioni Imparate
- Il comando di installazione dell'agent include già le variabili
  `WAZUH_MANAGER` e `WAZUH_AGENT_NAME` — non serve configurazione
  manuale post-installazione
- L'agent si registra automaticamente al Manager non appena il
  servizio parte — nessuna approvazione manuale richiesta
- La SCA viene eseguita automaticamente pochi secondi dopo la
  registrazione — Wazuh inizia a raccogliere dati immediatamente
- Un punteggio SCA basso su Kali è atteso: è una distro offensiva,
  non un server production da hardening
- Da questo momento Wazuh vede tutti i processi, connessioni
  e modifiche ai file che avvengono su Kali in tempo reale
