# 02 — pfSense WebGUI Configuration

## WebGUI Access

- **URL:** https://192.168.233.254
- **Certificate:** self-signed (browser warning is normal in lab environment)
- **Credentials:** admin / [custom password]

![Browser SSL warning](screenshots/pfsense-ssl-warning.png)

![pfSense login page](screenshots/pfsense-login.png)

## Setup Wizard (9 Steps)

![Wizard start](screenshots/pfsense-wizard-start.png)

### Step 1 — Netgate Global Support
Informational page, no action required.

![Step 1 Global Support](screenshots/wizard-step1-global-support.png)

### Step 2 — General Information

| Parameter | Value |
|---|---|
| Hostname | pfSense |
| Domain | homelab.local |
| Primary DNS | 1.1.1.1 |
| Secondary DNS | 8.8.8.8 |
| Override DNS | ✅ |

![Step 2 General Information](screenshots/wizard-step2-general.png)

### Step 3 — Time Server

| Parameter | Value |
|---|---|
| NTP Server | 2.pfsense.pool.ntp.org |
| Timezone | Europe/Rome |

![Step 3 Time Server](screenshots/wizard-step3-time-server.png)

### Step 5 — Configure LAN Interface
LAN confirmed at 192.168.233.254/24.

![Step 5 LAN](screenshots/wizard-step5-lan.png)

### Step 9 — Wizard Completed

![Wizard completed](screenshots/wizard-completed.png)

## Post-Wizard Dashboard

![Dashboard](screenshots/wizard-dashboard.png)

## System Status Post-Wizard

| Item | Status |
|---|---|
| Version | pfSense 2.8.1-RELEASE ✅ |
| Updates | None (latest) ✅ |
| Admin password | Changed ✅ |
| WAN | 192.168.47.128/24 ✅ |
| LAN | 192.168.233.254/24 ✅ |

## Note — Default Password
Wizard did not show the password change step as expected.
Red banner flagged the default password still active.
Changed manually via System → User Manager.

**Lesson learned:** After every installation, always verify
default credentials — do not assume the wizard changed them.

## Snapshot
- `03-pfsense-wizard-completato` — Wizard completed, password updated
