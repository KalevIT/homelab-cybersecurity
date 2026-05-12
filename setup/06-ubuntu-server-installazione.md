# 06 — Installazione Ubuntu Server LTS (Blue Team VM)

## Obiettivo
Installare Ubuntu Server LTS come VM Blue Team sulla rete LAB
(VMnet2/10.10.10.0/24). Questa VM ospiterà Wazuh SIEM per
monitorare e rilevare le attività di attacco nel lab.

## Configurazione VM

| Parametro   | Valore                                             |
| ----------- | -------------------------------------------------- |
| Nome VMware | Ubuntu-BlueTeam                                    |
| OS          | Ubuntu Server 26.04 LTS (GNU/Linux 7.0.0 x86_64)   |
| vCPU        | 2 core                                             |
| RAM         | 6144 MB (aumentata da 4096 MB per requisiti Wazuh) |
| Disco       | 100 GB (single file, SCSI)                         |
| Rete        | VMnet2 (LAB — 10.10.10.0/24)                       |
| Percorso    | D:\VM\MACHINES\UBUNTU-BLUETEAM\                    |

## Installazione — Parametri Scelti

### Lingua e Tastiera
![Lingua](screenshots/ubuntu-server-01-lingua.png)
![Tastiera](screenshots/ubuntu-server-02-tastiera.png)

### Rete
Il DHCP di pfSense assegnò 10.10.10.103/24 durante l'installazione.
Al primo riavvio l'IP era cambiato a 10.10.10.105/24 — normale
con DHCP, ma problematico per un server. Risolto con IP statico
dopo l'installazione (vedi sezione Fix IP Statico).

![Rete DHCP durante installazione](screenshots/ubuntu-server-04-rete-dhcp.png)

### Partizionamento
Schema semplice — tutto in una partizione ext4 su disco da 100 GB.

![Storage guidato](screenshots/ubuntu-server-06-partizione.png)
![Riepilogo partizioni](screenshots/ubuntu-server-07-disk.png)

### Utente e SSH

| Campo | Valore |
|---|---|
| Nome | Blue Team |
| Hostname | ubuntu-blueteam |
| Username | blueteam |
| OpenSSH | ✅ installato |

![Profilo utente](screenshots/ubuntu-server-08-user.png)
![SSH abilitato](screenshots/ubuntu-server-09-ssh.png)

### Snap aggiuntivi
Nessuno selezionato — installeremo Wazuh manualmente.

![Extra snaps](screenshots/ubuntu-server-10-extra-install.png)
![Installazione in corso](screenshots/ubuntu-server-11-installazione.png)

## Primo Avvio

![Primo login](screenshots/ubuntu-server-12-first-login.png)

## Fix — IP Statico

Con DHCP l'IP può cambiare a ogni riavvio, rendendo SSH e Wazuh
instabili. Ubuntu 26.04 usa **Netplan** per la configurazione di rete.

```bash
sudo nano /etc/netplan/50-cloud-init.yaml
```

Sostituisci il contenuto con:

```yaml
network:
  version: 2
  ethernets:
    ens32:
      dhcp4: false
      addresses:
        - 10.10.10.105/24
      routes:
        - to: default
          via: 10.10.10.254
      nameservers:
        addresses:
          - 10.10.10.254
          - 1.1.1.1
```

Applica:

```bash
sudo netplan apply
ip addr show ens32
# conferma: 10.10.10.105/24 statico
```

## Verifica Rete

```bash
ip route show
# default via 10.10.10.254 dev ens32 proto static ✅

ip addr show
# ens32: 10.10.10.105/24 ✅

ping 10.10.10.254 -c 4   # pfSense gateway ✅
ping 10.10.10.100 -c 4   # Kali ✅
ping 10.10.10.101 -c 4   # Metasploitable2 ✅
```

![ip addr e ping](screenshots/ubuntu-server-13-test-ping.png)

## Verifica Internet — Post Fix DNS

```bash
ping 8.8.8.8 -c 2
# 2/2 pacchetti ricevuti ✅ (connettività IP)

ping google.com -c 2
# 2/2 pacchetti ricevuti ✅ (risoluzione DNS funzionante)
```

L'output di `ip route show` mostra `proto static` — conferma
che l'IP statico Netplan è attivo correttamente.

![Internet e DNS verificati](screenshots/ubuntu-server-14-internet-ok-post-dns-fix.png)

## Accesso SSH da Kali

```bash
ping 10.10.10.105 -c 2
ssh blueteam@10.10.10.105
# accettare fingerprint: yes
whoami   # blueteam
```

![SSH da Kali](screenshots/kali-ssh-ubuntu-server.png)

## Mappa Rete Completa

| VM | IP | VMnet | Ruolo |
|---|---|---|---|
| Host Windows | 192.168.233.1 | VMnet1 | Host fisico |
| pfSense LAN | 192.168.233.254 | VMnet1 | Firewall mgmt |
| pfSense LAB | 10.10.10.254 | VMnet2 | Gateway lab |
| Kali Linux | 10.10.10.100 | VMnet2 | Attacker |
| Metasploitable2 | 10.10.10.101 | VMnet2 | Target |
| Ubuntu Server | 10.10.10.105 | VMnet2 | Blue Team / SIEM |

## Snapshot
- `00-ubuntu-pre-installazione` — VM creata, ISO montata
- `01-ubuntu-installato-rete-ok` — Ubuntu operativo, SSH funzionante
- `02-ubuntu-internet-ok-pre-wazuh` — IP fisso 10.10.10.105, pronto per Wazuh
- `03-ubuntu-wazuh-installato-dashboard-ok`

## Lezioni Imparate
- Ubuntu Server 26.04 usa `ens32` invece di `eth0` — nome
  diverso da Kali e Metasploitable2 (dipende dal driver VMware)
- DHCP su server è un anti-pattern: l'IP può cambiare al riavvio
  rompendo SSH, agent Wazuh, e qualsiasi servizio che lo referenzia
- Netplan è il sistema di configurazione rete moderno di Ubuntu —
  sostituisce `/etc/network/interfaces` delle versioni precedenti
- L'SSH fingerprint al primo collegamento è normale: va accettato
  una volta sola, poi viene memorizzato in `~/.ssh/known_hosts`
- `proto static` nell'output di `ip route show` conferma che
  la configurazione Netplan è attiva e non DHCP
