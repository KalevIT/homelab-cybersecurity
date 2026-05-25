# 00 — VMware Virtual Network Setup

## Objective
Configure the three VMware virtual networks required to isolate the lab
from the host machine while maintaining controlled internet access via pfSense.

## Network Architecture

```
Host Windows 11
│
├── VMnet1 (Host-Only) 192.168.233.0/24  ← pfSense management / WebGUI access
│       │
│       └── Host adapter: VMware Network Adapter VMnet1
│           Host IP: 192.168.233.1 (auto-assigned by VMware)
│           pfSense LAN: 192.168.233.254
│
├── VMnet2 (Host-Only) 10.10.10.0/24    ← Isolated lab network
│       │
│       ├── pfSense OPT1/LAB: 10.10.10.254  (gateway)
│       ├── Kali Linux:        10.10.10.100
│       ├── Metasploitable2:   10.10.10.101
│       ├── Ubuntu Server:     10.10.10.105
│       └── Windows Server:    10.10.10.10
│
└── VMnet8 (NAT) auto subnet            ← Internet access for pfSense WAN
        │
        └── pfSense WAN: DHCP from VMware NAT engine
```

## Why This Architecture

| Design Choice | Reason |
|---------------|--------|
| VMnet2 has no host adapter | Lab VMs are invisible to the host OS — true isolation |
| VMnet2 has no DHCP | pfSense assigns IPs manually — full control |
| VMnet1 has host adapter | Allows host browser to reach pfSense WebGUI (192.168.233.254) |
| VMnet1 has no DHCP | pfSense is the only device on this segment — static only |
| VMnet8 is NAT | pfSense WAN gets internet without exposing the host's real IP |

---

## Step-by-Step Configuration

### Prerequisites
- VMware Workstation Pro installed
- Run Virtual Network Editor **as Administrator**
  (Edit → Virtual Network Editor, or search "Virtual Network Editor")

---

### Step 1 — Restore Defaults

Click **"Restore Defaults"** at the bottom left.

This recreates:
- **VMnet1** — Host-only (with host adapter, DHCP enabled by default)
- **VMnet8** — NAT (with host adapter, DHCP enabled)

> Note: VMnet0 (Bridged) may also appear — ignore it, not used in this lab.

---

### Step 2 — Configure VMnet1 (pfSense Management)

Click **VMnet1** in the list, then set:

| Setting | Value |
|---------|-------|
| Type | Host-only |
| Connect a host virtual adapter | ✅ Checked |
| Use local DHCP service | ❌ Unchecked |
| Subnet IP | `192.168.233.0` |
| Subnet mask | `255.255.255.0` |

> **Why no DHCP?** The only device on VMnet1 is pfSense (192.168.233.254).
> The host adapter gets 192.168.233.1 automatically from VMware — no DHCP needed.

---

### Step 3 — Configure VMnet8 (NAT / Internet)

Click **VMnet8** in the list. Leave everything as default:

| Setting | Value |
|---------|-------|
| Type | NAT |
| Connect a host virtual adapter | ✅ Checked (auto) |
| Use local DHCP service | ✅ Checked (auto) |
| Subnet IP | (VMware auto-assigns, e.g. 192.168.197.0) |

> **Note:** The VMnet8 subnet may vary between installations (VMware picks it).
> This doesn't matter — pfSense WAN uses DHCP on VMnet8.

---

### Step 4 — Add VMnet2 (Isolated Lab Network)

Click **"Add Network..."** → select **VMnet2** → click OK.

With VMnet2 selected, set:

| Setting | Value |
|---------|-------|
| Type | Host-only |
| Connect a host virtual adapter | ❌ **Unchecked** |
| Use local DHCP service | ❌ **Unchecked** |
| Subnet IP | `10.10.10.0` |
| Subnet mask | `255.255.255.0` |

> **Why no host adapter?** VMnet2 must be completely invisible to the host.
> All routing goes through pfSense. If the host had a VMnet2 adapter,
> it could bypass pfSense and directly reach lab VMs — breaking isolation.

> **Display note:** VMnet2 may show as type "Custom" in the list instead of
> "Host-only". This is normal for manually added networks in VMware.
> Functionally identical — the radio button selection is what matters.

---

### Step 5 — Apply

Click **Apply** → then **OK**.

---

## Final State — Expected Result

```
Name    Type       External    Host Connection  DHCP     Subnet
------  ---------  ----------  ---------------  -------  -------------
VMnet1  Host-only  —           Connected        No       192.168.233.0
VMnet2  Custom     —           —                No       10.10.10.0
VMnet8  NAT        NAT         Connected        Enabled  192.168.197.x
```

---

## Host Network Adapters Created

After configuration, Windows will show two new virtual adapters:

| Adapter | IP (auto) | Used for |
|---------|-----------|---------|
| VMware Network Adapter VMnet1 | 192.168.233.1 | Access pfSense WebGUI |
| VMware Network Adapter VMnet8 | 192.168.197.1 | VMware NAT internal |

To verify in Windows: `ipconfig /all` — look for VMware adapters.

---

## After VMnet Setup — Next Step

Assign each VM to the correct network adapters:

| VM | Adapter 1 | Adapter 2 | Adapter 3 |
|----|-----------|-----------|-----------|
| pfSense | VMnet8 (WAN) | VMnet1 (LAN) | VMnet2 (OPT1/Lab) |
| Kali Linux | VMnet2 | — | — |
| Metasploitable2 | VMnet2 | — | — |
| Ubuntu Server | VMnet2 | — | — |
| Windows Server | VMnet2 | — | — |

---

## Troubleshooting

### VMnet2 shows as "Custom" instead of "Host-only"
Normal behavior for manually added networks. Functionally identical.
The radio button selection (Host-only) is what determines behavior.

### Cannot access pfSense WebGUI after setup
1. Verify VMnet1 host adapter exists: `ipconfig /all` on host
2. Check pfSense LAN is assigned to VMnet1 in VM settings
3. Verify pfSense LAN IP is 192.168.233.254 from console

### VMnet8 subnet changed after reinstall
Not an issue. pfSense WAN uses DHCP — it gets whatever VMnet8 assigns.
Only affects the host's VMnet8 adapter IP, which is not used by the lab.

---

## Snapshot Reference
This setup is a prerequisite for all VMs — no snapshot taken at this stage.
First snapshot is taken after pfSense is installed and operational.
See: `01-pfsense-vm-creation.md`
