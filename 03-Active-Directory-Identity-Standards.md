# Active Directory & Identity Standards

On-premises Active Directory design, naming conventions, RBAC model, tiered administration, Entra Connect sync configuration, and Group Policy structure for the lab environment.

> **Document version:** 1.3
> **Last updated:** 2026-04-18
> **Disclaimer:** Domain names, hostnames, identifiers, GUIDs, and example values have been adjusted for publication. Operational secrets — tenant IDs, service-account passwords, recovery keys, certificates — are not included anywhere in this repository.

---

## 1. Infrastructure Overview

The on-premises domain `corp.lab` is extended to the cloud via **Microsoft Entra ID** using **Entra Connect Sync**. This hybrid identity model provides seamless SSO across on-premises and cloud resources.

### Domain Controllers & Identity Servers

| Name | Role | VLAN | IP | OS |
|------|------|----:|----|----|
| `AT1SRV01` | Domain Controller | 110 | 10.69.110.5 | Windows Server 2022 |
| `AT1SRV02` | Entra Connect Sync | 110 | 10.69.110.6 | Windows Server 2022 |
| `AT1SRV04` | ODJ Connector | 110 | 10.69.110.8 | Windows Server 2022 |

### Network configuration

| Server | IP | Subnet | Gateway | DNS (primary) | RDP |
|--------|----|--------|---------|---------------|-----|
| `AT1SRV01` | 10.69.110.5 | /24 | 10.69.110.1 | 127.0.0.1 → bound to 10.69.110.5 | NLA enforced |
| `AT1SRV02` | 10.69.110.6 | /24 | 10.69.110.1 | 10.69.110.5 (`AT1SRV01`) | NLA enforced |
| `AT1SRV04` | 10.69.110.8 | /24 | 10.69.110.1 | 10.69.110.5 (`AT1SRV01`) | NLA enforced |

### `AT1SRV04` — ODJ Connector specifics

- Hosts the **Intune Connector for Active Directory** (ODJ Connector)
- Runs under a Managed Service Account: `msaODJConnector$`
- Writes computer objects into the **Autopilot OU** during Autopilot OOBE
- Service `IntuneODJConnector` is set to **Automatic (Delayed Start)**

### What makes `AT1SRV01` a Domain Controller (vs. member server)

- No local user accounts (domain accounts only)
- AD DS role installed, hosting `SYSVOL` and `NETLOGON` shares
- Additional ports open: 88 (Kerberos), 53 (DNS), 389 / 636 (LDAP / LDAPS), 3268 (GC), 445 (SMB)
- Manages Group Policy Objects for the entire domain
- Provides authentication and authorization for all domain members
- Hosts AD-integrated DNS zones

---

## 2. Logical OU Hierarchy

Designed following the **Top-Level-OU principle**: administrative objects are strictly separated from productive corporate data. The `_CORP` container holds all objects synchronized to Entra ID; `_ADMIN` is intentionally excluded from sync to keep privileged accounts out of the cloud.

```
corp.lab
│
├── _CORP                         <- Root OU for all synced corp data
│   │
│   ├── Users
│   │   ├── IT
│   │   ├── HR
│   │   ├── Finance
│   │   └── Management
│   │
│   ├── Groups
│   │   ├── Security Groups
│   │   │   ├── Roles         (Prefix: ROL_)
│   │   │   └── Permissions   (Prefix: PRM_)
│   │   └── Distribution Groups
│   │
│   ├── Computers
│   │   ├── Provisioning      <- Manual / classic domain-join staging
│   │   ├── Autopilot         <- Autopilot OOBE staging (ODJ target)
│   │   ├── Workstations
│   │   └── Servers
│   │
│   └── Service Accounts
│
├── _ADMIN                        <- High-security container, NOT synced
│   ├── Admin Users
│   ├── Admin Workstations       <- PAWs
│   └── Admin Servers            <- Tier 0/1 member servers, NOT synced
│
└── Domain Controllers           <- Built-in, NOT synced
```

### Notes on staging OUs

**`_CORP\Computers\Provisioning`** — Landing zone for devices that are domain-joined manually or via classic methods.

**`_CORP\Computers\Autopilot`** — Landing zone for Autopilot-provisioned devices during OOBE. The ODJ Connector on `AT1SRV04` creates computer objects here. A scheduled task on `AT1SRV01` moves devices to their final site-specific OU (e.g. `Workstations\AT1WKS`) after first user login, based on computer-name prefix and `LastLogonDate`.

**`_ADMIN\Admin Servers`** — Located under `_ADMIN` rather than `_CORP` to reflect Tier 0/1 trust level and to exclude these computer objects from Entra ID sync. Holds `AT1SRV02` (Entra Connect) and `AT1SRV04` (ODJ Connector). Targeted by `GPO-ADM-Baseline` and `GPO-ADM-Hardening` (planned — see [Section 8](#8-future-improvements)).

---

## 3. Directory Object Definitions & Sync Scope

| OU path | Purpose | Sync | Example objects |
|---------|---------|:----:|-----------------|
| `_CORP` | Root container for all synced corp data | ✓ | – |
| `_CORP\Users` | Subdivided into IT, HR, Finance, Management | ✓ | `AT1MUMA` (Max Mustermann) |
| `_CORP\Groups` | All security and distribution groups | ✓ | – |
| `_CORP\Groups\Sec\Roles` | `ROL_` — who is the user | ✓ | `ROL_IT_Staff`, `ROL_HR_Staff` |
| `_CORP\Groups\Sec\Permissions` | `PRM_` — what can the user access | ✓ | `PRM_M365_BusinessPremium` |
| `_CORP\Computers\Provisioning` | Staging for manually domain-joined devices | ✓ | – |
| `_CORP\Computers\Autopilot` | Staging for Autopilot devices (OOBE, ODJ target) | ✓ | `AT1WKS02` (pre-login) |
| `_CORP\Computers\Workstations` | All physical / virtual client devices | ✓ | `AT1WKS01` |
| `_CORP\Computers\Servers` | General member servers | ✓ | – |
| `_CORP\Service Accounts` | Technical service / application accounts | ✓ | `svc_sql_prod` |
| `_ADMIN` | High-security container, never synced | ✗ | – |
| `_ADMIN\Admin Users` | Privileged accounts for IT admins | ✗ | `AT1DORU_Admin` |
| `_ADMIN\Admin Workstations` | PAWs for administrative tasks only | ✗ | – |
| `_ADMIN\Admin Servers` | Privileged member servers, excluded from sync | ✗ | `AT1SRV02`, `AT1SRV04` |
| `Domain Controllers` | Built-in OU, DC objects managed by AD | ✗ | `AT1SRV01` |

---

## 4. Administrative Governance & Conventions

### 4.1 User Naming Convention

**Schema:** `[Site][LastName2][FirstName2]` — max. 4 characters each
**Example:** `AT1MUMA` — Site `AT1`, Mustermann → `MU`, Max → `MA`

The site prefix is derived from the device naming convention (`AT1` = Austria, Site 1) to keep all object types consistent across the directory.

#### Conflict resolution

If a generated username already exists, fall back to **first three characters of the last name + first character of the first name**:

| Case | Username |
|------|----------|
| Primary (Max Mustermann) | `AT1MUMA` |
| Conflict (Martin Musik) | `AT1MUSM` |

#### UPN format

| Aspect | Value |
|--------|-------|
| Pattern | `username@corp.lab` |
| Example | `at1muma@corp.lab` |
| Note | UPN must match the cloud identity in Entra ID exactly to enable seamless SSO. The user experiences no difference between on-prem and cloud sign-in. |

#### Admin & service account conventions

| Type | Pattern | Example | Location |
|------|---------|---------|----------|
| Admin account | `[Username]_Admin` | `AT1MUMA_Admin` | `_ADMIN\Admin Users` (never synced) |
| Service account | `svc_[service]_[env]` | `svc_sql_prod`, `svc_backup_dev` | `_CORP\Service Accounts` |

#### Provisioning

New users are created via the semi-automated PowerShell script `scripts/New-ADUserInteractive.ps1`. The script prompts for first / last name and department, derives the username per the convention above, and creates the account in the correct OU.

#### Current user register

| sAMAccountName | Display name | Department | OU |
|----------------|--------------|------------|----|
| `AT1UNGU` | Gustav Ungustl | IT | `_CORP\Users\IT` |
| `AT1DORU` | Rudolph Dolphus | IT | `_CORP\Users\IT` |
| `AT1POKI` | Kim Possible | HR | `_CORP\Users\HR` |
| `AT1GUJE` | Jennifer Gutdrauf | HR | `_CORP\Users\HR` |
| `AT1GOFR` | Friedrich Gönnjermin | Finance | `_CORP\Users\Finance` |
| `AT1SCJO` | Josephine Schilling | Finance | `_CORP\Users\Finance` |
| `AT1CRSE` | Sebastien Crypte | Finance | `_CORP\Users\Finance` |
| `AT1BUHE` | Heidemarie Buchinger | Management | `_CORP\Users\Management` |
| `AT1SCCH` | Christian Schmosa | Management | `_CORP\Users\Management` |

#### Admin accounts (`_ADMIN\Admin Users`, not synced)

| sAMAccountName | Display name |
|----------------|--------------|
| `AT1DORU_Admin` | Rudolph Dolphus (Admin) |
| `Administrator` | Administrator |

### 4.2 Group Naming Convention (AGDLP model)

Group structure implements the **AGDLP** principle (Account → Global → Domain Local → Permission), adapted as a two-tier RBAC model with `ROL_` and `PRM_` prefixes.

#### Assignment chain

```
User → ROL_ Group → PRM_ Group → Resource / Permission
```

#### Worked example

```
Max Mustermann
  ↳ member of: ROL_IT_Staff
      ↳ ROL_IT_Staff member of: PRM_Fileserver_IT_Read
          ↳ PRM_Fileserver_IT_Read has access to: \\fileserver\IT
```

When a user changes departments, only their `ROL_` group membership is updated — all permissions follow automatically without touching individual resource ACLs. Example: remove from `ROL_IT_Staff`, add to `ROL_HR_Staff` → user instantly loses all IT access and gains all HR access.

#### Group prefixes

| Prefix | Meaning | Examples | Location |
|--------|---------|----------|----------|
| `ROL_` | Role groups — *who* the user is | `ROL_IT_Staff`, `ROL_HR_Staff` | `_CORP\Groups\Sec\Roles` (synced) |
| `PRM_` | Permission groups — *what* the user can access | `PRM_Fileserver_HR_Read`, `PRM_M365_BusinessPremium_License` | `_CORP\Groups\Sec\Permissions` (synced) |
| `DEV_` | Device groups (cloud-only) — Intune / Autopilot targeting | `DEV_Autopilot_All`, `DEV_Intune_Compliance_WKS` | Entra ID only — security, dynamic device |
| `DP_` | Autopilot deployment profiles | `DP_Hybrid_Autopilot_Standard` | Intune (Devices → Enrollment → Deployment Profiles) |

#### Role groups (`_CORP\Groups\Sec\Roles`)

| Group | Members |
|-------|---------|
| `ROL_IT_Staff` | `AT1UNGU`, `AT1DORU` |
| `ROL_HR_Staff` | `AT1POKI`, `AT1GUJE` |
| `ROL_Finance_Staff` | `AT1GOFR`, `AT1SCJO`, `AT1CRSE` |
| `ROL_Management` | `AT1BUHE`, `AT1SCCH` |

#### Permission groups (`_CORP\Groups\Sec\Permissions`)

| Group | Nested role groups |
|-------|--------------------|
| `PRM_M365_BusinessPremium_License` | (assigned per user as needed) |
| `PRM_Workstations_LocalAdmin` | `ROL_IT_Staff` |
| `PRM_Workstations_RDP` | `ROL_IT_Staff` |
| `PRM_Fileserver_All_FullControl` | `ROL_IT_Staff` |
| `PRM_Fileserver_Root_Read` | `ROL_HR_Staff`, `ROL_Finance_Staff`, `ROL_Management` |
| `PRM_Servers_RDP` | `ROL_IT_Staff` |

#### Permission groups (`_ADMIN`, not synced)

| Group | Members |
|-------|---------|
| `PRM_ADM-Servers_RDP` | `AT1DORU_Admin`, `Administrator` |
| `PRM_PAW_Logon` | `AT1DORU_Admin`, `Administrator` |

> `_ADMIN` permission groups are populated **directly with admin accounts**, never with `ROL_` groups, to keep the admin path independent from synced role membership.

#### RBAC nesting map

```
ROL_IT_Staff       → PRM_Fileserver_All_FullControl
                   → PRM_Workstations_LocalAdmin
                   → PRM_Workstations_RDP
                   → PRM_Servers_RDP

ROL_HR_Staff       → PRM_Fileserver_Root_Read
ROL_Finance_Staff  → PRM_Fileserver_Root_Read
ROL_Management     → PRM_Fileserver_Root_Read
```

### 4.3 Tiered Administration Model

Access is separated into three tiers to limit the blast radius of a compromised account. Admin accounts must never be used outside their designated tier.

| Tier | Purpose | Assets | Access | Examples |
|:----:|---------|--------|--------|----------|
| **0** | Control Plane (highest privilege) | Domain Controllers, Entra Connect Sync Server, admin accounts | Only from `_ADMIN\Admin Workstations` (PAWs) | `AT1SRV01`, `AT1SRV02`, `AT1MUMA_Admin` |
| **1** | Server Tier | Application servers, member servers | Tier 1 admin accounts only | `AT1SRV04` |
| **2** | Workstation Tier (lowest privilege) | Workstations, standard user accounts | Standard accounts from `_CORP\Users` | `AT1WKS01`, `AT1MUMA` |

> Tier-0 isolation enforcement is partially implemented; full hardening of `AT1SRV02` and the planned `_ADMIN\Admin Servers` GPO chain is tracked in [Section 8](#8-future-improvements).

---

## 5. Entra Connect Sync Configuration

Entra Connect Sync runs on `AT1SRV02` and synchronizes on-premises AD with Microsoft Entra ID for hybrid identity.

| Aspect | Value |
|--------|-------|
| Sync server | `AT1SRV02` (10.69.110.6) |
| Sync method | Password Hash Sync (PHS) |
| Sync scope | `_CORP` OU only — `_ADMIN` and `Domain Controllers` excluded |
| Delta cycle | Every 30 minutes (default) |
| Full sync | Manual (`PolicyType Initial`) |

### Operational commands

```powershell
# Trigger a full / initial sync
Start-ADSyncSyncCycle -PolicyType Initial

# Trigger a delta sync
Start-ADSyncSyncCycle -PolicyType Delta

# Pause / resume the scheduler
Set-ADSyncScheduler -SyncCycleEnabled $false
Set-ADSyncScheduler -SyncCycleEnabled $true
```

### Service Connection Point (SCP)

The SCP is an AD object that tells devices where to register for Entra ID during Hybrid Join. It must contain:

| Attribute | Value |
|-----------|-------|
| `azureADName` | `corp.lab` (verified custom domain) |
| `azureADId` | `<TENANT-ID>` |

**Location:**
```
CN=<TENANT-DEVICE-REG-GUID>,
CN=Device Registration Configuration,
CN=Services,
CN=Configuration,
DC=corp,
DC=lab
```

> Entra Connect does **not** automatically update the SCP when a new verified domain is added. After a domain change, update manually via PowerShell or `Initialize-ADSyncDomainJoinedComputerSync`.

### Microsoft 365 license

| Aspect | Value |
|--------|-------|
| Plan | Microsoft 365 Business Premium |
| Includes | Intune Plan 1, Entra ID P1, Autopilot, Defender |
| Tenant | `corplab.onmicrosoft.com` |
| Domain | `corp.lab` (verified custom domain) |

---

## 6. Hybrid Entra Join

Hybrid Entra Join registers devices in **both** the on-premises AD and Microsoft Entra ID, enabling Conditional Access, Intune management, and SSO.

### Verified result state on `AT1WKS01`

| Property | Value | Notes |
|----------|-------|-------|
| `DomainJoined` | YES | Joined to `corp.lab` via `AT1SRV01` |
| `AzureAdJoined` | YES | Registered in Entra ID via Hybrid Join |
| `EnterpriseJoined` | NO | Expected — Workplace Join not in use |

Verification command (run on the client):

```powershell
dsregcmd /status
```

### Troubleshooting notes

- **`AzureAdJoined = NO`** → check the SCP first. `azureADName` must match the verified Entra ID domain (`corp.lab`), not the `*.onmicrosoft.com` tenant name.
- **After SCP changes:** run `dsregcmd /leave` on the client, reboot, then trigger `Start-ADSyncSyncCycle -PolicyType Initial`.
- **Hybrid Join task** runs via Task Scheduler:
  `Microsoft → Windows → Workplace Join → Automatic-Device-Join`

---

## 7. GPO Structure

Detailed step-by-step implementation guides for each GPO live under `gpos/` *(planned)*. This section covers naming, link strategy, and the per-GPO configuration tables that document **what** each policy enforces.

### 7.1 Naming convention

```
GPO-[SCOPE]-[FUNCTION]
```

| Scope | Meaning |
|-------|---------|
| `DOM` | Domain root |
| `WKS` | Workstations |
| `SRV` | Servers |
| `ADM` | Admin Servers |
| `SEC` | Cross-cutting / security |
| `PAW` | Admin Workstations |

### 7.2 GPO overview

| GPO | Link target(s) | Purpose | Status |
|-----|----------------|---------|--------|
| `GPO-DOM-Baseline` | `corp.lab` (domain root) | Password, lockout, audit policy | **Implemented** |
| `GPO-SEC-LAPS` | Workstations + Servers (separate links) | Windows LAPS, 30-day rotation | **Implemented** |
| `GPO-WKS-Baseline` | `_CORP\Computers\Workstations` | Firewall, screen lock, autoplay, guest, event log sizing | **Implemented** |
| `GPO-WKS-RemoteAccess` | `_CORP\Computers\Workstations` | RDP hardening, session host security, restricted-groups RDP allow-list | **Implemented** |
| `GPO-WKS-UserExperience` | `_CORP\Computers\Workstations` | Store, Copilot, Widgets, Spotlight, desktop personalization | **Implemented** |

### 7.3 GPO details

#### `GPO-DOM-Baseline`

**Link:** `corp.lab` (domain root)

##### Password policy

Path: `Computer Configuration → Policies → Windows Settings → Security Settings → Account Policies → Password Policy`

| Setting | Value |
|---------|-------|
| Enforce password history | 24 passwords remembered |
| Maximum password age | 180 days |
| Minimum password length | 14 characters |
| Password must meet complexity requirements | Enabled |
| Minimum password age | 30 days |

##### Account lockout policy

Path: `… → Account Policies → Account Lockout Policy`

| Setting | Value |
|---------|-------|
| Account lockout threshold | 10 invalid logon attempts |
| Account lockout duration | 10 minutes |
| Reset account lockout counter after | 10 minutes |

##### Audit policy

Path: `… → Local Policies → Audit Policy`

| Setting | Configuration |
|---------|---------------|
| Audit account logon events | Success, Failure |
| Audit account management | Success, Failure |

---

#### `GPO-SEC-LAPS`

**Link:** Three separate links — `_CORP\Computers\Workstations`, `_CORP\Computers\Servers`, and (planned) `_ADMIN\Admin Servers`.

> A single link at `_CORP\Computers` would also cover the `Provisioning` and `Autopilot` staging OUs, which should not receive LAPS policy before devices reach their final OU.

##### Prerequisites

```powershell
# Schema extension — once per forest
Update-LapsADSchema

# OU permissions — once per target OU
Set-LapsADComputerSelfPermission -Identity "OU=Workstations,OU=Computers,OU=_CORP,DC=corp,DC=lab"
Set-LapsADComputerSelfPermission -Identity "OU=Servers,OU=Computers,OU=_CORP,DC=corp,DC=lab"
```

##### Settings

Path: `Computer Configuration → Policies → Administrative Templates → System → LAPS`

| Setting | Value |
|---------|-------|
| Configure password backup directory | Active Directory |
| Enable password encryption | Enabled |
| Configure password size | 3 (number of stored encrypted passwords) |
| Configure password complexity | Large + small letters + numbers + special chars |
| Configure password age (days) | 30 |
| Post-authentication actions | Reset password after managed account is used |
| Automatic Account Management | Manage custom admin: `Aristoteles` (Enable: Yes, Randomize: No — lab simplicity) |
| Do not allow password expiration time longer than required | Enabled |
| Authorized password decryptor | *Not configured* (planned: `G-SEC-LAPS-Admins`) |

> If `GPO-WKS-Security` later renames or disables the built-in local Administrator, the LAPS-managed account name above must match the actual account, otherwise LAPS will fail silently on affected machines.

##### Verification

```powershell
gpupdate /force                      # on a target client
Get-LapsADPassword -Identity AT1WKS01  # on AT1SRV01
```

---

#### `GPO-WKS-Baseline`

**Link:** `_CORP\Computers\Workstations`

##### Windows Defender Firewall

Path: `Computer Configuration → Policies → Windows Settings → Security Settings → Windows Defender Firewall with Advanced Security → Properties` (per profile)

| Setting | Domain | Private | Public |
|---------|--------|---------|--------|
| Firewall state | On | On | On |
| Inbound connections | Block (default) | Block (default) | Block (default) |
| Outbound connections | Allow (default) | Allow (default) | Allow (default) |
| Display notification | No | No | No |
| Apply local firewall rules | – | – | No |
| Apply local connection security rules | – | – | No |
| **Logging — size limit (KB)** | 16384 | 16384 | 16384 |
| Log dropped packets | Yes | Yes | Yes |
| Log successful connections | Yes | Yes | Yes |

##### Screen lock

Path: `Computer Configuration → Policies → Windows Settings → Security Settings → Local Policies → Security Options`

| Setting | Value |
|---------|-------|
| Interactive logon: Machine inactivity limit | 900 (sec) |
| Interactive logon: Don't display last signed-in | Enabled |

Path: `User Configuration → Policies → Administrative Templates → Control Panel → Personalization`

| Setting | Value |
|---------|-------|
| Enable screen saver | Enabled |
| Password-protect the screen saver | Enabled |
| Screen saver timeout | Enabled — 900 |

##### Autoplay

Path: `Computer Configuration → Policies → Administrative Templates → Windows Components → AutoPlay Policies`

| Setting | Value |
|---------|-------|
| Disallow Autoplay for non-volume devices | Enabled |
| Set the default behavior for AutoRun | Enabled — *Do not execute any autorun commands* |
| Turn off Autoplay | Enabled — *All drives* |

##### Guest & local accounts

Path: `Computer Configuration → Policies → Windows Settings → Security Settings → Local Policies → Security Options`

| Setting | Value |
|---------|-------|
| Accounts: Guest account status | Disabled |
| Accounts: Limit local account use of blank passwords to console logon only | Enabled |
| Network access: Sharing and security model for local accounts | Classic — local users authenticate as themselves |

##### Event log sizing

Path: `Computer Configuration → Policies → Administrative Templates → Windows Components → Event Log Service`

| Log | Maximum log size (KB) | Behavior at max size |
|-----|----------------------:|----------------------|
| Application | 32 768 | Disabled (overwrite as needed) |
| Setup | 32 768 | Disabled |
| System | 32 768 | Disabled |
| Security | 196 608 | Disabled |

---

#### `GPO-WKS-RemoteAccess`

**Link:** `_CORP\Computers\Workstations`

##### Required group

```powershell
New-ADGroup -Name "PRM_Workstations_RDP" `
  -GroupScope DomainLocal -GroupCategory Security `
  -Path "OU=Permissions,OU=Security Groups,OU=Groups,OU=_CORP,DC=corp,DC=lab"

Add-ADGroupMember -Identity "PRM_Workstations_RDP" -Members "ROL_IT_Staff"
```

##### Remote Desktop Session Host

Path: `Computer Configuration → Policies → Administrative Templates → Windows Components → Remote Desktop Services → Remote Desktop Session Host`

| Subfolder | Setting | Value |
|-----------|---------|-------|
| Security | Require user authentication for remote connections by using NLA | Enabled |
| Security | Set client connection encryption level | Enabled — High Level |
| Security | Require secure RPC communication | Enabled |
| Security | Require use of specific security layer for remote connections | Enabled — SSL |
| Security | Always prompt for password upon connection | Enabled |
| Connections | Restrict RDS users to a single session | Enabled |
| Connections | Allow users to connect remotely by using RDS | Enabled |
| Connections | Automatic reconnection | Enabled |
| Windows Defender Firewall | Allow inbound Remote Desktop exceptions | Enabled |

##### RDP allow-list (Restricted Groups)

Path: `Computer Configuration → Policies → Windows Settings → Security Settings → Restricted Groups`

- Add group `Remote Desktop Users`
- Members of this group: `CORP\PRM_Workstations_RDP`
- "This group is a member of": *(empty)*

---

#### `GPO-WKS-UserExperience`

**Link:** `_CORP\Computers\Workstations`

##### Computer Configuration

| Path | Setting | Value |
|------|---------|-------|
| `Administrative Templates → Windows Components → Store` | Turn off the Store application | Enabled |
| `Administrative Templates → Windows Components → Widgets` | Allow Widgets | Disabled |
| `Policies → Administrative Templates → System → Group Policy` | Configure user GPO loopback processing mode | Enabled — *Merge* |

##### User Configuration

| Path | Setting | Value |
|------|---------|-------|
| `Administrative Templates → Start Menu and Taskbar` | Do not allow pinning Store app to the Taskbar | Enabled |
| `Administrative Templates → Windows Components → Windows Copilot` | Turn off Windows Copilot | Enabled |
| `Administrative Templates → Windows Components → Cloud Content` | Turn off all Windows spotlight features | Enabled |

##### Registry preferences

Path: `User Configuration → Preferences → Windows Settings → Registry`

| Action | Hive | Key path | Value name | Type | Value |
|--------|------|----------|------------|------|------:|
| Replace | HKCU | `Software\Microsoft\Windows\CurrentVersion\Explorer\Wallpapers` | `BackgroundType` | REG_DWORD | 0 |
| Replace | HKCU | `Software\Microsoft\Windows\CurrentVersion\Explorer\HideDesktopIcons\NewStartPanel` | `{2cc5ca98-6485-489a-920e-b3e88a6ccce3}` | REG_DWORD | 1 |

### 7.4 Build order

```
1. GPO-DOM-Baseline
2. GPO-SEC-LAPS                (after Update-LapsADSchema)
3. GPO-WKS-Baseline
4. GPO-WKS-RemoteAccess
5. GPO-WKS-UserExperience
```

### 7.5 Key design decisions

- **Pilot-then-roll-out** for every GPO: link first to a single test device via Security Filtering (`AT1WKS01$`), validate over 24–48 hours, then re-add `Authenticated Users` and remove the device-specific filter.
- **AppLocker starts in Audit mode** on all GPOs — switch to Enforce only after reviewing event logs.
- **Domain Controllers OU**: no custom GPOs. Default Domain Controllers Policy is left untouched.
- **`GPO-ADM-Baseline`** (planned) will exist because `_ADMIN\Admin Servers` does not inherit `GPO-SRV-Baseline` (different OU tree). Settings will be explicitly duplicated.
- **`GPO-SEC-LAPS`** is linked separately to each target OU rather than at `_CORP\Computers` to avoid applying LAPS policy in `Provisioning` and `Autopilot` staging OUs.

---

## 8. Future Improvements

| # | Area | Planned change | Status |
|--:|------|----------------|--------|
| 1 | GPO `GPO-WKS-Security` | SMBv1, UAC hardening, Credential Guard for workstations | Planned |
| 2 | GPO `GPO-SRV-Baseline` | Firewall, RDP, audit baseline for `_CORP\Computers\Servers` | Planned |
| 3 | GPO `GPO-ADM-Baseline` + `GPO-ADM-Hardening` | Server baseline + logon restrictions, PowerShell logging for `_ADMIN\Admin Servers` | Planned |
| 4 | GPO `GPO-PAW-Hardening` | Strictest policy for Privileged Access Workstations | Planned |
| 5 | LAPS — authorized decryptor | Create `G-SEC-LAPS-Admins` and configure as authorized password decryptor | Planned |
| 6 | Tier-0 isolation | Move `AT1SRV02` to dedicated Tier-0 storage and VLAN — see [01-network-design.md §9](01-network-design.md#9-known-limitations--planned-improvements) | Planned |
| 7 | PAW rollout | Build dedicated Privileged Access Workstation under `_ADMIN\Admin Workstations`; restrict admin logon to PAW only | Planned |
| 8 | Step-by-step implementation guides | Move detailed per-GPO build / pilot / rollback procedures to `gpos/*.md` | Planned |
