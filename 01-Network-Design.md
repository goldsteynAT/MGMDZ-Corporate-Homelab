# Network Design

UniFi-based network configuration covering VLAN segmentation, wireless, VPN, switching, and firewall strategy for the lab environment.

> **Document version:** 1.5
> **Last updated:** 2026-04-18
> **Note:** All hostnames, domain names, and IP addresses below describe a private lab environment. No production data.
> **Disclaimer:** IP scheme, hostnames, and identifiers in this document have been adjusted for publication. Operational secrets — keys, public IPs, real tenant identifiers — are not included anywhere in this repository.

---

## 1. Network Topology

**IP scheme:** `10.69.0.0/16` (RFC1918)
**Multicast (mDNS):** Enabled between *Home Network* ↔ *IoT Network*

| Name | VLAN | Subnet | Gateway | DHCP Range |
|------|-----:|--------|---------|------------|
| Default | 1 | 10.69.1.0/24 | 10.69.1.1 | .10 – .199 |
| Home Network | 20 | 10.69.20.0/24 | 10.69.20.1 | .10 – .199 |
| Camera Network | 30 | 10.69.30.0/24 | 10.69.30.1 | .10 – .199 |
| IoT Network | 40 | 10.69.40.0/24 | 10.69.40.1 | .10 – .199 |
| Guest Network | 99 | 10.69.99.0/24 | 10.69.99.1 | .6 – .254 |
| Corp Lab (Mgmt) | 100 | 10.69.100.0/24 | 10.69.100.1 | static only |
| Corp Servers | 110 | 10.69.110.0/24 | 10.69.110.1 | static only |
| Corp Clients | 120 | 10.69.120.0/24 | 10.69.120.1 | DHCP via DC relay |
| Management | 254 | 10.69.254.0/24 | 10.69.254.1 | .20 – .199 |
| VPN (WireGuard) | – | 10.69.200.0/24 | 10.69.200.1 | assigned by WireGuard |

### Key static assignments

| IP | Role |
|----|------|
| 10.69.100.10 | Physical hypervisor (`AT1HST01`, Proxmox Web GUI on VLAN 100) |
| 10.69.110.5 | Domain Controller (`AT1SRV01`) |
| 10.69.110.6 | Entra Connect Sync Server (`AT1SRV02`) |
| 10.69.110.8 | ODJ Connector (`AT1SRV04`) |
| 10.69.254.200 | Network hardware (USW-Lite-8) |
| 10.69.200.1 | WireGuard VPN gateway |

---

## 2. Wireless Configuration

**Security standard:** WPA2 / WPA3

| SSID | Mapped VLAN | Settings |
|------|-------------|----------|
| Home | 20 (Home) | 2.4 & 5 GHz, Fast Roaming enabled |
| Camera | 30 (Camera) | 2.4 & 5 GHz, hidden SSID |
| IoT | 40 (IoT) | 2.4 & 5 GHz, Band Steering off |
| Guest | 99 (Guest) | 5 GHz only, Fast Roaming on |

---

## 3. VPN Configuration (WireGuard)

| Setting | Value |
|---------|-------|
| VPN server | UniFi Express (built-in WireGuard) |
| VPN subnet | 10.69.200.0/24 |
| Gateway IP | 10.69.200.1 |
| Port | 51820 / UDP |
| Profile name | `HomeLab-VPN` |

### Access scope

VPN clients can reach the **Corporate Lab Group `[G4]`** only:

- 10.69.100.0/24 (Corp Lab / Proxmox)
- 10.69.110.0/24 (Corp Servers)
- 10.69.120.0/24 (Corp Clients)

VPN clients **cannot** reach:

- 10.69.20.0/24 (Home Network)
- 10.69.30.0/24 (Camera Network)
- 10.69.40.0/24 (IoT Network)

### VPN clients

| Client | Notes |
|--------|-------|
| `LT1` (Laptop) | Interface IP assigned by WireGuard |

---

## 4. Switch Port & Device Configuration

### Port profiles

- **Trunk ports (default):** allow all VLANs
- **Lab Trunk (Port 2):** native VLAN 100, tagged VLANs 110, 120
  Used for: Lenovo ThinkCentre M920q (Proxmox host)
- **Access ports:** allow a single VLAN only

### Hypervisor networking

- Proxmox bridge `vmbr0` configured as **VLAN aware**
- VMs are tagged with their respective VLAN in the Proxmox hardware settings

---

## 5. Proxmox Host & VM Inventory

### Host: `AT1HST01`

| Component | Specification |
|-----------|---------------|
| Hardware | Lenovo ThinkCentre M920q |
| CPU | Intel i7-9700T, 8 cores |
| RAM | 32 GB DDR4 (2 × 16 GB) |
| Storage | 256 GB NVMe + 1 TB SSD |
| Bridge | `vmbr0` (VLAN aware) |

### Storage pools

| Pool ID | Device | Size | Purpose |
|---------|--------|------|---------|
| `local-lvm` | NVMe | 256 GB | Tier 0 — core infrastructure VMs (DC, Sync, templates) |
| `workload-storage` | SSD | 1 TB | Tier 1 — workstations, connectors, workloads |

### Naming convention

```
[SITE][SITE-NR][TYPE][NR]   →   e.g. AT1SRV01

AT  = Austria
1   = Site number
SRV = Server      WKS = Workstation
HST = Hypervisor  TPL = Template
01  = Sequential number
```

### VM inventory

| Name | OS | VLAN | IP | Role | Storage |
|------|----|----:|----|------|---------|
| `AT1SRV01` | Windows Server 2022 | 110 | 10.69.110.5 | DC, DNS, DHCP | local-lvm |
| `AT1SRV02` | Windows Server 2022 | 110 | 10.69.110.6 | Entra Connect Sync | local-lvm |
| `AT1SRV04` | Windows Server 2022 | 110 | 10.69.110.8 | ODJ Connector | workload-storage |
| `AT1TPL01` | Windows Server 2022 | – | – | Template | local-lvm |
| `AT1TPL02` | Windows 11 Enterprise | – | – | Template | local-lvm |
| `AT1WKS01` | Windows 11 Enterprise | 120 | DHCP | Corporate workstation | workload-storage |

---

## 6. DHCP & DNS

DHCP for corporate clients is handled by the Domain Controller (`AT1SRV01`), **not** by the UniFi router.
UniFi VLAN 120 is configured as **DHCP Relay → 10.69.110.5**.

### DHCP scope (corporate clients)

| Option | Value |
|--------|-------|
| Range | 10.69.120.20 – 10.69.120.199 |
| Subnet mask | 255.255.255.0 |
| Gateway (003) | 10.69.120.1 |
| DNS (006) | 10.69.110.5 |
| Suffix (015) | `corp.lab` |
| Lease time | 8 days |

### DNS configuration

| Setting | Value |
|---------|-------|
| Primary zone | `corp.lab` (AD-integrated) |
| Forwarders | 1.1.1.1, 8.8.8.8 |
| Reverse zones | `110.69.10.in-addr.arpa`, `120.69.10.in-addr.arpa` |
| Bind interface | 10.69.110.5 (not 127.0.0.1 / ::1) |

---

## 7. Firewall Groups

| Group | Members |
|-------|---------|
| `[G1] RFC1918` | 192.168.0.0/16, 172.16.0.0/12, 10.0.0.0/8 |
| `[G2] Gateways` | All VLAN gateway IPs (10.69.x.1) |
| `[G4] Corporate Lab` | 10.69.100.0/24, 10.69.110.0/24, 10.69.120.0/24 |
| `[G5] VPN Clients` | 10.69.200.0/24 |

---

## 8. Firewall Rules

### Section A — LAN In (Inter-VLAN routing control)

| # | Name | Action | Source | Destination |
|--:|------|--------|--------|-------------|
| 1 | Allow established / related | ACCEPT | any | any |
| 2 | Allow Home to all VLANs | ACCEPT | Home Net | RFC1918 |
| 3 | Allow Corp Clients to DC | ACCEPT | VLAN 120 | 10.69.110.0/24 |
| 4 | Allow VPN to Corporate Lab | ACCEPT | VPN `[G5]` | Corporate Lab `[G4]` |
| 5 | Block Inter-VLAN | DROP | RFC1918 | RFC1918 |

#### Rule 4 detail — Allow VPN to Corporate Lab

| Setting | Value |
|---------|-------|
| Type | LAN In |
| Action | Accept |
| Protocol | All |
| Source type | Object (Address Group: VPN) |
| Destination type | Object (Address Group: Corporate Lab Group) |
| Before predefined | Yes |

> **Ordering matters.** Rules 3 and 4 must remain above Rule 5 (Block Inter-VLAN), otherwise the explicit allows are shadowed by the deny.

### Section B — LAN Local (Gateway protection)

| Name | Action | Source | Destination |
|------|--------|--------|-------------|
| Block Camera to gateway | DROP | Camera Net | `[G2] Gateways` |
| Block IoT to gateway | DROP | IoT Net | `[G2] Gateways` |
| Block Lab to gateway mgmt | DROP | `[G4] Corporate Lab` | `[G2] Gateways` |

---

## 9. Known Limitations & Planned Improvements

This section documents identified weaknesses in the current ruleset and the planned mitigations. The lab is in active development; this design is iteratively hardened.

### Identified

| # | Area | Observation | Mitigation | Status |
|--:|------|-------------|------------|--------|
| 1 | LAN In · Rule 2 | `Home Net → RFC1918` is broad. A compromise in the Home segment (phishing, IoT bridge, untrusted device) would expose the full Corporate Lab including the DC and Entra Connect Server. | Restrict to `Home Net → Management VLAN (254)` and add explicit allows for required management paths only. | Planned |
| 2 | LAN In · Rule 3 | `Corp Clients → 10.69.110.0/24` allows clients into the entire server range, including the Entra Connect Server (Tier 0). The rule name suggests DC-only access. | Narrow to `Corp Clients → DC (10.69.110.5)` plus explicit per-service allows. | Planned |
| 3 | LAN Local | Default VLAN (1) and Home VLAN (20) can currently reach gateway management interfaces. Only the Management VLAN should. | Add explicit drop rules for non-Management sources to `[G2] Gateways`. | Planned |
| 4 | Tier-0 separation | Entra Connect Sync Server (`AT1SRV02`) is co-located with non-Tier-0 servers in VLAN 110. As the bridge between on-prem AD and Entra ID, it is a Tier 0 asset. | Move to a dedicated Tier-0 VLAN with restricted inbound/outbound; align with Microsoft tiered admin model. | Planned |
| 5 | Outbound segmentation | Camera and IoT VLANs currently retain unrestricted outbound internet access. | Apply destination allowlists for both VLANs; block by default, allow per-vendor endpoints. | Planned |
| 6 | DHCP relay path | DHCP relay from VLAN 120 to 10.69.110.5 implicitly creates a permitted path. Documented for completeness. | Acceptable — required for AD-integrated DHCP. No action. | Accepted |

### Approach

Improvements are tracked in the project [Roadmap](../README.md#roadmap). Each change is implemented in isolation, validated against the affected segments, and documented in the corresponding section above before being marked as resolved.

---

## Result

- **Lab segments** have outbound internet access but are isolated from Home, IoT, and Camera networks.
- **Home network** retains full management access to the Lab and Proxmox host.
- **VPN clients** have scoped access to the Corporate Lab group only.
- **Camera, IoT, and Lab** segments cannot reach UniFi gateway management interfaces — only the Management VLAN can.
