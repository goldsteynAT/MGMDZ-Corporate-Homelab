<div align="center">
  
# MGMDZ - Corporate Homelab
"Man geht mit der Zeit oder man geht mit der Zeit"

A personal lab environment built to design, document, and operate an enterprise-style hybrid identity platform:<br>
**Active Directory**, Microsoft **Entra** & **Intune**, Windows **Autopilot**, **GPO**-based hardening, segmented **networking**

![Status](https://img.shields.io/badge/status-active-brightgreen)
![Phase](https://img.shields.io/badge/phase-3%20of%205-blue)
![Domain](https://img.shields.io/badge/domain-mgmdz.net-informational)
![Hypervisor](https://img.shields.io/badge/hypervisor-Proxmox%20VE-orange)
![Identity](https://img.shields.io/badge/identity-Hybrid%20AD%20%2B%20Entra%20ID-blueviolet)
![License](https://img.shields.io/badge/docs-CC%20BY%204.0-lightgrey)

</div>

---

## Architecture and Roadmap

<img src="./architecture-roadmap.svg" alt="ArchitectureRoadmap"/>

> The lab also runs separate VLANs for Home, IoT, Camera, and Guest networks. These are isolated from the corporate lab segments by default.

---

## Screenshots
<table style="width: 100%; border-collapse: collapse;">
  <tr>
    <td align="center" width="25%">
      <img src="./images/Proxmox.png" alt="Bild 1" width="100%"><br>
      <b>Proxmox Hypervisor - Virtual Machines</b>
    </td>
    <td align="center" width="25%">
      <img src="./images/DomainController.png" alt="Bild 1" width="100%"><br>
      <b>Domain Controller - OUs and GPOs</b>
    </td>
    <td align="center" width="25%">
      <img src="./images/EntraConnect.png" alt="Bild 2" width="100%"><br>
      <b>Entra Connect - Synchronization Service</b>
    </td>
    <td align="center" width="25%">
      <img src="./images/ODJConnector.png" alt="Bild 3" width="100%"><br>
      <b>ODJ Connector - Event Viewer & Services</b>
    </td>
  </tr>
</table>


---

## What this lab demonstrates

- **Hybrid identity**: on-prem Active Directory ↔ Microsoft Entra ID via Entra Connect Sync
- **Windows Autopilot** in a Hybrid Azure AD Join scenario, fronted by an ODJ Connector
- **Structured AD design** - Top-Level-OU principle, AGDLP-style RBAC with `ROL_` / `PRM_` group prefixes, intentional sync-scope separation between `_CORP` and `_ADMIN`
- **Group Policy hardening** - domain baseline, LAPS with custom managed admin account, RDP hardening, workstation baseline, restricted-groups model
- **Network segmentation** - VLAN-based isolation between corporate, home, IoT, camera, and guest networks; WireGuard VPN with scoped access to the lab segment only
- **Documentation as infrastructure** - every layer is documented, versioned, and reviewable

~ Work in progress ~ <br> additional features such as 
Asset Management (Snipe-IT), Monitoring (Prometheus + Grafana), Ticket System (Zammad), Backup (VEAAM), Patch Management (WSUS), Print Server,.. will be added over time.

---

---
## Environment - Hardware

### Networking
The network is a minimalist UniFi setup, consisting of a UniFi Express Router and an 8-port UniFi switch

<table border="0">
<tr>
<td colspan="2">
<img src="./images/Unifi_Router_Switch.jpeg" alt="Unifi Router Switch" width="730">
</td>
</tr>
<tr>
<td valign="top">
<table>
  <tr><th>Component</th><th>Spec</th></tr>
  <tr><td>Router / Firewall</td><td>UniFi Express</td></tr>
  <tr><td>Switch</td><td>UniFi USW Lite 8 PoE</td></tr>
  <tr><td>VPN</td><td>WireGuard (built-in)</td></tr>
</table>
</td>
<td valign="top">
<table>
  <tr><th>VLAN</th><th>Name</th><th>Subnet</th></tr>
  <tr><td>100</td><td>Corp Lab / Mgmt</td><td><code>10.69.100.0/24</code></td></tr>
  <tr><td>110</td><td>Corp Servers</td><td><code>10.69.110.0/24</code></td></tr>
  <tr><td>120</td><td>Corp Clients</td><td><code>10.69.120.0/24</code></td></tr>
</table>
</td>
</tr>
</table>

### Server
Current setup: One Prxmox node (Lenovo M920Q)<br>
Expansion: A second Lenovo ThinkCentre with an i5-8500T and 32GB RAM has been acquired

<table border="0">
<tr>
<td valign="top" width="340">
<img src="./images/LenovoThinkCentreM920Q.jpeg" alt="Lenovo ThinkCentre M920q" width="320">
</td>
<td valign="top">
<table>
  <tr><th>Component</th><th>Spec</th></tr>
  <tr><td>Host</td><td>Lenovo ThinkCentre M920q — <code>AT1HST01</code></td></tr>
  <tr><td>CPU</td><td>Intel i7-9700T (8 cores)</td></tr>
  <tr><td>RAM</td><td>32 GB DDR4 SODIMM (2 × 16 GB)</td></tr>
  <tr><td>Storage — Tier 0</td><td>256 GB NVMe → <code>local-lvm</code></td></tr>
  <tr><td>Storage — Tier 1</td><td>1 TB SSD → <code>workload-storage</code></td></tr>
  <tr><td>Hypervisor</td><td>Proxmox VE — <code>vmbr0</code> VLAN-aware</td></tr>
</table>
<br>
<table>
  <tr><th>Name</th><th>Role</th><th>VLAN</th><th>OS</th></tr>
  <tr><td><code>AT1SRV01</code></td><td>Domain Controller, DNS, DHCP</td><td>110</td><td>Windows Server 2022</td></tr>
  <tr><td><code>AT1SRV02</code></td><td>Entra Connect Sync</td><td>110</td><td>Windows Server 2022</td></tr>
  <tr><td><code>AT1SRV04</code></td><td>ODJ Connector</td><td>110</td><td>Windows Server 2022</td></tr>
  <tr><td><code>AT1WKS01</code></td><td>Corporate workstation</td><td>120</td><td>Windows 11 Enterprise</td></tr>
</table>
</td>
</tr>
</table>

---

## Tech stack

**Hypervisor & infra** · Proxmox VE · LVM-Thin · QEMU/KVM · UEFI/Q35 <br>
**Identity** · Active Directory Domain Services · Microsoft Entra ID · Entra Connect Sync · Hybrid Azure AD Join <br>
**Endpoint management** · Windows Autopilot · Intune · ODJ Connector · Windows LAPS · Group Policy <br>
**Networking** · UniFi · VLANs · Firewall rules · WireGuard <br>
**OS** · Windows Server 2022 · Windows 11 Enterprise <br>
**Scripting** · PowerShell · Active Directory module · LAPS module <br>
**Documentation** · Markdown · Obsidian · Snipe-IT <br>

---

## Sources / Udemy Courses
[Mastering Active Directory](https://www.udemy.com/course/mastering-active-directory-mit-microsoft-windows-server)
<br>
[MD-102: Endpoint Administrator](https://www.udemy.com/course/md-102-endpoint-administrator-o)
<br>
[Proxmox Masterclass](https://www.udemy.com/course/proxmox-hands-on-masterclass-from-beginner-to-expert)

---

## Related repositories

- [Autopilot Hash Collection](https://github.com/goldsteynAT) — PowerShell tooling used in this lab to harvest hardware hashes for Autopilot registration

