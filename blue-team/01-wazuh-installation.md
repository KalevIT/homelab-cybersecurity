# 01 — Wazuh SIEM Installation (Blue Team VM)

## Category
Blue Team / SIEM / Monitoring

## Objective
Install Wazuh 4.14.5-1 all-in-one on Ubuntu Server 10.10.10.105
as the SIEM platform to monitor attack activities in the lab.

## Environment

| Role | VM | IP |
|---|---|---|
| SIEM | Ubuntu BlueTeam | 10.10.10.105 |

## Wazuh Architecture (single-node)

| Component | Function |
|---|---|
| Wazuh Indexer | Event database (OpenSearch) — port 9200 |
| Wazuh Manager | Collects and analyzes logs from agents — port 1514/1515 |
| Wazuh Dashboard | Web interface — port 443 |
| Filebeat | Log forwarding Manager → Indexer |

## Prerequisites

```bash
free -h    # available RAM
df -h /    # disk space
nproc      # CPU cores
```

| Resource  | Detected value         | Minimum required |
| --------- | ---------------------- | ---------------- |
| RAM       | 3.3 GB (VM total 4 GB) | 4 GB             |
| Free disk | 87 GB                  | 50 GB ✅          |
| CPU       | 2 cores                | 2 cores ✅        |

**Problem:** Total available RAM (3.3 GB) was below the threshold
required by Wazuh (4 GB). Fixed by adding 2 GB of swap and
increasing the VM to 6 GB in VMware.

![Prerequisites and swap](screenshots/wazuh-01-prerequisiti-swap.png)

### Fix — Add Swap

```bash
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
```

### Fix — Increase VM RAM

Ubuntu off → VMware Edit VM Settings → Memory: 4096 MB → **6144 MB** → OK.

## Installation Procedure

### Step 1 — Download script and configuration

```bash
curl -sO https://packages.wazuh.com/4.11/wazuh-install.sh
curl -sO https://packages.wazuh.com/4.11/config.yml
nano config.yml
```

`config.yml` content:

```yaml
nodes:
  indexer:
    - name: node-1
      ip: "10.10.10.105"
  server:
    - name: wazuh-1
      ip: "10.10.10.105"
  dashboard:
    - name: dashboard
      ip: "10.10.10.105"
```

![config.yml configured](screenshots/wazuh-02-config-yml.png)

### Step 2 — Certificate generation

```bash
sudo bash wazuh-install.sh --generate-config-files
```

Output: root, admin, indexer, filebeat, dashboard certificates generated
in `wazuh-install-files.tar`. ✅

![Generate config files](screenshots/wazuh-03-generate-config-indexer-error.png)

### Step 3 — Wazuh Indexer installation

```bash
sudo bash wazuh-install.sh --wazuh-indexer node-1
```

### Step 4 — Start Cluster

```bash
sudo bash wazuh-install.sh --start-cluster
```

Output: cluster security configuration initialized, internal users updated,
filebeat.yml updated, cluster started. ✅

### Step 5 — Wazuh Server installation

```bash
sudo bash wazuh-install.sh --wazuh-server wazuh-1
```

Output: wazuh-manager installed, wazuh-manager started, filebeat installed
and started. ✅

![Server install completed](screenshots/wazuh-04-server-install-ok.png)

### Step 6 — Wazuh Dashboard installation

```bash
sudo bash wazuh-install.sh --wazuh-dashboard dashboard
```

Final output:

```
INFO: You can access the web interface https://10.10.10.105:443
      User: admin
      Password: GJi3fSCl+zjPgZNB0yNOTpCOGAlyk.xp
INFO: Installation finished.
```

![Installation complete with credentials](screenshots/wazuh-07-installazione-completa-credenziali.png)

## Errors Encountered

### Error 1 — Insufficient RAM (first attempt)

```
ERROR: Your system does not meet the recommended minimum hardware
requirements of 4Gb of RAM and 2 CPU cores.
```

**Cause:** Ubuntu 26.04 on a 4 GB VM shows only 3.3 GB RAM available
to the system — below the Wazuh threshold.
**Solution:** VM increased to 6 GB in VMware.

### Error 2 — Dashboard connection refused (first attempt)

```
ERROR: Cannot connect to Wazuh dashboard.
ERROR: Failed to connect with node-1. Connection refused.
```

**Cause:** Dashboard could not find the indexer because the indexer had not
yet been installed (blocked by the previous RAM error).
**Solution:** Installed indexer first, then re-launched dashboard.

![Partial status - manager ok, indexer missing](screenshots/wazuh-05-status-manager-ok-indexer-missing.png)

![Dashboard error - indexer not found](screenshots/wazuh-06-dashboard-error-no-indexer.png)

## Service Verification Post-Installation

```bash
sudo systemctl status wazuh-indexer   # active (running) ✅
sudo systemctl status wazuh-manager   # active (running) ✅
sudo systemctl status wazuh-dashboard # active (running) ✅
```

## Dashboard Access

- **URL:** https://10.10.10.105 (port 443)
- **Certificate:** self-signed — normal browser warning in lab environment
- **User:** admin
- **Password:** saved in password manager

![SSL browser warning](screenshots/wazuh-08-ssl-warning.png)

![Login page](screenshots/wazuh-09-login.png)

![Dashboard overview](screenshots/wazuh-10-dashboard-overview.png)

## Post-Installation Status

| Item             | Status                                            |
| ---------------- | ------------------------------------------------- |
| Wazuh version    | 4.14.5-1 ✅                                        |
| Indexer          | Running ✅                                         |
| Manager          | Running ✅                                         |
| Dashboard        | Running ✅                                         |
| Registered agents| 0 — to configure                                  |
| Alerts (24h)     | 243 Medium, 115 Low (generated by system itself)  |

## Snapshot
- `03-ubuntu-wazuh-installato-dashboard-ok`

## Lessons Learned
- Ubuntu reports less RAM than the total VM — kernel and system
  processes immediately take ~700 MB; Wazuh needs at least 6 GB VM
- Wazuh all-in-one installs 3 separate components that depend
  on each other: installation order is mandatory
  (indexer → cluster start → server → dashboard)
- The "OS not in supported list" warning for Ubuntu 26.04 is ignorable
  in a lab — Wazuh works anyway
- The "Medium/Low" alerts visible immediately in the dashboard are generated
  by the system itself (audit, login, file changes) — not by external agents
- Admin password is shown ONLY ONCE at end of installation:
  copy it immediately to a password manager
