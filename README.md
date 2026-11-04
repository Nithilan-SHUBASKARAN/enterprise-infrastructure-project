<p align="center">
  <img src="images/logo-sbn.png" width="180" alt="SBN Solution Tech Logo">
</p>

# 🚀 Enterprise Network & Virtualized Infrastructure — SBN Solution Tech

> **Technical Exam Project — BTS SIO (IT Services for Organizations) SISR Option (2024–2026 Session)**  
> *Technical documentation of network architecture, Proxmox VE virtualization, and enterprise service deployment.*

---

🌐 **Languages / Langues** : [English](README.md) | [Français](README_FR.md)

---

## 📋 1. Project Overview

**SBN Solution Tech** is a company specializing in high-performance computer hardware sales and IT repair services. To meet the organizational and security requirements of management, the repair workshop, and the retail store, a fully virtualized, highly available enterprise IT infrastructure was designed and implemented.

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
 [VLAN 10 : Management]               [VLAN 20 : Workshop]                 [VLAN 30 : Store]
 192.168.10.0/24                      192.168.20.0/24                      192.168.30.0/24
  - SRV-DC01-D (Active Directory)      - Client PC-0X-A (Win 11)            - SRV-DC02-M (AD Replicated)
  - OpenMediaVault NAS (RAID 1)        - Dolibarr ERP Server                - Xibo CMS Server (Docker)
  - Mailcow Mail Server (Docker)                                            - Store Display Players
```

![Infrastructure Architecture Schema](images/schema-infrastructure.png)

---

## 🏗️ 2. Network Architecture & IP Addressing

### 2.1. Logical VLAN Segmentation (802.1Q)
| VLAN | Role / Service | Subnet CIDR | Gateway (pfSense) | DHCP Range / Pool | Static IP Reservations |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **VLAN 10** | Management & HR | `192.168.10.0/24` | `192.168.10.1` | `192.168.10.10–50` | AD Servers (`.200`), NAS (`.250`), Mail (`.12`) |
| **VLAN 20** | Repair Workshop | `192.168.20.0/24` | `192.168.20.1` | `192.168.20.10–50` | Dolibarr ERP Server (`.200`), PC-A Clients |
| **VLAN 30** | Retail Store & Display | `192.168.30.0/24` | `192.168.30.1` | `192.168.30.10–50` | Secondary AD (`.200`), Xibo Server (`.14`), Players |
| **VLAN 1** | Management / Admin | `192.168.1.0/24` | `192.168.1.1` | `192.168.1.10–50` | Proxmox VE (`192.168.1.X`), Dell iDRAC |

### 2.2. Key Infrastructure Servers
| Hostname | Operating System | IP Address | Deployed Roles & Services |
| :--- | :--- | :--- | :--- |
| `SRV-DC01-D` | Windows Server 2022 | `192.168.10.200` | Primary Domain Controller, DNS, DFS Root |
| `SRV-DC02-M` | Windows Server 2022 | `192.168.30.200` | Secondary Domain Controller, AD & DFS Replication |
| `NAS-D` | OpenMediaVault 8.1 / Debian | `192.168.10.250` | RAID 1 NAS Storage, SMB Shares, OMV-Extras |
| `MAILCOW` | Ubuntu Server 22.04 / Docker | `192.168.20.12` | Mailcow Suite (SMTP/IMAP/SOGo) + LDAP Sync |
| `Dolibarr` | Debian / Apache / MariaDB | `192.168.20.200` | Commercial ERP / CRM + Samba File Server |
| `SRV-XIBO-M` | Ubuntu Server 22.04 / Docker | `192.168.30.14` | Xibo CMS Digital Signage Server & SFTP |
| `Player-XIBO0X` | Windows 11 Pro | DHCP (VLAN 30) | Display Monitors running Xibo Player |

---

## 🛠️ 3. Detailed Step-by-Step Configuration

### 3.1. Virtualization (Proxmox VE) & Firewalling (pfSense)

![pfSense Routing and Firewall Architecture](images/pfsense-routing.png)

#### Step A: Network Bridges Setup in Proxmox VE
1. Access the Proxmox Web GUI (`https://192.168.1.X:8006`).
2. Navigate to **Datacenter** > **Proxmox Node** > **System** > **Network**.
3. Keep `vmbr0` linked to physical interface `enp4s0` (WAN).
4. Create Linux Bridge **`vmbr1`** for administration (`10.0.0.1/30`, Autostart checked).
5. Create main trunk bridge **`vmbr2`** (`LAN_VLANs`, check **VLAN aware**).
6. Click **Apply Configuration**.

#### Step B: pfSense Setup & Inter-VLAN Routing
1. Assign 3 virtual network interfaces to pfSense VM:
   * `em0` / `vtnet0` -> `vmbr0` (WAN Internet)
   * `em1` / `vtnet1` -> `vmbr1` (LAN Admin - `10.0.0.1/30`)
   * `em2` / `vtnet2` -> `vmbr2` (LAN Trunk VLANs)
2. In pfSense GUI (**Interfaces** > **VLANs**), add sub-interfaces on `em2`:
   * Tag `10` -> `VLAN10` (`192.168.10.1/24`)
   * Tag `20` -> `VLAN20` (`192.168.20.1/24`)
   * Tag `30` -> `VLAN30` (`192.168.30.1/24`)
3. **Firewall Rules**:
   * Block inter-VLAN traffic by default.
   * Explicitly allow AD traffic between VLAN 10 and 30 (`LDAP 389/636`, `Kerberos 88`, `RPC 135`, `DNS 53`).

---

### 3.2. Active Directory Domain Services, DFS & Group Policies (GPO)

<p align="center">
  <img src="images/ad-dfs-architecture.png" width="750" alt="Active Directory & DFS Architecture">
</p>

#### Step A: Domain Controllers Deployment & Replication
1. Install **AD DS** and **DNS** roles on `SRV-DC01-D` and create forest **`sbn.solution`**.
2. Join `SRV-DC02-M` to domain `sbn.solution` and promote it as replicated domain controller.

##### Screenshots of IP Configuration & Active Directory:

<div align="center">

| IP Config SRV-DC01-D (VLAN 10 — Management) | IP Config SRV-DC02-M (VLAN 30 — Store) |
| :---: | :---: |
| <img src="images/captures/ip-dc01.png" width="420" alt="IP Config SRV-DC01-D"> | <img src="images/captures/ip-dc02.png" width="420" alt="IP Config SRV-DC02-M"> |

</div>

##### Active Directory, DNS & Groups Console:

![Active Directory Users & Computers Console](images/captures/ad-users-pcs.png)

![DNS Manager & Forward Lookup Zones](images/captures/dns-manager.png)

![Active Directory Groups & OMV Permissions](images/captures/Utilisateurs-Groupes.png)

#### Step B: DFS Namespace & GPO Setup
1. **DFS (Distributed File System)**:
   * Namespace Root: `\\sbn.solution\Users`
   * Target Folder: `Data` on dedicated drive `S:` (250 GB)
   * Replication Group: `Users-Replication` synchronized between DC01 and DC02.

##### DFS & GPO Redirection Screenshots (Full-width vertical layout):

**DFS Namespace Root (`\\sbn.solution\Users`)**:
![DFS Namespace Root](images/captures/dfs-users.png)

**Group Policy — Folder Redirection (GPO)**:
![GPO Folder Redirection](images/captures/gpo-redirection.png)

2. **Deployed GPOs**:
   * **Users-Folder-Redirection**: Redirects *Documents, Desktop, Pictures, Downloads* to `\\sbn.solution\Users\Data\%USERNAME%`.
   * **NAS-Mapping-A**: Maps drive `A:` (`\\NAS-D\Atelier`) for group `Users-A`.
   * **NAS-Mapping-D**: Maps drive `D:` (`\\NAS-D\Direction`) for group `Users-D`.
   * **NAS-Mapping-M**: Maps drive `M:` (`\\NAS-D\Magasin`) for group `Users-M`.

#### Step C: Corporate Wallpapers & Customization Policies (GPO Wallpapers)

To enforce corporate branding across departments, custom desktop wallpapers are automatically applied via GPO:

**Management Department Desktop Wallpaper**:
![Management Desktop Wallpaper](images/captures/wallpaper-direction.png)

**Workshop Repair Department Desktop Wallpaper**:
![Workshop Desktop Wallpaper](images/captures/wallpaper-atelier.png)

**Retail Store Department Desktop Wallpaper**:
![Retail Store Desktop Wallpaper](images/captures/wallpaper-magasin.png)

---

### 3.3. Centralized NAS Storage (OpenMediaVault RAID 1)

![OpenMediaVault RAID 1 Storage Architecture](images/omv-storage-raid.png)

1. **Disk Layout**:
   * System Disk: `/dev/sdb` (64 GB).
   * RAID 1 Volume (Mirror): `/dev/sda` (250 GB) + `/dev/sdc` (250 GB) -> Volume `397a23f3-3263-423e-8d58-5f41380dc0bc`.
2. **Groups & SMB Shares**:
   * Created Groups: `Administrators`, `Users-A` (Workshop), `Users-D` (Management), `Users-M` (Store).
   * Shared Folders: `Atelier`, `Direction`, `Magasin`, `Wallpaper`.
3. **OMV-Extras Extensions**: Activation of **FileBrowser** and **PhotoPrism**.

##### Disk Management & Scheduled Backup Task Screenshots:

**OpenMediaVault Disk & Partition Layout**:
![Disk Management](images/captures/Disques.png)

**Automated Incr. Backup Task**:
![Automated Backup Task](images/captures/backup-task.png)

---

### 3.4. Containerized Application Services (Mailcow, Dolibarr, Xibo)

![Docker Container Services Architecture](images/docker-services.png)

#### A. Unified Email Suite (Mailcow Dockerized)
* **FQDN**: `mail.sbn.solution`
* **Sync cron**: AD LDAP synchronization every minute.

#### Command-line installation procedure:
```bash
# 1. System update and dependencies
sudo apt update && sudo apt upgrade -y
sudo apt install curl git python3-pip apt-transport-https ca-certificates software-properties-common -y

# 2. Hostname setup
sudo hostnamectl set-hostname mail.sbn.solution

# 3. Docker & Docker Compose setup
curl -sSL https://get.docker.com/ | sh
sudo systemctl enable --now docker

# 4. Clone official Mailcow repository
cd /opt
sudo git clone https://github.com/mailcow/mailcow-dockerized
cd mailcow-dockerized

# 5. Config generation and startup
./generate_config.sh  # Enter mail.sbn.solution
sudo docker compose pull
sudo docker compose up -d
```

#### Real-time Active Directory LDAP Sync (Cron):
```bash
crontab -e
# Executed every minute:
* * * * * cd /opt/mailcow-dockerized && /usr/bin/docker compose exec -T php-fpm-mailcow php /web/inc/scripts/ldap_sync.php > /dev/null 2>&1
```

---

### 3.5. ERP / CRM Dolibarr
1. **Web Stack**: Linux Debian, Apache2, PHP, MariaDB.
2. **Features**: Commercial management, inventory, invoicing.
3. **SMB Share & Backups**: Integrated Samba share and automated database dumps sent to OMV NAS `NAS-D`.

---

### 3.6. Digital Signage (Xibo CMS & SFTP)

#### Xibo CMS Deployment via Docker:
```bash
mkdir /opt/xibo && cd /opt/xibo
wget -O xibo-docker.tar.gz https://xibosignage.com/api/downloads/cms
tar --strip-components=1 -zxvf xibo-docker.tar.gz
cp config.env.template config.env
nano config.env  # Configure MYSQL_PASSWORD
docker compose up -d
```

#### FileZilla SFTP Connection for Media Uploads:
* **Host**: `192.168.30.14` (Xibo CMS Server)
* **Protocol**: SFTP (SSH)
* **Port**: `22`
* **User**: `xibo`

##### Xibo CMS Dashboard Screenshot:
![Xibo CMS Dashboard](images/captures/xibo-page.png)

---

## 🔒 4. Backup Strategy & Security Policies

* **Perimeter Security**: Strict inter-VLAN filtering on pfSense with default-deny policy.
* **Backup Policy**: Daily incremental exports of databases and shares to OpenMediaVault RAID 1.
* **High Availability**: Hardware fault tolerance (RAID 1), AD DS domain replication, and DFS-R sync.

---

## 🎓 5. Academic Framework & Credits

Project completed as part of the **BTS Services Informatiques aux Organisations (SIO) - SISR Option** (2024–2026 Session).  
*GitHub - Design & Publication by* **Nithilan SHUBASKARAN**
