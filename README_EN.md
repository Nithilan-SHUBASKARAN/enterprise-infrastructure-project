# 🚀 Enterprise Network & Virtualized Infrastructure — SBN Solution Tech

> **Technical Exam Project — BTS SIO (IT Services for Organizations) SISR Option**  
> *Complete step-by-step deployment guide, network architecture, system configurations, and diagrams.*

---

🌐 **Languages** : [Français](README.md) | [English](README_EN.md)

## 📋 1. Project & Company Overview

**SBN Solution Tech** is a company specializing in high-end computer hardware sales and IT repair services. To fulfill corporate requirements for Direction, Workshop, and Store operations, a 100% virtualized, segmented, and highly available enterprise IT infrastructure was designed and deployed.

The entire environment is hosted on a **Proxmox VE** hypervisor on a physical **Dell PowerEdge R730** server with remote management via **iDRAC**.

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
 [VLAN 10 : Direction]                [VLAN 20 : Workshop]                 [VLAN 30 : Store]
 192.168.10.0/24                      192.168.20.0/24                      192.168.30.0/24
  - SRV-DC01-D (Active Directory)      - Client PC-0X-A (Win 11)            - SRV-DC02-M (Replicated AD)
  - OpenMediaVault NAS (RAID 1)        - Dolibarr Server (ERP/Samba)        - Xibo CMS Server (Docker)
  - Mailcow Mail Server (Docker)                                            - Xibo Showcase Players
```
![Infrastructure Architecture Diagram](images/schema-infrastructure.png)

---

## 🏗️ 2. IP Addressing Plan & Network Mapping

### 2.1. VLAN Segmentation (802.1Q)
| VLAN | Role / Service | Subnet CIDR | Gateway (pfSense) | DHCP Range | Static IP Reservations |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **VLAN 10** | Direction & Accounting | `192.168.10.0/24` | `192.168.10.1` | `192.168.10.10–50` | `SRV-DC01-D` (`.200`), `NAS-D` (`.250`) |
| **VLAN 20** | Repair Workshop | `192.168.20.0/24` | `192.168.20.1` | `192.168.20.10–50` | `Dolibarr` (`.200`), `MAILCOW` (`.12`) |
| **VLAN 30** | Store & Showcase | `192.168.30.0/24` | `192.168.30.1` | `192.168.30.10–50` | `SRV-DC02-M` (`.200`), `SRV-XIBO-M` (`.14`) |
| **VLAN 1** | Management / Admin | `192.168.1.0/24` | `192.168.1.1` | `192.168.1.10–50` | Proxmox VE (`192.168.1.X`), Dell iDRAC |

### 2.2. Key Servers & Host Table (`sbn.solution` DNS Zone)
| Hostname | Operating System | IP Address | Role & Deployed Services |
| :--- | :--- | :--- | :--- |
| `SRV-DC01-D` | Windows Server 2022 | `192.168.10.200` | Primary Domain Controller, Primary DNS, DFS Root `\\sbn.solution\Users` |
| `SRV-DC02-M` | Windows Server 2022 | `192.168.30.200` | Secondary Domain Controller, Secondary DNS, AD DS & DFS-R Replication |
| `NAS-D` | OpenMediaVault 8.1 / Debian | `192.168.10.250` | NAS RAID 1 Storage, SMB Shares, OMV-Extras (FileBrowser, PhotoPrism) |
| `MAILCOW` | Ubuntu Server 22.04 / Docker | `192.168.20.12` | Mailcow Suite (SMTP/IMAP/SOGo) + Cron LDAP Sync via `s_mailcow` |
| `Dolibarr` | Debian / Apache / MariaDB | `192.168.20.200` | ERP / CRM Software + Samba File Server |
| `SRV-XIBO-M` | Ubuntu Server 22.04 / Docker | `192.168.30.14` | Xibo CMS Digital Signage Server & SFTP File Transfer |
| `PC-01-A` to `PC-03-A` | Windows 11 Pro | `192.168.20.15–17` | Workshop Workstations (OU `PC-A`) |
| `PC-01-D` to `PC-03-D` | Windows 11 Pro | `192.168.10.20–21` | Direction Workstations (OU `PC-D`) |
| `PC-01-M` to `PC-03-M` | Windows 11 Pro | `192.168.30.16–21` | Store Workstations (OU `PC-M`) |
| `Player-XIBO01/02` | Windows 11 Pro | DHCP (VLAN 30) | Showcase Display Screens with Xibo Player |

---

## 🛠️ 3. Technical Specifications & Verified Configurations

### 3.1. Active Directory DS, Organizational Units & Security Groups
* **Active Directory Domain**: `sbn.solution`
* **Organizational Units (OU)**:
  * `Atelier` ➔ Sub-OU: `PC-A`
  * `Direction` ➔ Sub-OU: `PC-D`
  * `Magasin` ➔ Sub-OU: `PC-M`
* **Security Groups & User Members**:
  * **`Users-A`** (Workshop): Hugo SIMON, Romain DUB...
  * **`Users-D`** (Direction): Julien, Lucas, Nithilan
  * **`Users-M`** (Store): Antoine, Maxime
  * **Service Account**: `s_mailcow` (LDAP authentication sync)

### 3.2. Deployed Group Policy Objects (GPOs)
1. **`Redirections-Dossiers-Users`** (Linked to root domain `sbn.solution`):
   * Redirects *Desktop, Documents, Downloads, Pictures, Music, Videos* to `\\sbn.solution\Users\Data\%USERNAME%`.
2. **`NAS-Mapping-A`** (Linked to `Atelier` OU):
   * Maps drive `A:` to `\\NAS-D\Atelier` for group `Users-A`.
3. **`NAS-Mapping-D`** (Linked to `Direction` OU):
   * Maps drive `D:` to `\\NAS-D\Direction` for group `Users-D`.
4. **`NAS-Mapping-M`** (Linked to `Magasin` OU):
   * Maps drive `M:` to `\\NAS-D\Magasin` for group `Users-M`.

### 3.3. DFS Architecture & Replication
* **DFS Namespace Path**: `\\sbn.solution\Users`
* **Namespace Servers**: `\\SRV-DC01-D.sbn.solution\Users` and `\\SRV-DC02-M.sbn.solution\Users`.
* **DFS Target Folder**: `Data` ➔ points to `\\SRV-DC01-D\Users$` and `\\SRV-DC02-M\Users$`.
* **Local Drive Path on DCs**: Dedicated volume `S:\Users-Data` (250 GB).
* **DFS-R Replication Group**: `Users-Replication` synchronizing user profile data between `SRV-DC01-D` and `SRV-DC02-M`.

### 3.4. OpenMediaVault 8.1 NAS (`NAS-D`)
![OpenMediaVault NAS RAID 1 Storage Architecture](images/omv-storage-raid.png)
* **System Specs**: Host `nas-d.sbn.solution`, Static IP `192.168.10.250/24`, Gateway `192.168.10.1`, DNS `192.168.10.200` & `192.168.30.200`.
* **Disk Layout**:
  * System Drive: `/dev/sdb` (64 GiB).
  * Data Drives: `/dev/sda` (250 GiB) + `/dev/sdc` (250 GiB) configured in **RAID 1 (Mirroring)**.
  * RAID 1 Volume UUID: `397a23f3-3263-423e-8d58-5f41380dc0bc`.
* **SMB/CIFS Share Paths**:
  * `Atelier` ➔ `/srv/dev-disk-by-uuid-397a23f3-3263-423e-8d58-5f41380dc0bc/Atelier/`
  * `Direction` ➔ `/srv/dev-disk-by-uuid-397a23f3-3263-423e-8d58-5f41380dc0bc/Direction/`
  * `Magasin` ➔ `/srv/dev-disk-by-uuid-397a23f3-3263-423e-8d58-5f41380dc0bc/Magasin/`
  * `Wallpaper` ➔ `/srv/dev-disk-by-uuid-397a23f3-3263-423e-8d58-5f41380dc0bc/Wallpaper/`

### 3.5. Automated Backup Strategy (Windows Task Scheduler)
* **Task Name**: `backup-NAS-D-Failover-Automatisation`
* **Account**: `SBN\Administrateur`
* **Triggers**: At system startup + Daily at 08:00 (repeating every 5 mins).
* **Action**: Automated incremental backup of `NAS-D` shared folders (`Atelier`, `Direction`, `Magasin`) and DFS folders to local targets `S:\BACKUP\NAS-D` and `S:\BACKUP\Users-Data`.

---

## 🛠️ 4. Deployment Procedures

### 4.1. Virtualization & Firewalling (Proxmox VE & pfSense)
![pfSense Routing and Firewall Architecture](images/pfsense-routing.png)

#### Proxmox VE Bridge Configuration:
1. Connect to Proxmox VE (`https://192.168.1.X:8006`).
2. Keep `vmbr0` bound to physical interface `enp4s0` (WAN).
3. Create `vmbr1` (Management Bridge).
4. Create `vmbr2` (`LAN_VLANs`, with **VLAN aware** enabled).

#### pfSense Sub-interfaces:
* `em0` (`vtnet0`) ➔ `vmbr0` (WAN Internet)
* `em1` (`vtnet1`) ➔ `vmbr1` (Management LAN - `10.0.0.1/30`)
* `em2` (`vtnet2`) ➔ `vmbr2` (Trunk VLANs)
  * `em2.10` ➔ `VLAN10` (`192.168.10.1/24`)
  * `em2.20` ➔ `VLAN20` (`192.168.20.1/24`)
  * `em2.30` ➔ `VLAN30` (`192.168.30.1/24`)

---

### 4.2. Docker Application Stack (Mailcow, Dolibarr, Xibo)
![Docker Container Architecture](images/docker-services.png)

#### Mailcow Deployment & Cron LDAP Sync:
```bash
# 1. Clone & deploy Mailcow
cd /opt && sudo git clone https://github.com/mailcow/mailcow-dockerized
cd mailcow-dockerized
./generate_config.sh  # Hostname: mail.sbn.solution
sudo docker compose pull && sudo docker compose up -d

# 2. Add minute-by-minute AD/LDAP sync Cron job
crontab -e
* * * * * cd /opt/mailcow-dockerized && /usr/bin/docker compose exec -T php-fpm-mailcow php /web/inc/scripts/ldap_sync.php > /dev/null 2>&1
```

#### Xibo CMS & SFTP FileZilla Integration:
* **CMS Deployment**: Docker container on Ubuntu (`192.168.30.14`).
* **FileZilla SFTP Details**: Host `192.168.30.14`, Port `22`, User `xibo` for media asset transfers.

---

## 🔒 5. Security & Backup Policies

* **Perimeter Firewalling**: Strict inter-VLAN blocking on pfSense with default deny policy.
* **Backup Policy**: Daily automated incremental backups of MariaDB databases and SMB share folders to OpenMediaVault RAID 1.
* **High Availability**: RAID 1 disk mirroring, Active Directory DS domain replication, and DFS-R folder synchronization.

---

## 🎓 6. Academic Framework

Technical exam project created for the **BTS Services Informatiques aux Organisations (SIO) - SISR Option** (2024–2026 Session) by **Nithilan SHUBASKARAN**.
