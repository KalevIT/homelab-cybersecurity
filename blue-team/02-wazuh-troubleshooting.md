# 02 — Troubleshooting: Wazuh Unreachable After Reboot

## Category
Blue Team / SIEM / Troubleshooting / Configuration

## Objective
Diagnose and fix issues that made the Wazuh dashboard
unreachable after every reboot, and prevented agent events
from appearing in the dashboard.

## Environment

| Role | VM | IP |
|---|---|---|
| SIEM | Ubuntu BlueTeam | 10.10.10.105 |

## Symptoms

**Symptom 1** — On VM reboot:
```
Wazuh dashboard server is not ready yet
```

**Symptom 2** — With Kali agent active but no events visible:
```
No results match your search criteria
```

---

## Full Diagnostics

### BLOCK 1 — Base System

```bash
uname -a && lsb_release -a && uptime && free -h && df -h && nproc
lscpu | grep -E "Model name|CPU\(s\)|Thread"
```

| Resource | Value |
|---|---|
| OS | Ubuntu 26.04 LTS, kernel 7.0.0-15-generic |
| Total RAM | 5.2 GB |
| Swap | 5.8 GB (active) |
| Free disk | 71 GB / 98 GB |
| CPU | 2 cores, AMD Ryzen 9 9900X |

![Base system](screenshots/wazuh-ts-01-sistema-base.png)

### BLOCK 2 — Network and Ports

```bash
ip addr show && ip route show
ping 10.10.10.254 -c 2 && ping 8.8.8.8 -c 2
sudo ss -tlnp
```

**Critical note — port 9200 binding:**
```
[::ffff:10.10.10.105]:9200  →  java (wazuh-indexer)
```
Indexer listens on `10.10.10.105:9200`, **not** on `127.0.0.1:9200`.

![Network and ports](screenshots/wazuh-ts-02-rete-porte.png)

### BLOCK 3 — Services Status

```bash
sudo systemctl status wazuh-{indexer,manager,dashboard} --no-pager
sudo systemctl is-enabled wazuh-{indexer,manager,dashboard}
```

**Critical dashboard logs:**
```
[ConnectionError]: connect ECONNREFUSED 127.0.0.1:9200
```

![Services status](screenshots/wazuh-ts-03-status-servizi.png)
![Is-enabled](screenshots/wazuh-ts-04-is-enabled.png)

### BLOCK 4 — Netplan

```bash
ls /etc/netplan/
# → 00-installer-config.yaml  (not 50-cloud-init.yaml as documented)
networkctl status
```

![Netplan and networkctl](screenshots/wazuh-ts-05-netplan-networkctl.png)

### BLOCK 5 — OpenSearch Configuration and Versions

```bash
sudo cat /etc/wazuh-indexer/opensearch.yml
# → network.host: 10.10.10.105
dpkg -l | grep wazuh
# → wazuh-dashboard/indexer/manager: 4.14.5-1
```

![opensearch.yml and dpkg](screenshots/wazuh-ts-06-opensearch-yml-dpkg.png)

### BLOCK 6 — Filebeat

```bash
sudo filebeat test output     # → OK (connectivity works)
sudo systemctl status filebeat # → inactive (dead) — disabled ← BUG
sudo cat /etc/filebeat/filebeat.yml
# output.elasticsearch.hosts: 10.10.10.105:9200 ← correct configuration
```

![Filebeat diagnosis](screenshots/wazuh-ts-09-filebeat-diagnosi.png)

---

## Root Cause Analysis

### Bug 1 — Dashboard looks for indexer on localhost (symptom 1)

```bash
sudo curl -sk -u admin:[PASS] https://127.0.0.1:9200   # FAILED
sudo curl -sk -u admin:[PASS] https://10.10.10.105:9200 # SUCCESS

grep "opensearch.hosts" /etc/wazuh-dashboard/opensearch_dashboards.yml
# opensearch.hosts: https://localhost:9200  ← WRONG
```

Wazuh installer on Ubuntu 26.04 (unsupported OS) configures
the dashboard to connect to `localhost:9200` but the indexer
listens on `10.10.10.105:9200`.

![Final fix diagnosis](screenshots/wazuh-ts-08-diagnosi-fix-finale.png)

### Bug 2 — Filebeat not started (symptom 2)

Filebeat is the bridge between Manager and Indexer.
It was installed but `disabled` and `inactive (dead)`.

```
Kali Agent → Manager (receives) → Filebeat (OFF) → Indexer (empty)
                                                         ↓
                                             Dashboard: "No results"
```

### Bug 3 — Startup ordering at boot

Services start in parallel. Dashboard started before
the indexer (OpenSearch/Java, 3-5 min init) was ready.

---

## Fixes Applied

### Fix 1 — Permanent Swap

```bash
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

### Fix 2 — opensearch.hosts (symptom 1)

```bash
sudo sed -i \
  's|opensearch.hosts: https://localhost:9200|opensearch.hosts: https://10.10.10.105:9200|' \
  /etc/wazuh-dashboard/opensearch_dashboards.yml
sudo systemctl restart wazuh-dashboard
```

### Fix 3 — Startup ordering

```bash
sudo mkdir -p /etc/systemd/system/wazuh-dashboard.service.d/
sudo tee /etc/systemd/system/wazuh-dashboard.service.d/override.conf << EOF
[Unit]
After=wazuh-indexer.service wazuh-manager.service
Requires=wazuh-indexer.service
EOF

sudo mkdir -p /etc/systemd/system/wazuh-manager.service.d/
sudo tee /etc/systemd/system/wazuh-manager.service.d/override.conf << EOF
[Unit]
After=wazuh-indexer.service
Requires=wazuh-indexer.service
EOF

sudo systemctl daemon-reload
```

### Fix 4 — Filebeat enabled (symptom 2)

```bash
sudo systemctl enable filebeat
sudo systemctl start filebeat
# Active: active (running) ✅

sudo mkdir -p /etc/systemd/system/filebeat.service.d/
sudo tee /etc/systemd/system/filebeat.service.d/override.conf << EOF
[Unit]
After=wazuh-indexer.service wazuh-manager.service
Requires=wazuh-indexer.service
EOF

sudo systemctl daemon-reload
```

![Filebeat fix and override](screenshots/wazuh-ts-10-filebeat-fix-override.png)

---

## Post-Fix Status

| Service | Status | Enabled | Fix |
|---|---|---|---|
| wazuh-indexer | ✅ running | ✅ | — |
| wazuh-manager | ✅ running | ✅ | Fix 3 |
| wazuh-dashboard | ✅ running | ✅ | Fix 2 + Fix 3 |
| filebeat | ✅ running | ✅ | Fix 4 |

**Result:** Dashboard reachable, 213 events from kali-attacker
visible in Threat Hunting section. ✅

## Snapshot
- `05-ubuntu-filebeat-attivo-alert-ok`

## Lessons Learned
- Wazuh has 4 critical components: indexer, manager, dashboard,
  **filebeat** — if one is off the system is partially blind
- Filebeat is the Manager → Indexer bridge: without it events
  reach the Manager but never appear in the dashboard
- `filebeat test output` only checks connectivity, not whether the
  service is running — always also check `systemctl status`
- `systemctl is-enabled` ≠ `systemctl status`: a service can be
  enabled but dead, or running but disabled
- On Ubuntu 26.04 (not supported by Wazuh), the installer generates
  partially wrong configurations that require manual fixes
- Adding all dependent services to systemd overrides avoids
  race conditions at boot
