# 🚀 Documentation Technique d'Infrastructure & Services Virtuels — SBN Solution Tech

> **Projet d'Épreuve Technique — BTS SIO (Services Informatiques aux Organisations) Option SISR**  
> *Guide de déploiement pas à pas, procédures d'installation, architectures réseau, configurations système et schémas.*

---

🌐 **Langues / Languages** : [Français](README.md) | [English](README_EN.md)

## 📋 1. Présentation Générale de l'Entreprise & du Projet

**SBN Solution Tech** est une société spécialisée dans la vente et la réparation de matériel informatique haut de gamme. Pour répondre aux besoins d'organisation et de sécurité de la direction, de l'atelier de réparation et du magasin, une infrastructure d'entreprise complète, 100 % virtualisée et hautement disponible a été conçue.

L'ensemble de l'environnement est virtualisé sur la plateforme **Proxmox VE** hébergée sur un serveur physique **Dell PowerEdge R730** avec administration à distance via **iDRAC**.

```text
                                    [ WAN / Accès Internet ]
                                               |
                                     (vmbr0 - WAN Proxmox)
                                               |
                                    +---------------------+
                                    |  pfSense Firewall   | (Routage Inter-VLAN & DHCP)
                                    +---------------------+
                                               |
                                  (vmbr2 - Trunk VLAN-Aware)
                                               |
          +------------------------------------+------------------------------------+
          |                                    |                                    |
 [VLAN 10 : Direction]                [VLAN 20 : Atelier]                  [VLAN 30 : Magasin]
 192.168.10.0/24                      192.168.20.0/24                      192.168.30.0/24
  - SRV-DC01-D (Active Directory)      - Client PC-0X-A (Win 11)            - SRV-DC02-M (AD Répliqué)
  - NAS OpenMediaVault (RAID 1)        - Serveur Dolibarr (ERP/Samba)       - Serveur Xibo CMS (Docker)
  - Mailcow Mail Server (Docker)                                            - Players Xibo Vitrine
```
![Schéma Logique de l'Infrastructure](images/schema-infrastructure.png)

---

## 🏗️ 2. Plan d'Adressage IP & Cartographie Réseau

### 2.1. Cartographie des VLANs (802.1Q)
| VLAN | Rôle / Service | Réseau CIDR | Passerelle (pfSense) | Plage DHCP / Machines | Réservations d'IP Fixes |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **VLAN 10** | Direction & Compta | `192.168.10.0/24` | `192.168.10.1` | `192.168.10.10–50` | `SRV-DC01-D` (`.200`), `NAS-D` (`.250`) |
| **VLAN 20** | Atelier de Réparation | `192.168.20.0/24` | `192.168.20.1` | `192.168.20.10–50` | `Dolibarr` (`.200`), `MAILCOW` (`.12`) |
| **VLAN 30** | Magasin & Vitrine | `192.168.30.0/24` | `192.168.30.1` | `192.168.30.10–50` | `SRV-DC02-M` (`.200`), `SRV-XIBO-M` (`.14`) |
| **VLAN 1** | Management / Admin | `192.168.1.0/24` | `192.168.1.1` | `192.168.1.10–50` | Proxmox VE (`192.168.1.X`), iDRAC Dell |

### 2.2. Table des Serveurs, Hôtes et Enregistrements DNS (`sbn.solution`)
| Nom d'Hôte | Système d'Exploitation | Adresse IP | Rôle & Services Déployés |
| :--- | :--- | :--- | :--- |
| `SRV-DC01-D` | Windows Server 2022 | `192.168.10.200` | DC Principal, DNS Primaire, Racine DFS `\\sbn.solution\Users` |
| `SRV-DC02-M` | Windows Server 2022 | `192.168.30.200` | DC Secondaire, DNS Secondaire, Réplication AD DS & DFS-R |
| `NAS-D` | OpenMediaVault 8.1 / Debian | `192.168.10.250` | NAS RAID 1, Partages SMB, OMV-Extras (FileBrowser, PhotoPrism) |
| `MAILCOW` | Ubuntu Server 22.04 / Docker | `192.168.20.12` | Suite Mailcow (SMTP/IMAP/SOGo) + Sync LDAP Cron via `s_mailcow` |
| `Dolibarr` | Debian / Apache / MariaDB | `192.168.20.200` | ERP / CRM Commercial + Serveur de fichiers Samba |
| `SRV-XIBO-M` | Ubuntu Server 22.04 / Docker | `192.168.30.14` | Serveur d'Affichage Dynamique Xibo CMS & Transfert SFTP |
| `PC-01-A` à `PC-03-A` | Windows 11 Pro | `192.168.20.15–17` | Postes Clients Atelier (OU `PC-A`) |
| `PC-01-D` à `PC-03-D` | Windows 11 Pro | `192.168.10.20–21` | Postes Clients Direction (OU `PC-D`) |
| `PC-01-M` à `PC-03-M` | Windows 11 Pro | `192.168.30.16–21` | Postes Clients Magasin (OU `PC-M`) |
| `Player-XIBO01/02` | Windows 11 Pro | DHCP (VLAN 30) | Écrans d'affichage en vitrine avec Xibo Player |

---

## 🛠️ 3. Spécifications & Configurations Techniques (Vérifiées sur Captures)

### 3.1. Active Directory DS, Unités d'Organisation & Groupes
* **Domaine Active Directory** : `sbn.solution`
* **Arborescence des Unités d'Organisation (OU)** :
  * `Atelier` ➔ Sous-OU : `PC-A`
  * `Direction` ➔ Sous-OU : `PC-D`
  * `Magasin` ➔ Sous-OU : `PC-M`
* **Groupes de Sécurité & Membres Utilisateurs** :
  * **`Users-A`** (Atelier) : Hugo SIMON, Romain DUB...
  * **`Users-D`** (Direction) : Julien, Lucas, Nithilan
  * **`Users-M`** (Magasin) : Antoine, Maxime
  * **Compte de Service AD** : `s_mailcow` (Synchronisation LDAP Mailcow)

### 3.2. Stratégies de Groupe (GPO) Déployées
1. **`Redirections-Dossiers-Users`** (Liée à la racine `sbn.solution`) :
   * Redirige *Bureau, Documents, Téléchargements, Images, Musique, Vidéos* vers `\\sbn.solution\Users\Data\%USERNAME%`.
2. **`NAS-Mapping-A`** (Liée à l'OU `Atelier`) :
   * Mappe le lecteur `A:` vers `\\NAS-D\Atelier` pour le groupe `Users-A`.
3. **`NAS-Mapping-D`** (Liée à l'OU `Direction`) :
   * Mappe le lecteur `D:` vers `\\NAS-D\Direction` pour le groupe `Users-D`.
4. **`NAS-Mapping-M`** (Liée à l'OU `Magasin`) :
   * Mappe le lecteur `M:` vers `\\NAS-D\Magasin` pour le groupe `Users-M`.

### 3.3. Architecture DFS (Distributed File System) & Réplication
* **Espace de noms DFS** : `\\sbn.solution\Users`
* **Serveurs d'Espace de noms** : `\\SRV-DC01-D.sbn.solution\Users` et `\\SRV-DC02-M.sbn.solution\Users`.
* **Dossier Cible DFS** : `Data` ➔ pointe vers `\\SRV-DC01-D\Users$` et `\\SRV-DC02-M\Users$`.
* **Disque Physique Cible** : Volume dédié `S:\Users-Data` (250 Go).
* **Groupe de Réplication DFS-R** : `Users-Replication` synchronisant en continu les données entre `SRV-DC01-D` et `SRV-DC02-M`.

### 3.4. NAS OpenMediaVault 8.1 (`NAS-D`)
![Architecture de Stockage OpenMediaVault NAS RAID 1](images/omv-storage-raid.png)
* **Informations Système** : Hôte `nas-d.sbn.solution`, IP Fixe `192.168.10.250/24`, passerelle `192.168.10.1`, DNS `192.168.10.200` & `192.168.30.200`.
* **Configuration Disques** :
  * Disque Système : `/dev/sdb` (64 GiB).
  * Disques Données : `/dev/sda` (250 GiB) + `/dev/sdc` (250 GiB) configurés en **RAID 1 (Miroir)**.
  * UUID du volume RAID 1 : `397a23f3-3263-423e-8d58-5f41380dc0bc`.
* **Chemins des Partages SMB/CIFS** :
  * `Atelier` ➔ `/srv/dev-disk-by-uuid-397a23f3-3263-423e-8d58-5f41380dc0bc/Atelier/`
  * `Direction` ➔ `/srv/dev-disk-by-uuid-397a23f3-3263-423e-8d58-5f41380dc0bc/Direction/`
  * `Magasin` ➔ `/srv/dev-disk-by-uuid-397a23f3-3263-423e-8d58-5f41380dc0bc/Magasin/`
  * `Wallpaper` ➔ `/srv/dev-disk-by-uuid-397a23f3-3263-423e-8d58-5f41380dc0bc/Wallpaper/`

### 3.5. Automatisation de la Sauvegarde (Planificateur de Tâches)
* **Nom de la Tâche Windows Server** : `backup-NAS-D-Failover-Automatisation`
* **Auteur / Compte** : `SBN\Administrateur`
* **Déclencheurs** : Au démarrage du système + Tous les jours à 08:00 (répétition toutes les 5 min).
* **Action** : Sauvegarde incrémentale automatisée des dossiers du NAS `NAS-D` (`Atelier`, `Direction`, `Magasin`) et des dossiers DFS vers la cible local `S:\BACKUP\NAS-D` et `S:\BACKUP\Users-Data`.

---

## 🛠️ 4. Procédures de Déploiement Étape par Étape

### 4.1. Virtualisation Proxmox VE & Routage pfSense
![Architecture de Routage et Filtrage pfSense](images/pfsense-routing.png)

#### Configuration des Bridges Proxmox VE :
1. Connectez-vous à Proxmox VE (`https://192.168.1.X:8006`).
2. Conservez `vmbr0` sur l'interface physique `enp4s0` (WAN).
3. Créez `vmbr1` (Bridge Administration).
4. Créez `vmbr2` (`LAN_VLANs`, avec l'option **VLAN aware** activée).

#### Configuration des Sub-interfaces pfSense :
* `em0` (`vtnet0`) ➔ `vmbr0` (WAN Internet)
* `em1` (`vtnet1`) ➔ `vmbr1` (LAN Administration - `10.0.0.1/30`)
* `em2` (`vtnet2`) ➔ `vmbr2` (Trunk VLANs)
  * `em2.10` ➔ `VLAN10` (`192.168.10.1/24`)
  * `em2.20` ➔ `VLAN20` (`192.168.20.1/24`)
  * `em2.30` ➔ `VLAN30` (`192.168.30.1/24`)

---

### 4.2. Services Applicatifs Docker & Messagerie (Mailcow, Dolibarr, Xibo)
![Architecture des Conteneurs Docker](images/docker-services.png)

#### Déploiement de Mailcow et Cron de Synchronisation LDAP :
```bash
# 1. Déploiement de Mailcow Dockerized
cd /opt && sudo git clone https://github.com/mailcow/mailcow-dockerized
cd mailcow-dockerized
./generate_config.sh  # Nom d'hôte: mail.sbn.solution
sudo docker compose pull && sudo docker compose up -d

# 2. Ajout de la tâche Cron de synchronisation AD/LDAP (exécutée chaque minute)
crontab -e
* * * * * cd /opt/mailcow-dockerized && /usr/bin/docker compose exec -T php-fpm-mailcow php /web/inc/scripts/ldap_sync.php > /dev/null 2>&1
```

#### Déploiement de Xibo CMS & Connexion SFTP FileZilla :
* **Déploiement CMS** : Conteneur Docker sur Ubuntu (`192.168.30.14`).
* **Saisie SFTP FileZilla** : Hôte `192.168.30.14`, Port `22`, Utilisateur `xibo` pour le transfert des layouts et médias vers les écrans en vitrine.

---

## 🔒 5. Sécurité, Sauvegardes & Conformité

* **Sécurité Périmétrique** : Filtrage strict inter-VLAN sur pfSense avec politique de déni par défaut.
* **Politique de Sauvegarde** : Exportation quotidienne incrémentale des bases de données et dossiers vers le volume RAID 1 d'OpenMediaVault.
* **Haute Disponibilité** : Tolérance aux pannes matérielles (RAID 1), réplication de domaine AD DS et synchronisation DFS.

---

## 🎓 6. Cadre Académique & Crédits

Projet d'Épreuve Technique réalisé dans le cadre du **BTS Services Informatiques aux Organisations (SIO) - Option SISR** (Session 2024–2026) par **Nithilan SHUBASKARAN**.
