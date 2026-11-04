# 🚀 Enterprise Network & Virtualized Infrastructure — SBN Solution Tech

> **Technical Exam Project — BTS SIO (IT Services for Organizations) SISR Option (2024–2026 Session)**  
> *Complete technical documentation for network architecture, Proxmox VE virtualization, Active Directory, DFS, and containerized enterprise services.*

---

🌐 **Languages / Langues** : [Français](README.md) | [English](README_EN.md)

---

## 📋 1. General Project & Enterprise Overview

**SBN Solution Tech** is an enterprise specializing in high-end computer hardware sales and IT repair services. To meet corporate operational requirements for Management (Direction), Repair Workshop (Atelier), and Retail Store (Magasin), a fully virtualized, highly available, and segmented enterprise IT infrastructure was designed and deployed.

The entire environment is hosted on a **Proxmox VE** hypervisor platform running on a physical **Dell PowerEdge R730** server.

```text
                                    [ WAN / Internet Access ]
                                               |
                                     (vmbr0 - WAN Proxmox)
                                               |
                                    +---------------------+
                                    |  pfSense Firewall   | (Inter-VLAN Routing & DHCP)
                                    +---------------------+
                                               |
                                  (vmbr2 - Trunk VLAN-Aware)
                                               |
          +------------------------------------+------------------------------------+
          |                                    |                                    |
 [VLAN 10 : Direction]                [VLAN 20 : Atelier]                  [VLAN 30 : Magasin]
 192.168.10.0/24                      192.168.20.0/24                      192.168.30.0/24
  - SRV-DC01-D (Active Directory)      - Client PC-0X-A (Win 11)            - SRV-DC02-M (Replicated AD)
  - OpenMediaVault NAS (RAID 1)        - Dolibarr Server (ERP/Samba)        - Xibo CMS Server (Docker)
  - Mailcow Mail Server (Docker)                                            - Xibo Store Display Players
```

![Infrastructure Architecture Schema](images/schema-infrastructure.png)

---

## 🏗️ 2. IP Addressing Plan & Routing Table

### 2.1. VLAN Mapping (802.1Q)
| VLAN | Role / Service | CIDR Subnet | Gateway (pfSense) | DHCP Pool / Hosts | Static IP Reservations |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **VLAN 10** | Direction & Accounting | `192.168.10.0/24` | `192.168.10.1` | `192.168.10.10–50` | AD Server (`.200`), NAS (`.250`), Mail (`.12`) |
| **VLAN 20** | Repair Workshop | `192.168.20.0/24` | `192.168.20.1` | `192.168.20.10–50` | Dolibarr ERP (`.200`), PC-A Workstations |
| **VLAN 30** | Retail Store & Display | `192.168.30.0/24` | `192.168.30.1` | `192.168.30.10–50` | Secondary AD (`.200`), Xibo Server (`.14`), Players |
| **VLAN 1** | Management / Admin | `192.168.1.0/24` | `192.168.1.1` | `192.168.1.10–50` | Proxmox VE (`192.168.1.X`), Dell iDRAC |

### 2.2. Servers & Infrastructure Equipment
| Hostname | Operating System | IP Address | Deployed Roles & Services |
| :--- | :--- | :--- | :--- |
| `SRV-DC01-D` | Windows Server 2022 | `192.168.10.200` | Primary Domain Controller, DNS, DFS Root |
| `SRV-DC02-M` | Windows Server 2022 | `192.168.30.200` | Secondary Domain Controller, AD & DFS Replication |
| `NAS-D` | OpenMediaVault 8.1 / Debian | `192.168.10.250` | RAID 1 NAS Storage, SMB Shares, OMV-Extras |
| `MAILCOW` | Ubuntu Server 22.04 / Docker | `192.168.20.12` | Mailcow Mail Suite (SMTP/IMAP/SOGo) + LDAP Sync |
| `Dolibarr` | Debian / Apache / MariaDB | `192.168.20.200` | Commercial ERP / CRM + Samba File Server |
| `SRV-XIBO-M` | Ubuntu Server 22.04 / Docker | `192.168.30.14` | Xibo CMS Digital Signage Server & SFTP |
| `Player-XIBO0X` | Windows 11 Pro | DHCP (VLAN 30) | Retail Store Display Monitors running Xibo Player |

---

## 🛠️ 3. Step-by-Step Deployment Guide

### 3.1. Proxmox VE Virtualization & pfSense Routing

![pfSense Routing and Firewall Architecture](images/pfsense-routing.png)

#### Step A: Proxmox VE Linux Bridges Setup
1. Access Proxmox Web Interface (`https://192.168.1.X:8006`).
2. Navigate to **Datacenter** > **Proxmox Node** > **System** > **Network**.
3. Keep `vmbr0` connected to physical interface `enp4s0` (WAN).
4. Create administration bridge **`vmbr1`** (Linux Bridge, Autostart checked).
5. Create main trunk bridge **`vmbr2`** (Name: `LAN_VLANs`, check **VLAN aware**).
6. Click **Apply Configuration**.

#### Step B: pfSense Setup & Inter-VLAN Routing
1. Assign 3 virtual network interfaces to pfSense VM:
   * `em0` / `vtnet0` -> `vmbr0` (WAN Internet)
   * `em1` / `vtnet1` -> `vmbr1` (LAN Admin - `10.0.0.1/30`)
   * `em2` / `vtnet2` -> `vmbr2` (LAN Trunk VLANs)
2. In pfSense (**Interfaces** > **VLANs**), create sub-interfaces on `em2`:
   * Tag `10` -> `VLAN10` (`192.168.10.1/24`)
   * Tag `20` -> `VLAN20` (`192.168.20.1/24`)
   * Tag `30` -> `VLAN30` (`192.168.30.1/24`)
3. **Firewall Rules**:
   * Block inter-VLAN traffic by default.
   * Explicitly allow AD traffic between VLAN 10 and 30 (`LDAP 389/636`, `Kerberos 88`, `RPC 135`, `DNS 53`).

---

### 3.2. Active Directory DS, DFS & Group Policies (GPO)

#### Step A: Domain Controllers Deployment & Replication
1. Install **AD DS** and **DNS** roles on `SRV-DC01-D` and promote forest **`sbn.solution`**.
2. Join `SRV-DC02-M` to domain `sbn.solution` and promote it as replicated DC.

![Active Directory & DFS Architecture](images/ad-dfs-architecture.png)

#### Step B: DFS Namespace & GPOs Configuration
1. **DFS (Distributed File System)**:
   * Namespace Root: `\\sbn.solution\Users`
   * Target Directory: `Data` on dedicated drive `S:` (250 GB)
   * Replication Group: `Users-Replication` synchronizing between DC01 and DC02.
2. **Deployed GPOs**:
   * **Users-Folder-Redirection**: Redirects *Desktop, Documents, Pictures, Downloads* to `\\sbn.solution\Users\Data\%USERNAME%`.
   * **NAS-Mapping-A**: Maps drive `A:` (`\\NAS-D\Atelier`) for group `Users-A`.
   * **NAS-Mapping-D**: Maps drive `D:` (`\\NAS-D\Direction`) for group `Users-D`.
   * **NAS-Mapping-M**: Maps drive `M:` (`\\NAS-D\Magasin`) for group `Users-M`.

---

### 3.3. OpenMediaVault NAS Storage (RAID 1 & SMB Shares)

![OpenMediaVault RAID 1 Storage Architecture](images/omv-storage-raid.png)

1. **Disk Layout**:
   * System Disk: `/dev/sdb` (64 GB).
   * RAID 1 Volume (Mirroring): `/dev/sda` (250 GB) + `/dev/sdc` (250 GB) -> Volume `397a23f3-3263-423e-8d58-5f41380dc0bc`.
2. **SMB/CIFS Groups & Shares**:
   * Created Groups: `Administrators`, `Users-A` (Workshop), `Users-D` (Direction), `Users-M` (Store).
   * Shared Folders: `Atelier`, `Direction`, `Magasin`, `Wallpaper`.

---

### 3.4. Containerized Application Services (Mailcow, Dolibarr, Xibo)

![Docker Container Services Architecture](images/docker-services.png)

#### A. Unified Email Suite (Mailcow Dockerized)
* **Deployment**: Docker Compose on Ubuntu Server 22.04 LTS (`mail.sbn.solution`).
* **Active Directory Sync**: Automated minute cron job running LDAP user synchronization script.

#### B. Enterprise ERP / CRM (Dolibarr)
* **Stack**: LAMP (Linux Debian, Apache2, PHP 8.2, MariaDB).
* **Backups**: Automated database dumps exported to OpenMediaVault NAS.

#### C. Digital Signage (Xibo CMS)
* **CMS Host**: Ubuntu Server (`192.168.30.14`).
* **Media Transfers**: SFTP access (Port 22) via FileZilla for store display players.

---

## 🔒 4. Security, Backup Strategy & Compliance

* **Perimeter Security**: Strict inter-VLAN filtering on pfSense with default-deny policy.
* **Backup Strategy**: Automated daily incremental database dumps and file shares exported to OpenMediaVault RAID 1 NAS.
* **High Availability**: Hardware fault-tolerance (RAID 1), Active Directory domain replication, and DFS-R folder sync.

---

## 🎓 5. Academic Framework & Credits

Technical exam project completed by **Nithilan SHUBASKARAN** for the **BTS Services Informatiques aux Organisations (SIO) - SISR Option** (2024–2026 Session).
