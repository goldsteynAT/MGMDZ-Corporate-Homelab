# Inventory Standards

Naming conventions, asset tracking, and virtualization standards for the homelab environment. This document defines *how* assets are organized; it complements [01-network-design.md](01-network-design.md), which covers the operational network topology.

> **Document version:** 1.1
> **Last updated:** 2026-04-18
> **Disclaimer:** Hostnames, identifiers, and example values have been adjusted for publication. Operational secrets — serial numbers, MAC addresses, API tokens — are not included anywhere in this repository.

---

## 1. Naming Convention

Every device follows the schema `[SITE][SITE-NR][TYPE][NR]`.

```
AT1HST01
│ │  │  └── Sequential number
│ │  └───── Type code
│ └──────── Site number
└────────── Site (country)
```

**Domain suffix:** `corp.lab`

### Type codes

| Code | Type | Examples |
|------|------|----------|
| `HST` | Physical host | Proxmox node, bare-metal server |
| `SRV` | Server | AD-DC, application servers, databases |
| `WKS` | Workstation | Windows 10/11 VMs, physical workstations |
| `LAP` | Laptop | Mobile devices |
| `SWI` | Switch | Network switches |
| `GWY` | Gateway | Routers, firewalls, modems |
| `IOT` | IoT device | Smart-home hubs, printers, cameras |

---

## 2. VM ID Convention

VM IDs are assigned in ranges aligned with the VLAN tag of the corresponding network. Where possible, the **first two digits of the ID match the VLAN tag**.

| ID range | Category | Description | Example |
|----------|----------|-------------|---------|
| 101 – 109 | Infrastructure | Templates | `101` (WinSrv2022 template) |
| 110 – 119 | Corporate Server | Systems in VLAN 110 | `110` (`AT1SRV01`) |
| 120 – 129 | Workstations | Client VMs in VLAN 120 | `120` (`AT1WKS01`) |
| 254 | Management | Management appliances in VLAN 254 | `254` (mgmt tool) |

### Current VM register

| Name | ID | OS | VLAN | IP | Role | Storage |
|------|---:|----|----:|----|------|---------|
| `AT1SRV01` | 110 | Windows Server 2022 | 110 | 10.69.110.5 | DC, DNS, DHCP | local-lvm |
| `AT1SRV02` | 111 | Windows Server 2022 | 110 | 10.69.110.6 | Entra Connect Sync | local-lvm |
| `AT1SRV04` | 113 | Windows Server 2022 | 110 | 10.69.110.8 | ODJ Connector | workload-storage |
| `AT1TPL01` | 101 | Windows Server 2022 | – | – | Template | local-lvm |
| `AT1TPL02` | 102 | Windows 11 Enterprise | – | – | Template | local-lvm |
| `AT1WKS01` | 120 | Windows 11 Enterprise | 120 | DHCP | Corporate workstation | workload-storage |

---

## 3. Asset Database (Obsidian)

An Obsidian database titled **Hardware Inventory** acts as the operational dashboard for all physical and virtual assets.

### Properties (in display order)

| # | Property | Type | Example |
|--:|----------|------|---------|
| 1 | Model | Select | Lenovo ThinkCentre M920q |
| 2 | Primary IP | Text | 10.69.100.10 |
| 3 | OS | Select | Proxmox VE |
| 4 | Role | Select | Hypervisor, NAS, Backup |
| 5 | CPU Model | Text | i7-9700T |
| 6 | CPU Cores | Number | 8 |
| 7 | RAM Total (GB) | Number | 32 |
| 8 | RAM Slots | Select | 2 / 2 filled |
| 9 | Storage (GB) | Number | 256 |
| 10 | Location | Select | Server rack |
| 11 | Status | Status | Active, In stock |
| 12 | Serial Number | Text | (chassis S/N) |
| 13 | MAC Address | Text | (primary NIC) |

### Master template — `STANDARD PHYSICAL HOST` (`AT1HST`)

Each entry in the database is created from a master template with two areas:

**Area A — Properties (top):** all "hard facts" from the table above, populated at creation. These are the filterable / searchable fields.

**Area B — Page content (bottom):** a maintenance-oriented detail table with exact part numbers and identifiers.

| Column | Content (example) |
|--------|-------------------|
| Model | Exact model number (e.g. `XXRS00QT00`) |
| CPU | Details and vintage (e.g. `i7-9700T @ 2 GHz / Q2 2019`) |
| RAM | Exact part number (e.g. `Samsung M471...`, `2666 MHz CL19`) |
| Storage | Exact part number (e.g. `Samsung MZVLB...`, NVMe) |

Maintenance logs and notes (e.g. PCIe passthrough configurations, BIOS changes) are recorded below the table.

---

## 4. Asset Management Strategy

A two-layer approach separates day-to-day operations from formal asset management.

### Level 1 — Obsidian (Operations)

| Aspect | Detail |
|--------|--------|
| Purpose | Daily operations, planning, overview |
| Focus | *„What's the IP?", „What role does this server have?", „Is RAM still free?"* |
| Maintenance | Manual upon commissioning |

### Level 2 — Snipe-IT (Asset Management — planned)

| Aspect | Detail |
|--------|--------|
| Purpose | Inventory, warranty, license management |
| Focus | *„Purchase date", „Warranty period", „Invoice number"* |
| Maintenance | Import / automated / on purchase |

---

## 5. Virtualization Standards (Proxmox)

Standard configurations applied on the hypervisor `AT1HST01` to ensure consistent resource allocation, performance, and supportability.

### 5.1 VM sizing tiers

| OS Type | vCPU | RAM | Disk | Notes |
|---------|----:|----:|----:|-------|
| Windows Server 2022 (AD) | 2 | 3 GB | 40 GB | Lean setup |
| Windows Server 2022 (App) | 2 | 4 GB | 40 GB | Standard |
| Windows 11 Client | 2 | 4 – 6 GB | 64 GB | Microsoft minimum |
| Linux Server (CLI) | 1 | 1 GB | 20 GB | Lightweight |

### 5.2 Hardware virtualization settings

| Setting | Value | Reason |
|---------|-------|--------|
| CPU type | `host` | CPU pass-through for full feature set |
| SCSI controller | VirtIO SCSI single | Performance, isolation per VM |
| Disk bus | SCSI (Discard / TRIM enabled) | Reclaim space on thin pools |
| Network | VirtIO (paravirtualized) | Performance |
| BIOS / Machine | OVMF (UEFI) / Q35 | Required for Windows 11, modern PCIe |
| QEMU Guest Agent | Enabled | Clean shutdown, IP reporting, freeze/thaw for backup |

### 5.3 Storage pool strategy

`AT1HST01` uses a two-tier storage layout to separate critical infrastructure from workloads and direct I/O where it matters.

#### Tier 0 — `local-lvm` (256 GB NVMe)

| | |
|---|---|
| Purpose | Core infrastructure that must always be available and benefits from maximum I/O performance. |
| Contents | `AT1SRV01` (DC), `AT1SRV02` (Entra Sync), `AT1TPL01`, `AT1TPL02` |
| Pool ID | `local-lvm` |

#### Tier 1 — `workload-storage` (1 TB SSD, LVM-Thin)

| | |
|---|---|
| Purpose | Workloads, client VMs, and future expansion. Thin provisioning allows over-allocation across multiple VMs. |
| Contents | `AT1SRV04` (ODJ Connector), `AT1WKS01`, future VMs |
| Pool ID | `workload-storage` |
| LVM volume group | `vg-workloads` |
| Thin pool | `tp-workloads` |

#### Placement rule

- **Core infrastructure VMs** (domain auth, identity sync, templates) → `local-lvm`
- **Workloads, workstations, optional services** → `workload-storage`

### 5.4 VM startup order

Configured per VM in the Proxmox **Options → Start/Shutdown order** panel to enforce a clean boot sequence after a host restart.

| Order | VM | Role | Start at boot |
|------:|----|------|:-:|
| 1 | `AT1SRV01` | DC, DNS, DHCP | Yes |
| 2 | `AT1SRV02` | Entra Connect Sync | Yes |
| 3 | `AT1SRV04` | ODJ Connector | Yes |

> The DC must be fully online before Entra Connect attempts its first sync, and before the ODJ Connector attempts to talk to AD. Startup-order delays are tuned to roughly 60 seconds between tiers.

---

## 6. Future Improvements

| # | Area | Planned change | Status |
|--:|------|----------------|--------|
| 1 | Asset management | Migrate purchase / warranty data from Obsidian to Snipe-IT | Planned |
| 2 | Sync | Automate VM register from Proxmox API into the Obsidian database | Planned |
| 3 | Tier-0 isolation | Move `AT1SRV02` (Entra Connect) onto a dedicated Tier-0 storage and VLAN — see [01-network-design.md §9](01-network-design.md#9-known-limitations--planned-improvements) | Planned |
| 4 | Backup | Define backup schedule and retention per tier (Tier 0 daily, Tier 1 weekly) | Planned |
