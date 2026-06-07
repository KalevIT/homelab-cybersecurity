# 00b — VM Storage Layout and Migration Guide

> **Why this document exists:** The lab host was reformatted once during
> this project. This guide documents where every file lives, what each
> file does, and exactly what to do when hardware changes — so a full
> rebuild never starts from scratch again.

---

## Current Storage Layout

```
Host: Windows 11 Pro
│
├── C:\  (OS drive — do NOT store VM files here)
│
└── F:\  (Dedicated lab SSD — 931.5 GB)
    └── VM\
        ├── DISKS\                    ← VM virtual hard disk files (.vmdk)
        │   ├── KALI-ATTACKER\
        │   ├── METASPLOITABLE2\
        │   ├── PFSENSE-FIREWALL\
        │   ├── UBUNTU-BLUETEAM\
        │   ├── WINSERVER2025-DC\
	    │   └── CLIENT01\
        │
        └── GITHUB.REPO\             ← Documentation repository
            └── homelab-cybersecurity\
```

> **Design choice:** VM disks are on a dedicated SSD separate from the OS.
> This means a Windows reinstall does not touch any VM data.
> Always keep VM files on a non-OS drive.

---

## VMware File Types — What Each File Does

When you look inside a VM folder, you see many files. Here is what each one is:

| Extension | Name | Purpose | Safe to delete? |
|-----------|------|---------|-----------------|
| `.vmx` | VM configuration | Main config file — defines vCPU, RAM, disk paths, network | ❌ Never |
| `.vmdk` | Virtual disk | The actual hard drive of the VM (can be split into multiple files) | ❌ Never |
| `.nvram` | BIOS/UEFI state | Saves BIOS settings between reboots | ⚠️ Lose BIOS settings |
| `.vmsd` | Snapshot database | Index of all snapshots — links snapshots to disk delta files | ❌ Never (breaks snapshots) |
| `.vmsn` | Snapshot memory | RAM state at snapshot time (if "Include memory" was checked) | ⚠️ Only with its snapshot |
| `-delta.vmdk` | Snapshot disk delta | Changes since the last snapshot — DO NOT delete manually | ❌ Never |
| `.vmxf` | Team config | VMware team metadata | ⚠️ Safe if not using teams |
| `.log` | VMware log | Debug logs — can grow large | ✅ Safe to delete |
| `.lck/` | Lock directory | Prevents two VMware instances from opening the same VM | ✅ Safe if VM is off |

### The Snapshot Chain

Snapshots work as a chain of delta files:

```
Base disk (original)
    ↓
Snapshot 00 delta  ← changes after snapshot 00
    ↓
Snapshot 01 delta  ← changes after snapshot 01
    ↓
Snapshot 02 delta  ← current state
```

**Never manually delete `-delta.vmdk` files.** Always use
VMware's Snapshot Manager to delete or consolidate snapshots.

---

## Scenario 1 — PC Reformat (Already Happened Once)

### What to do BEFORE reformatting

```
Checklist:
[ ] Export each VM: VM → Manage → Export (creates .ova file)
    OR simply note the .vmx path — if VMs are on a non-OS disk,
    they survive the reformat untouched
[ ] Commit and push all documentation to GitHub
[ ] Note VMware license key (if applicable)
[ ] Export VMware Virtual Network Editor settings (optional)
```

### What to do AFTER reformatting

```
Step 1 — Reinstall VMware Workstation Pro
Step 2 — Reconfigure Virtual Network Editor (see 00-vmware-virtual-network-setup.md)
Step 3 — Re-import VMs:
  VMware → File → Open → navigate to .vmx file on the lab disk
  Repeat for each VM
Step 4 — Verify network adapter assignments for each VM
Step 5 — Boot pfSense first, verify it routes correctly
Step 6 — Clone/re-pull GitHub repository
```

> **Key insight:** If VM files are on a non-OS disk (F:\), they are
> physically untouched by a Windows reformat. You only need to
> re-register them in VMware — no data is lost.

---

## Scenario 2 — Drive Letter Change

VMware stores absolute paths in `.vmx` files. If the drive letter
changes (e.g., `F:\` becomes `G:\`), VMware cannot find the VM files.

### Symptom
```
VMware: "Cannot open the disk 'F:\VM\DISKS\...\disk.vmdk'"
VMware: "The file does not exist"
```

### Fix — Method 1: Edit .vmx directly
```
1. Open the .vmx file in Notepad (right-click → open with)
2. Find all occurrences of the old drive letter: F:\
3. Replace with the new drive letter: G:\
4. Save the file
5. Re-open in VMware
```

### Fix — Method 2: VMware path remapping
```
VMware → Edit → Preferences → Workspace
Update "Default location for virtual machines" to the new path
Then: File → Open → navigate to new .vmx location
```

### Prevention
Assign a fixed drive letter to your lab SSD in Windows:
```
Windows → Disk Management → right-click lab disk → Change Drive Letter
Assign a letter unlikely to change (F:, G:, L: — avoid D: and E:)
```

---

## Scenario 3 — Disk Upgrade (Replace Lab SSD)

When upgrading to a larger SSD:

```
Option A — Clone the entire disk (recommended):
  Use Macrium Reflect Free or similar
  Clone F:\ → new SSD
  Result: all VMs, snapshots, and repo preserved exactly

Option B — Copy files manually:
  xcopy F:\VM\ G:\VM\ /E /H /C /I
  Note: this copies ALL snapshots too — can be very large
  Verify file integrity after copy

Option C — Fresh start:
  Export important VMs as .ova before the change
  Re-import on new disk
  Lose snapshot history (fresh snapshots after)
```

> **Recommendation:** Clone the disk. A 650GB → 2TB upgrade
> with Macrium Reflect takes ~2 hours and preserves everything
> including snapshot chains.

---

## Scenario 4 — Moving the GitHub Repository

If you move the repo from `F:\VM\GITHUB.REPO\homelab-cybersecurity\`
to a new location:

```bash
# In the new location, update the remote (if needed)
git remote -v
# Should show: origin https://github.com/[user]/homelab-cybersecurity.git
# The remote URL does not depend on local path — nothing to change

# If git loses track of the repo (rare):
cd [new-path]\homelab-cybersecurity
git status
# Should work immediately
```

No file paths inside the documentation reference absolute local paths,
so moving the repo folder has no impact on content.

---

## Scenario 5 — Disk Failure Recovery

Worst case: the lab SSD fails with no backup.

### What can be recovered from GitHub
```
✅ All documentation (.md files)
✅ All screenshots committed to the repo
✅ Theory, setup guides, red-team, blue-team notes
❌ VM files (not stored in GitHub — too large)
❌ Snapshot chains
❌ VM configurations (.vmx files, unless committed)
```

### Prevention — What to commit to GitHub
The `.gitignore` should exclude `.vmdk` and `.vmx` files (they're huge).
But consider committing one `vm-configs/` folder with just the `.vmx`
files (text files, ~2KB each) — they document the exact VM configuration
and make recreation much faster.

### Recovery time estimate (no backup)
| VM | Time to recreate | Complexity |
|----|-----------------|------------|
| pfSense | ~1 hour | Low |
| Kali | ~30 min (ISO install) | Low |
| Metasploitable2 | ~10 min (import .vmx) | Very low |
| Ubuntu + Wazuh | ~3 hours | Medium (Wazuh setup) |
| Windows Server + AD | ~2 hours | Medium |
| CLIENT01 + domain join | ~1 hour | Low-Medium |

Total from zero: ~7-8 hours with documentation. Without docs: 2-3x longer.

---

## Current File Inventory

| VM | VMX Location | Disk Size | Latest Snapshot |
|----|-------------|-----------|-----------------|
| pfSense | `F:\VM\DISKS\PFSENSE-FIREWALL\` | ~1 GB | `05-pfsense-lab-gateway-254-internet-ok` |
| Kali | `F:\VM\DISKS\KALI-ATTACKER\` | ~12.4 GB | `08-kali-bloodhound-sharphound-installed` |
| Metasploitable2 | `F:\VM\DISKS\METASPLOITABLE2\` | ~892 MB | `03-meta-post-unrealircd` |
| Ubuntu/Wazuh | `F:\VM\DISKS\UBUNTU-BLUETEAM\` | ~10.5 GB | `05-ubuntu-filebeat-attivo-alert-ok` |
| WinServer 2025 | `F:\VM\DISKS\WINSERVER2025-DC\` | ~15 GB | `04-dc01-dns-forwarder-8888-internet-ok` |
        └── CLIENT01\
| CLIENT01 | `F:\VM\DISKS\CLIENT01\` | ~15 GB | `01-client01-joined-domain-homelab` |

---

## Maintenance Checklist (Run Periodically)

```
Monthly:
[ ] Commit and push all new documentation to GitHub
[ ] Review snapshot count — delete obsolete intermediate snapshots
[ ] Check disk space on lab SSD
[ ] Verify all VMs boot correctly from latest snapshot

Before any hardware change:
[ ] Push all uncommitted docs to GitHub
[ ] Note current snapshot names for each VM
[ ] Verify lab SSD is identifiable by serial number (not just letter)

After any hardware change:
[ ] Verify VMnet configuration in Virtual Network Editor
[ ] Boot pfSense first and verify routing
[ ] Test internet access from all VMs
[ ] Take "post-migration" snapshot on each VM
```
