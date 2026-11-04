# 🚀 Network Infrastructure & Virtualized Services — SBN Solution Tech

> **Technical Project — Higher National Diploma / Associate Degree in IT & Network Administration (BTS SIO SISR — 2024–2026 Session)**  
> *Technical documentation for network architecture, Proxmox VE virtualization, and enterprise service deployment.*

---

## 📋 Project Overview

This project aims to design, deploy, and manage a fully virtualized, secure 100% enterprise network infrastructure for **SBN Solution Tech** (a computer hardware sales and repair business).

The entire environment is hosted on a **Proxmox VE** hypervisor platform running on a physical **Dell PowerEdge R730** server.

---

## 🏗️ Network Architecture & Virtualization

### 1. Hypervisor & Routing
* **Hypervisor**: Proxmox VE (Debian Linux-based, utilizing KVM for VMs and LXC for containers).
* **Firewall / Main Router**: **pfSense** Virtual Machine (FreeBSD) handling inter-VLAN routing, security filtering, and centralized DHCP service.

### 2. VLAN Segmentation (802.1Q)
The internal network is logically segmented into subnets to ensure traffic isolation and security:
* **VLAN 10 — Management / Accounting**: `192.168.10.0/24`
* **VLAN 20 — Workshop (Atelier)**: `192.168.20.0/24`
* **VLAN 30 — Store / Retail (Magasin)**: `192.168.30.0/24`
* **LAN Management / Infrastructure**: `192.168.1.0/24` (pfSense / Proxmox administration network)

### 3. Infrastructure Logical Diagram

```text
                     [ WAN / Internet Access ]
                                 |
                          (vmbr0 - WAN)
                                 |
                        +-----------------+
                        |   pfSense VM    | (Firewall & Centralized DHCP)
                        +-----------------+
                                 |
                      (vmbr1 - Trunk VLAN-Aware)
                                 |
       +-------------------------+-------------------------+
       |                         |                         |
  [VLAN 10: Management]    [VLAN 20: Workshop]     [VLAN 30: Retail Store]
  192.168.10.0/24          192.168.20.0/24         192.168.30.0/24
   - Active Directory       - Workshop Workstations - Xibo Server (CMS)
   - OpenMediaVault NAS     - Dolibarr ERP/CRM      - POS & Display Players
   - Mailcow Mail Server
```

---

## 🛠️ Deployed Services & Servers

### 🔐 Active Directory & Windows Server Infrastructure
* **Domain Controllers**: `SRV-DC01-D` and `SRV-DC02-M` (Domain `sbn.solution`).
* **DNS Service**: Integrated within Active Directory for internal name resolution.
* **File Management & High Availability**:
  * **DFS** Namespaces (`\\sbn.solution\Users\Data`).
  * **DFS Replication** (`Users-Replication`) for automatic synchronization of user directories between both domain controllers.
* **Group Policies (GPO)**:
  * Automated redirection of user folders (*Desktop, Documents, Downloads, Pictures, Music, Videos*).
  * Automatic network drive mapping and corporate desktop wallpaper enforcement.

### ✉️ Unified Communications (Mailcow Mail Server)
* **Architecture**: Containerized deployment based on **Docker**.
* **Features**: SMTP, IMAP, ActiveSync services, **SOGo** Webmail interface, antivirus, and anti-spam protection.
* **Authentication**: LDAP synchronization and Active Directory integration.

### 💼 Enterprise ERP / CRM (Dolibarr)
* **Tech Stack**: Linux server (Debian/Ubuntu), Apache2, PHP, MariaDB database.
* **Features**: Inventory tracking, commercial management, logistics, and financial reporting.
* **File Sharing**: Associated SMB server (**Samba**) for business document storage and sharing.

### 📺 Digital Signage (Xibo)
* **Architecture**: **Xibo CMS** server running under Docker on Ubuntu Server (`192.168.30.14`).
* **Broadcasting**: Management and control of showcase display players via **Xibo Player**.
* **Administration & Media**: Secure media file transfers via **SFTP / FileZilla**.

### 💾 Centralized Storage & Backup (OpenMediaVault NAS)
* **NAS System**: OpenMediaVault on Debian base (`192.168.10.250`).
* **Physical Redundancy**: Volume configured in **RAID 1 (Mirroring)**.
* **SMB/CIFS Shares**: `Direction`, `Atelier`, `Magasin`, and `Wallpaper` folders with group-restricted access (`Administrators`, `Users-A`, `Users-M`, `Users-D`).
* **Extensions (OMV-Extras)**: Integration of FileBrowser and PhotoPrism.
* **Backups**: Automated incremental backups for Dolibarr's MariaDB database and sensitive network files.

---

## 🖥️ Client Workstations & Validation

* **Windows Clients**: Windows 11 Pro joined to the `sbn.solution` domain, tested with full GPO enforcement.
* **Linux Clients**: **Ubuntu Desktop** workstations joined to the Active Directory domain to validate centralized authentication and web service interoperability (Mailcow, Dolibarr, Samba).

---

## 🔒 Security & Compliance

* **Perimeter Security**: Strict inter-VLAN traffic filtering via pfSense rules.
* **Backups**: Daily incremental backup schedule to dedicated NAS.
* **Privacy & GDPR Compliance**: Protection of client records and adherence to least-privilege principles.

---

## 🎓 Project Framework

Project developed as part of the **BTS SIO (Services Informatiques aux Organisations) — SISR Option** (2024–2026 Session).
