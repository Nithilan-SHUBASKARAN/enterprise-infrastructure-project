# 🚀 Infrastructure Réseau & Services Virtuels — SBN Solution Tech

> **Projet d'Épreuve Technique — BTS SIO (Services Informatiques aux Organisations) Option SISR (Session 2024–2026)**  
> *Guide de déploiement pas à pas, architectures réseau, configurations système, procédures de sécurité et schémas d'infrastructure.*

---

🌐 **Langues / Languages** : [Français](README.md) | [English](README_EN.md)

---

## 📋 1. Présentation Générale de l'Entreprise & du Projet

**SBN Solution Tech** est une société spécialisée dans la vente et la réparation de matériel informatique haut de gamme. Pour répondre aux besoins d'organisation et de sécurité de la direction, de l'atelier de réparation et du magasin, une infrastructure d'entreprise complète, 100 % virtualisée et hautement disponible a été conçue.

L'ensemble de l'environnement est virtualisé sur la plateforme **Proxmox VE** s'appuyant sur un serveur physique **Dell PowerEdge R730**.

### 🏗️ Schéma Logique Global de l'Infrastructure

![Schéma Logique de l'Infrastructure](images/schema-infrastructure.png)

---

## 🏗️ 2. Plan d'Adressage IP & Segmentation Réseau

### 2.1. Cartographie des VLANs (802.1Q)
| VLAN | Rôle / Service | Réseau CIDR | Passerelle (pfSense) | Plage DHCP / Pool | Réservations d'IP Fixes |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **VLAN 10** | Direction & Compta | `192.168.10.0/24` | `192.168.10.1` | `192.168.10.10–50` | Serveurs AD (`.200`), NAS (`.250`), Mail (`.12`) |
| **VLAN 20** | Atelier de Réparation | `192.168.20.0/24` | `192.168.20.1` | `192.168.20.10–50` | Serveur Dolibarr ERP (`.200`), Clients PC-A |
| **VLAN 30** | Magasin & Vitrine | `192.168.30.0/24` | `192.168.30.1` | `192.168.30.10–50` | Serveur AD 2 (`.200`), Serveur Xibo (`.14`), Players |
| **VLAN 1** | Management / Admin | `192.168.1.0/24` | `192.168.1.1` | `192.168.1.10–50` | Proxmox VE (`192.168.1.X`), iDRAC Dell |

### 2.2. Table des Serveurs & Équipements
| Nom d'Hôte | Système d'Exploitation | Adresse IP | Rôle & Services Déployés |
| :--- | :--- | :--- | :--- |
| `SRV-DC01-D` | Windows Server 2022 | `192.168.10.200` | Contrôleur de Domaine Principal, DNS, DFS Root |
| `SRV-DC02-M` | Windows Server 2022 | `192.168.30.200` | Contrôleur de Domaine Secondaire, Réplication AD & DFS |
| `NAS-D` | OpenMediaVault 8.1 / Debian | `192.168.10.250` | Stockage NAS RAID 1, Partages SMB, OMV-Extras |
| `MAILCOW` | Ubuntu Server 22.04 / Docker | `192.168.20.12` | Messagerie Mailcow (SMTP/IMAP/SOGo) + Sync LDAP |
| `Dolibarr` | Debian / Apache / MariaDB | `192.168.20.200` | ERP / CRM Commercial + Serveur de fichiers Samba |
| `SRV-XIBO-M` | Ubuntu Server 22.04 / Docker | `192.168.30.14` | Serveur d'Affichage Dynamique Xibo CMS & SFTP |
| `Player-XIBO0X` | Windows 11 Pro | DHCP (VLAN 30) | Écrans d'affichage en vitrine avec Xibo Player |

---

## 🛠️ 3. Déploiement Détaillé & Configurations Pas à Pas

### 3.1. Virtualisation (Proxmox VE) & Routage (pfSense)

![Architecture de Routage et Filtrage pfSense](images/pfsense-routing.png)

#### Procédure de configuration pfSense :
1. Déclaration des sous-interfaces VLAN sur `vtnet1` (VLAN 10, 20, 30).
2. Configuration des plages DHCP respectives pour chaque VLAN.
3. Règle de filtrage inter-VLAN : Déni par défaut avec autorisations explicites (flux Active Directory, DNS et SMB).

---

### 3.2. Active Directory, Service DNS & Synchronisation DFS

![Architecture Active Directory et Réplication DFS](images/ad-dfs-architecture.png)

#### A. Paramétrage Réseau des Contrôleurs de Domaine (IPv4 & DNS)

![Configuration IP SRV-DC01-D](images/captures/ip-dc01.png)

![Configuration IP SRV-DC02-M](images/captures/ip-dc02.png)

#### B. Console de Gestion Active Directory (Utilisateurs, Groupes & OUs)

![Active Directory Users and Computers](images/captures/ad-users-pcs.png)

![Gestionnaires DNS & Zones Directes](images/captures/dns-manager.png)

![Gestionnaires de Groupes AD](images/captures/ad-groups-omv.png)

#### C. Espace de Noms DFS & Réplication Inter-Serveurs

![Espace de Noms DFS Data](images/captures/dfs-namespace-data.png)

![Espace de Noms DFS Users](images/captures/dfs-users.png)

![Statut de Réplication DFS](images/captures/dfs-replication-status.png)

![Partages Réseau DFS BKP](images/captures/dfs-backup-folders.png)

#### D. Stratégies de Groupe (GPO) & Redirection de Dossiers

![Configuration GPO Redirection Dossiers](images/captures/gpo-redirection.png)

* **Redirections-Dossiers-Users** : Redirige *Documents, Bureau, Images, Téléchargements* vers `\\sbn.solution\Users\Data\%USERNAME%`.
* **NAS-Mapping-A** : Mappe le lecteur `A:` (`\\NAS-D\Atelier`) pour le groupe `Users-A`.
* **NAS-Mapping-D** : Mappe le lecteur `D:` (`\\NAS-D\Direction`) pour le groupe `Users-D`.
* **NAS-Mapping-M** : Mappe le lecteur `M:` (`\\NAS-D\Magasin`) pour le groupe `Users-M`.

---

### 3.3. Stockage Centralisé OpenMediaVault (NAS RAID 1 & SMB)

![Architecture de Stockage OpenMediaVault NAS RAID 1](images/omv-storage-raid.png)

#### Captures de validation OpenMediaVault :

![Informations Système OMV](images/captures/system-info.png)

![Gestion des Disques Physiques OMV](images/captures/disk-management.png)

![Interfaces Réseau IPv4 OMV](images/captures/interfaces-reseau-ipv4.png)

![Interfaces Réseau DNS OMV](images/captures/interfaces-reseau-dns.png)

![Partages SMB CIFS OMV](images/captures/shared-folders.png)

![Résolution DNS NAS vers SRVDC02](images/captures/nas-hostname-resolution.png)

---

### 3.4. Services Applicatifs Docker (Mailcow, Dolibarr, Xibo)

![Architecture des Conteneurs Docker](images/docker-services.png)

#### A. Messagerie Unifiée Mailcow (Docker & LDAP)
* **FQDN** : `mail.sbn.solution`
* **Synchronisation LDAP AD** : Tâche cron exécutée chaque minute pour intégrer les utilisateurs Active Directory.

```bash
# Commande de déploiement Mailcow
cd /opt && sudo git clone https://github.com/mailcow/mailcow-dockerized
cd mailcow-dockerized && ./generate_config.sh
sudo docker compose up -d
```

#### B. ERP / CRM Commercial Dolibarr
* **Stack Web** : LAMP (Debian, Apache2, MariaDB, PHP).
* **Partage SMB** : Dossiers de pièces jointes partagés pour l'atelier.

#### C. Affichage Dynamique Xibo CMS
* **Hôte** : `SRV-XIBO-M` (`192.168.30.14`).
* **Diffusion** : Webmail / CMS sous Docker et diffusion vers les players vitrine.

---

## 💾 4. Automatisation des Sauvegardes & Planificateur de Tâches

![Tâche Planifiée de Sauvegarde Automatisée](images/captures/backup-task.png)

![Historique d'Exécution du Planificateur](images/captures/planificateur-tache-history.png)

![Arborescence des Dossiers de Sauvegarde](images/captures/backup-folders-tree.png)

![Partage Réseau Restauration Users Data](images/captures/dossier-partage-users-data.png)

---

## 🔒 5. Sécurité, Sauvegardes & Conformité

* **Sécurité Périmétrique** : Filtrage strict inter-VLAN sur pfSense avec politique de déni par défaut.
* **Haute Disponibilité** : Tolérance aux pannes matérielles (RAID 1), réplication Active Directory et synchronisation DFS-R.
* **Confidentialité & RGPD** : Protection des données clients et principe du moindre privilège.

---

## 🎓 6. Cadre Académique & Crédits

Projet réalisé dans le cadre de la formation **BTS Services Informatiques aux Organisations (SIO) - Option SISR** (Session 2024–2026) par **Nithilan SHUBASKARAN**.
