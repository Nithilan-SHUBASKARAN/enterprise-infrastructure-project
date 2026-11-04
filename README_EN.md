# 🚀 Enterprise Network & Virtualized Infrastructure — SBN Solution Tech

> **Technical Exam Project — BTS SIO (IT Services for Organizations) SISR Option (2024–2026 Session)**  
> *Technical documentation of network architecture, Proxmox VE virtualization, and enterprise service deployment.*

---

## 🌐 Language Selection
* **[Français (French)](README-v2.md)**
* **[English](README_EN-v2.md)**

---

## 📋 Project Overview

This project aims to design, deploy, and administer a 100% virtualized, segmented, and secure enterprise IT infrastructure for **SBN Solution Tech**, a company specializing in high-performance computer hardware sales and IT repair services.

The entire environment is hosted on a **Proxmox VE** hypervisor running on a physical **Dell PowerEdge R730** server.

![Infrastructure Architecture Schema](images/schema-infrastructure.png)

---

## 🏗️ Network Architecture & Addressing

### 1. Hypervisor & Routing Engine
* **Hypervisor**: Proxmox VE (Debian Linux base, KVM for VMs, LXC for containers).
* **Firewall / Primary Router**: **pfSense** VM (FreeBSD) managing inter-VLAN routing, perimeter firewall rules, and centralized DHCP.

### 2. Logical VLAN Segmentation (802.1Q) & IP Addressing Table

| Segment / VLAN | Network Subnet | Gateway (pfSense) | IP Allocation Range | Main Role & Associated Services |
| :--- | :--- | :--- | :--- | :--- |
| **Management / LAN** | `192.168.1.0/24` | `192.168.1.1` | Static IPs | Hypervisor Proxmox VE (`.254`), pfSense Admin |
| **VLAN 10 (Management / HR)** | `192.168.10.0/24` | `192.168.10.1` | `192.168.10.100 - .199` | AD DC1 (`.200`), NAS OMV (`.250`), Mailcow (`.10`) |
| **VLAN 20 (Workshop)** | `192.168.20.0/24` | `192.168.20.1` | `192.168.20.100 - .199` | Dolibarr ERP/CRM (`.15`), Workshop Workstations |
| **VLAN 30 (Store / Retail)** | `192.168.30.0/24` | `192.168.30.1` | `192.168.30.100 - .199` | AD DC2 (`.200`), Xibo CMS (`.14`), POS & Display Players |

---

## 🛠️ Detailed Component Deployment & Step-by-Step Configuration

### 1. Virtualization (Proxmox VE) & Firewalling (pfSense)
* **Proxmox Bridges**: `vmbr0` (WAN bridge bound to physical NIC), `vmbr1` (VLAN-Aware LAN trunk).
* **pfSense Network Interfaces**:
  * `WAN` (`vmbr0`): DHCP / Static External IP.
  * `LAN` (`vmbr1`): Parent interface for tagged VLAN sub-interfaces (`vtnet1.10`, `vtnet1.20`, `vtnet1.30`).
* **Inter-VLAN Firewall Rules**:
  * Strict isolation between Store (VLAN 30) and Management (VLAN 10).
  * Allowed ports: `53` (DNS), `88` (Kerberos), `445/139` (SMB), `80/443` (HTTP/HTTPS).

*📌 [pfSense Firewall Rules Configuration]*

---

### 2. Active Directory Domain Services & Windows Server Infrastructure
* **Domain Name**: `sbn.solution`
* **Domain Controllers**:
  * Primary DC: `SRV-DC01-D` (`192.168.10.200`) on VLAN 10.
  * Secondary DC: `SRV-DC02-M` (`192.168.30.200`) on VLAN 30.
* **DFS Namespace & Replication**:
  * Namespace Path: `\\sbn.solution\Users\Data`
  * Replication Group: `Users-Replication` synchronizing user home profiles between DC01 and DC02.
* **Group Policy Objects (GPO)**:
  * `GPO-Folder-Redirection`: Automatic redirection of *Desktop, Documents, Downloads, Pictures*.
  * `GPO-Drive-Mapping`: Automatic mapping of `A:` (Atelier), `D:` (Direction), `M:` (Magasin).

*📌 [Active Directory Users and Computers]*
*📌 [DFS Management and Replication Status]*

---

### 3. Centralized NAS Storage (OpenMediaVault RAID 1)
* **Operating System**: OpenMediaVault 6/7 (`192.168.10.250`).
* **Physical Disk Redundancy**: Software **RAID 1 (Mirroring)** using two SATA drives (`/dev/sda`, `/dev/sdc`).
* **SMB/CIFS Shares & Access Control**:
  * Share `/srv/dev-disk-by-uuid/Direction` -> Group `Users-D` (Read/Write)
  * Share `/srv/dev-disk-by-uuid/Atelier` -> Group `Users-A` (Read/Write)
  * Share `/srv/dev-disk-by-uuid/Magasin` -> Group `Users-M` (Read/Write)
  * Share `/srv/dev-disk-by-uuid/Wallpaper` -> Group `Domain Users` (Read-Only)

*📌 [OpenMediaVault RAID 1 Storage Status]*

---

### 4. Containerized Unified Messaging (Mailcow Dockerized)
* **Host OS & Deployment**: Ubuntu Server 22.04 LTS (`192.168.10.10`).
* **Stack**: Docker Engine & Docker Compose (`mailcow-dockerized`).
* **FQDN & Services**: `mail.sbn.solution` hosting Webmail (SOGo), Postfix SMTP, Dovecot IMAP, Rspamd.
* **Active Directory Integration**: LDAP sync configuration for central user authentication.

*📌 [Mailcow Admin UI & Sync Status]*

---

### 5. Enterprise ERP / CRM (Dolibarr) & Shared Storage
* **Host Machine**: Debian 12 LXC Container / VM (`192.168.20.15`).
* **LAMP Stack**: Apache2, MariaDB Server, PHP 8.2.
* **Functionality**: Customer relationships, invoicing, inventory tracking, repair job work-orders.

*📌 [Dolibarr ERP Dashboard]*

---

### 6. Digital Signage (Xibo CMS)
* **Server Deployment**: Xibo CMS inside Docker container on Ubuntu Server (`192.168.30.14`).
* **Media Transfer**: SFTP access (Port 22) via FileZilla for marketing media uploads.
* **Client Display**: Windows/Linux Xibo Player connected to Store display monitors.

*📌 [Xibo Digital Signage Layout Manager]*

---

## 🔒 Backup Strategy & Security Policies

1. **Automated Database Backups**: Nightly cron job dumping Dolibarr MariaDB and Mailcow MySQL databases to OpenMediaVault NAS.
2. **Proxmox Backup (VZDump)**: Weekly full VM/LXC snapshots stored on external NAS target.
3. **Least Privilege & Access Control**: Strict ACLs applied across Active Directory, OMV Samba shares, and pfSense rules.

---

## 🎓 Academic Framework

Project carried out as part of the **BTS Services Informatiques aux Organisations (SIO) - SISR Option** (2024–2026 Session).