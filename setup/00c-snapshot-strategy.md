# 00c — Snapshot Strategy and Management

> Snapshots are the safety net of the lab. This document explains
> the naming convention, when to take snapshots, how to restore them,
> and why they are especially critical in an offensive security lab.

---

## Why Snapshots Matter More in a Security Lab

In a normal lab you might lose some configuration.
In a security lab, a single wrong command can:
- Corrupt the AD domain (requires full reinstall)
- Permanently alter a vulnerable service you need intact
- Lock you out of a VM (wrong password, broken SSH)
- Destroy evidence of an attack you wanted to analyze again

Snapshots let you **fail forward**: try something aggressive,
if it breaks revert to clean state in 30 seconds, try again.

---

## Naming Convention

Every snapshot in this lab follows a strict format:

```
[NUMBER]-[vm-shortname]-[what-was-done]-[key-detail]

Examples:
  00-winserver2025-installato-pre-ad
  01-dc01-addomain-installed-homelab-local
  07-kali-wireshark-cattura-vsftpd
  05-ubuntu-filebeat-attivo-alert-ok
```

### Rules
| Rule | Example | Reason |
|------|---------|--------|
| Leading number (2 digits) | `05-` | Sorting — snapshots appear in correct order |
| Lowercase, hyphens only | `post-exploit` not `PostExploit` | Consistency, no spaces in names |
| VM context in name | `dc01-` not just `ad-` | Multi-VM labs can have ambiguous names |
| State description | `verified-dns-ok` | Tells you what was working at that point |
| Avoid dates in name | No `2026-05-17-...` | Date is stored automatically by VMware |

### Anti-patterns to avoid
```
❌ "Snapshot 1"          → meaningless after a week
❌ "working backup"      → working how? backup of what?
❌ "before fix"          → before which fix?
❌ "test"                → test of what?
✅ "03-kali-vsftpd-exploit-metasploit"
✅ "02-dc01-verified-dns-forwarder-ok"
```

---

## When to Take Snapshots

### Mandatory snapshot points

```
BEFORE any phase starts:
  → Take snapshot BEFORE installing/configuring anything new
  → This is your "known good" restore point

AFTER a phase succeeds:
  → Take snapshot AFTER verifying the phase works correctly
  → This becomes the baseline for the next phase

BEFORE any destructive test:
  → Take snapshot before any attack that modifies the target
  → Before Kerberoasting, BloodHound, DCSync, etc.

AFTER a successful attack (for Blue Team):
  → Leave evidence in place for SIEM analysis
  → Take snapshot of "compromised state" for later study
```

### Snapshot decision tree

```
Am I about to change something significant?
        │
        ├─ YES → Take snapshot first
        │         Name it: [N]-[vm]-pre-[what-you're-about-to-do]
        │
        └─ NO  → Did something just work correctly?
                        │
                        ├─ YES → Take snapshot now
                        │         Name it: [N]-[vm]-[what-works]-ok
                        │
                        └─ NO  → Continue (no snapshot needed)
```

---

## Current Snapshot Inventory

### pfSense (6 snapshots)
| #   | Name                                     | State at snapshot                |
| --- | ---------------------------------------- | -------------------------------- |
| 00  | `00-pre-installazione`                   | VM created, not started          |
| 01  | `01-pfsense-installato-e-avviato`        | pfSense running, pre-WebGUI      |
| 02  | `02-pfsense-lan-ip-corretto`             | LAN fixed to .254                |
| 03  | `03-pfsense-wizard-completato`           | WebGUI wizard done, password set |
| 04  | `04-pfsense-opt1-lab-configurata`        | OPT1/LAB interface configured    |
| 05  | `05-pfsense-lab-gateway-254-internet-ok` | CURRENT — lab routing working    |

### Kali Linux (8 snapshots)
| #   | Name                                      | State at snapshot                                              |
| --- | ----------------------------------------- | -------------------------------------------------------------- |
| 00  | `00-kali-pre-installazione`               | VM created, ISO mounted                                        |
| 01  | `01-kali-installato-desktop-ok`           | Kali installed, IP confirmed                                   |
| 02  | `02-kali-prima-exploitation-bindshell`    | Pre-exploitation baseline                                      |
| 03  | `03-kali-vsftpd-exploit-metasploit`       | After vsftpd exploit                                           |
| 04  | `04-kali-internet-ok-post-dns-fix`        | Internet verified                                              |
| 05  | `05-kali-unrealircd-exploit`              | After UnrealIRCd exploit                                       |
| 06  | `06-kali-wazuh-agent-attivo`              | Wazuh Agent installed and active                               |
| 07  | `07-kali-wireshark-cattura-vsftpd`        | Wireshark capture of vsftpd exploit                            |
| 08  | `08-kali-bloodhound-sharphound-installed` | CURRENT — BloodHound CE + SharpHound installed, tools verified |

### Metasploitable2 (4 snapshots)
| #   | Name                                     | State at snapshot                   |
| --- | ---------------------------------------- | ----------------------------------- |
| 00  | `00-metasploitable2-pre-avvio`           | VM imported, not started            |
| 01  | `01-metasploitable2-avviato-rete-ok`     | Running, network verified           |
| 02  | `02-meta-rete-ok-pre-exploit-unrealircd` | Pre-UnrealIRCd baseline             |
| 03  | `03-meta-post-unrealircd`                | CURRENT — post all Phase 2 exploits |

### Ubuntu Server / Wazuh (6 snapshots)
| #   | Name                                      | State at snapshot                     |
| --- | ----------------------------------------- | ------------------------------------- |
| 00  | `00-ubuntu-pre-installazione`             | VM created                            |
| 01  | `01-ubuntu-installato-rete-ok`            | Ubuntu installed, SSH working         |
| 02  | `02-ubuntu-internet-ok-pre-wazuh`         | Static IP, ready for Wazuh            |
| 03  | `03-ubuntu-wazuh-installato-dashboard-ok` | Wazuh installed                       |
| 04  | `04-ubuntu-wazuh-post-troubleshooting-ok` | All Wazuh bugs fixed                  |
| 05  | `05-ubuntu-filebeat-attivo-alert-ok`      | CURRENT — Filebeat active, 213 events |

### Windows Server 2025 / DC01 (5 snapshots)
| #   | Name                                       | State at snapshot               |
| --- | ------------------------------------------ | ------------------------------- |
| 00  | `00-winserver2025-installato-pre-ad`       | Base OS, static IP, pre-AD      |
| 01  | `01-dc01-addomain-installed-homelab-local` | Domain created, DC promoted     |
| 02  | `02-dc01-verified-dns-forwarder-ok`        | AD verified, DNS client set     |
| 03  | `03-dc01-ad-users-created`                 | OUs and users created           |
| 04  | `04-dc01-dns-forwarder-8888-internet-ok`   | CURRENT — DNS forwarder working |

---

## How to Restore a Snapshot

### Method 1 — VMware GUI (recommended)
```
1. Power off the VM (or use "Revert" which handles this automatically)
2. VM → Snapshots → Snapshot Manager
3. Select the snapshot to restore
4. Click "Go To" (or "Revert To")
5. Confirm → VM reverts to that exact state
```

> **Warning:** Reverting overwrites the current state permanently.
> If you want to keep the current state too, take a snapshot BEFORE reverting.

### Method 2 — Quick revert to latest snapshot
```
VM → Snapshots → Revert to Latest Snapshot
Shortcut: in the VM tab, click the "Revert" button
```

### Method 3 — Revert without powering off (memory snapshot)
Only works if the snapshot was taken with "Include VM memory":
```
The VM reverts to exactly the running state at snapshot time
Applications, open terminals, and RAM state are all restored
```

---

## Snapshot Best Practices

### Do
```
✅ Take a snapshot before every new lab phase
✅ Verify the VM works correctly before taking a snapshot
✅ Use descriptive names — you will forget context after a week
✅ Keep "clean baseline" snapshots (pre-exploitation states)
✅ Document each snapshot in the relevant setup .md file
```

### Don't
```
❌ Let snapshot chains grow beyond 10 without cleanup
   (each delta file adds disk overhead and slows VMs)

❌ Delete snapshots manually from the filesystem
   (always use VMware Snapshot Manager → Delete)

❌ Take snapshots of running VMs that are mid-operation
   (wait for the operation to complete first)

❌ Use snapshots as long-term backups
   (snapshots are stored with the VM — if the disk dies, they're gone)
   (for real backup: export to .ova and store elsewhere)
```

### Snapshot chain cleanup
When a snapshot series is no longer needed (e.g., you are past Phase 2
and will never revert to pre-vsftpd state):
```
VMware → Snapshot Manager → select old snapshot → Delete
Note: "Delete" removes just that point but preserves the state
      in the chain (it merges deltas forward)
      This reclaims disk space without data loss.
```

---

## Planned Snapshots for Phase 4 — AD Attacks

### Windows 11 / CLIENT01 — current and upcoming

| #   | Name                                     | State                                               |
| --- | ---------------------------------------- | --------------------------------------------------- |
| 00  | `00-client01-installed-pre-config`       | OS installed, localadmin account                    |
| 01  | `01-client01-joined-domain-homelab`      | CURRENT — domain joined, alice.rossi verified       |
| 02  | `02-client01-sharphound-bloodhound-ok` ⬜ | To take after successful BloodHound data collection |

### Kali — next snapshots

```
09-kali-bloodhound-data-imported    ← after BloodHound CE shows domain graph (pending)
```

### DC01 — planned additional snapshots

```
05-dc01-pre-ad-attacks              ← clean baseline before attacks start
06-dc01-post-kerberoasting          ← after Kerberoasting documented
```
