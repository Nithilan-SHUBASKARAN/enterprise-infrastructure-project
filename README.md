<p align="center">
  <img src="images/logo-sbn.png" width="220" alt="Logo SBN Solution Tech">
</p>

# 🚀 Infrastructure Réseau & Services Virtuels — SBN Solution Tech

> **Projet d'Épreuve Technique — BTS SIO (Services Informatiques aux Organisations) Option SISR (Session 2024–2026)**  
> *Documentation technique d'architecture réseau, de virtualisation Proxmox VE et de déploiement de services d'entreprise.*

---

🌐 **Langues / Languages** : [Français](README.md) | [English](README_EN.md)

---

## 📋 1. Présentation Générale de l'Entreprise & du Projet

**SBN Solution Tech** est une société spécialisée dans la vente et la réparation de matériel informatique haut de gamme. Pour répondre aux besoins d'organisation et de sécurité de la direction, de l'atelier de réparation et du magasin, une infrastructure d'entreprise complète, 100 % virtualisée et hautement disponible a été conçue.

L'ensemble de l'environnement est virtualisé sur la plateforme **Proxmox VE** hébergée sur un serveur physique **Dell PowerEdge R730**.

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

<p align="center">
  <img src="images/schema-infrastructure.png" width="850" alt="Schéma Logique de l'Infrastructure">
</p>

---

## 🏗️ 2. Plan d'Adressage IP & Table de Routage

### 2.1. Cartographie des VLANs (802.1Q)
| VLAN | Rôle / Service | Réseau CIDR | Passerelle (pfSense) | Plage DHCP / Machines | Réservations d'IP Fixes |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **VLAN 10** | Direction & Compta | `192.168.10.0/24` | `192.168.10.1` | `192.168.10.10–50` | Serveurs AD (`.200`), NAS (`.250`), Mail (`.12`) |
| **VLAN 20** | Atelier de Réparation | `192.168.20.0/24` | `192.168.20.1` | `192.168.20.10–50` | Serveur Dolibarr ERP (`.200`), Clients PC-A |
| **VLAN 30** | Magasin & Vitrine | `192.168.30.0/24` | `192.168.30.1` | `192.168.30.10–50` | Serveur AD 2 (`.200`), Serveur Xibo (`.14`), Players |
| **VLAN 1** | Management / Admin | `192.168.1.0/24` | `192.168.1.1` | `192.168.1.10–50` | Proxmox VE (`192.168.1.X`), iDRAC Dell |

### 2.2. Table des Serveurs et Équipements Clés
| Nom d'Hôte | Système d'Exploitation | Adresse IP | Rôle & Services Déployés |
| :--- | :--- | :--- | :--- |
| `SRV-DC01-D` | Windows Server 2022 | `192.168.10.200` | Contrôleur de Domaine Principal, DNS, DFS Root |
| `SRV-DC02-M` | Windows Server 2022 | `192.168.30.200` | Contrôleur de Domaine Secondaire, Réplication AD & DFS |
| `NAS-D` | OpenMediaVault 8.1 / Debian | `192.168.10.250` | Stockage NAS RAID 1, Partages SMB, OMV-Extras |
| `MAILCOW` | Ubuntu Server 22.04 / Docker | `192.168.20.12` | Suite de Messagerie Mailcow (SMTP/IMAP/SOGo) + Sync LDAP |
| `Dolibarr` | Debian / Apache / MariaDB | `192.168.20.200` | ERP / CRM Commercial + Serveur de fichiers Samba |
| `SRV-XIBO-M` | Ubuntu Server 22.04 / Docker | `192.168.30.14` | Serveur d'Affichage Dynamique Xibo CMS & SFTP |
| `Player-XIBO0X` | Windows 11 Pro | DHCP (VLAN 30) | Écrans d'affichage en vitrine avec Xibo Player |

---

## 🛠️ 3. Guide de Déploiement Étape par Étape

### 3.1. Virtualisation Proxmox VE & Routage pfSense

<p align="center">
  <img src="images/pfsense-routing.png" width="850" alt="Architecture de Routage et Filtrage pfSense">
</p>

#### Étape A : Configuration des Bridges Réseau sous Proxmox VE
1. Accédez à l'interface web Proxmox (`https://192.168.1.X:8006`).
2. Allez dans **Datacenter** > **Noeud Proxmox** > **System** > **Network**.
3. Conservez `vmbr0` connecté à l'interface physique `enp4s0` (WAN).
4. Créez un bridge d'administration **`vmbr1`** (Linux Bridge, Autostart coché).
5. Créez le bridge principal des VLANs **`vmbr2`** (Nom : `LAN_VLANs`, cocher **VLAN aware**).
6. Cliquez sur **Apply Configuration**.

#### Étape B : Configuration de pfSense et Routage Inter-VLAN
1. Assignez les 3 interfaces virtuelles à la VM pfSense :
   * `em0` / `vtnet0` -> `vmbr0` (WAN Internet)
   * `em1` / `vtnet1` -> `vmbr1` (LAN Administration - `10.0.0.1/30`)
   * `em2` / `vtnet2` -> `vmbr2` (LAN Trunk VLANs)
2. Dans l'interface pfSense (**Interfaces** > **VLANs**), ajoutez les sub-interfaces sur `em2` :
   * Tag `10` -> `VLAN10` (`192.168.10.1/24`)
   * Tag `20` -> `VLAN20` (`192.168.20.1/24`)
   * Tag `30` -> `VLAN30` (`192.168.30.1/24`)
3. **Règles Pare-feu (Firewall Rules)** :
   * Bloquer par défaut le flux inter-VLAN.
   * Autoriser spécifiquement les flux AD entre VLAN 10 et 30 (`LDAP 389/636`, `Kerberos 88`, `RPC 135`, `DNS 53`).

---

### 3.2. Active Directory, DFS & Stratégies de Groupe (GPO)

<p align="center">
  <img src="images/ad-dfs-architecture.png" width="750" alt="Architecture Active Directory & DFS">
</p>

#### Étape A : Déploiement et Réplication des Contrôleurs de Domaine
1. Installez le rôle **AD DS** et **DNS** sur `SRV-DC01-D` et créez la forêt **`sbn.solution`**.
2. Joignez le serveur secondaire `SRV-DC02-M` au domaine `sbn.solution` et promouvez-le en contrôleur de domaine répliqué.

##### Captures de configuration IP (Côte à côte, format compact 700x800) :
| SRV-DC01-D (VLAN 10 — Direction) | SRV-DC02-M (VLAN 30 — Magasin) |
| :---: | :---: |
| <img src="images/captures/ip-dc01.png" width="420" alt="Configuration IP SRV-DC01-D"> | <img src="images/captures/ip-dc02.png" width="420" alt="Configuration IP SRV-DC02-M"> |

##### Console Active Directory, DNS & Groupes :

**Console Utilisateurs et Ordinateurs Active Directory** :
![Console Utilisateurs & Ordinateurs Active Directory](images/captures/ad-users-pcs.png)

**Gestionnaire DNS & Zones de Recherche Directe** :
![Gestionnaire DNS & Zones de Recherche Directe](images/captures/dns-manager.png)

**Gestion des Groupes & Utilisateurs AD / OMV** :
![Gestion des Groupes & Utilisateurs AD](images/captures/ad-groups-omv.png)

#### Étape B : Configuration du DFS & Stratégies de Groupe (GPO)
1. **DFS (Distributed File System)** :
   * Racine de l'espace de noms : `\\sbn.solution\Users`
   * Dossier cible : `Data` sur disque dédié `S:` (250 Go)
   * Groupe de réplication : `Users-Replication` synchronisé entre DC01 et DC02.

##### Captures DFS & Redirection GPO (Affichage vertical grand format) :

**Espace de Noms & Racine DFS (`\\sbn.solution\Users`)** :
![Espace de Noms DFS](images/captures/dfs-users.png)

**Stratégie de Groupe — Redirection des Dossiers Utilisateurs (GPO)** :
![Redirection de Dossiers GPO](images/captures/gpo-redirection.png)

2. **GPOs Déployées** :
   * **Redirections-Dossiers-Users** : Redirige *Documents, Bureau, Images, Téléchargements* vers `\\sbn.solution\Users\Data\%USERNAME%`.
   * **NAS-Mapping-A** : Mappe le lecteur `A:` (`\\NAS-D\Atelier`) pour le groupe `Users-A`.
   * **NAS-Mapping-D** : Mappe le lecteur `D:` (`\\NAS-D\Direction`) pour le groupe `Users-D`.
   * **NAS-Mapping-M** : Mappe le lecteur `M:` (`\\NAS-D\Magasin`) pour le groupe `Users-M`.

#### Étape C : Stratégies de Personnalisation & Fond d'Écran d'Entreprise (GPO Wallpapers)

Application automatique du fond d'écran d'entreprise adapté à chaque service via GPO :

**Fond d'écran — Service Direction (`wallpaper-direction.png`)** :
![Fond d'écran Direction](images/captures/wallpaper-direction.png)

**Fond d'écran — Service Atelier (`wallpaper-atelier.png`)** :
![Fond d'écran Atelier](images/captures/wallpaper-atelier.png)

**Fond d'écran — Service Magasin (`wallpaper-magasin.png`)** :
![Fond d'écran Magasin](images/captures/wallpaper-magasin.png)

---

### 3.3. NAS OpenMediaVault (RAID 1 & Partages SMB)

<p align="center">
  <img src="images/omv-storage-raid.png" width="850" alt="Architecture de Stockage OpenMediaVault NAS RAID 1">
</p>

1. **Architecture Disque** :
   * Disque Système : `/dev/sdb` (64 Go).
   * Volume RAID 1 (Miroir) : `/dev/sda` (250 Go) + `/dev/sdc` (250 Go) -> Volume `397a23f3-3263-423e-8d58-5f41380dc0bc`.
2. **Groupes et Partages SMB/CIFS** :
   * Groupes créés : `Administrators`, `Users-A` (Atelier), `Users-D` (Direction), `Users-M` (Magasin).
   * Dossiers partagés : `Atelier`, `Direction`, `Magasin`, `Wallpaper`.
3. **Extensions OMV-Extras** : Activation de **FileBrowser** et **PhotoPrism**.

##### Captures de Gestion des Disques & Tâches de Sauvegarde Automatisées :

**Gestion des Disques et Partitions** :
![Gestion des Disques](images/captures/Disques.png)

**Tâche Planifiée de Sauvegarde Automatique** :
![Backup Task](images/captures/backup-task.png)

---

### 3.4. Services Applicatifs Docker (Mailcow, Dolibarr, Xibo)

<p align="center">
  <img src="images/docker-services.png" width="850" alt="Architecture des Conteneurs Docker">
</p>

#### A. Messagerie Unifiée Mailcow (Docker & LDAP)

#### Procédure d'installation en ligne de commande :
```bash
# 1. Mise à jour système et dépendances
sudo apt update && sudo apt upgrade -y
sudo apt install curl git python3-pip apt-transport-https ca-certificates software-properties-common -y

# 2. Configuration du nom d'hôte FQDN
sudo hostnamectl set-hostname mail.sbn.solution

# 3. Installation de Docker et Docker Compose
curl -sSL https://get.docker.com/ | sh
sudo systemctl enable --now docker

# 4. Clonage du dépôt officiel Mailcow
cd /opt
sudo git clone https://github.com/mailcow/mailcow-dockerized
cd mailcow-dockerized

# 5. Génération de la configuration et déploiement
./generate_config.sh  # Saisir mail.sbn.solution
sudo docker compose pull
sudo docker compose up -d
```

#### Synchronisation Instantanée avec Active Directory (Cron) :
Pour créer automatiquement les boîtes mails dès qu'un utilisateur est ajouté dans l'AD Windows :
```bash
crontab -e
# Ajouter la tâche planifiée exécutée chaque minute :
* * * * * cd /opt/mailcow-dockerized && /usr/bin/docker compose exec -T php-fpm-mailcow php /web/inc/scripts/ldap_sync.php > /dev/null 2>&1
```

---

### 3.5. ERP / CRM Dolibarr

1. **Stack Web** : Linux Debian, Apache2, PHP, MariaDB.
2. **Fonctionnalités** : Suivi commercial, gestion des stocks et de la facturation.
3. **Partage SMB & Sauvegardes** : Intégration de Samba pour les pièces jointes et script de dump automatisé de la base MariaDB vers le NAS `NAS-D`.

---

### 3.6. Affichage Dynamique Xibo & Transfert SFTP FileZilla

#### Déploiement de Xibo CMS sous Docker :
```bash
mkdir /opt/xibo && cd /opt/xibo
wget -O xibo-docker.tar.gz https://xibosignage.com/api/downloads/cms
tar --strip-components=1 -zxvf xibo-docker.tar.gz
cp config.env.template config.env
nano config.env  # Configurer MYSQL_PASSWORD
docker compose up -d
```

**Interface Web & Administration Xibo CMS (`xibo-page.png`)** :
![Xibo CMS Interface Web](images/captures/xibo-page.png)

#### Connexion SFTP FileZilla pour le transfert de médias :
* **Hôte** : `192.168.30.14` (Serveur Xibo CMS)
* **Protocol** : SFTP (SSH)
* **Port** : `22`
* **Utilisateur** : `xibo`

---

## 🔒 4. Sécurité, Sauvegardes & Conformité

* **Sécurité Périmétrique** : Filtrage strict inter-VLAN sur pfSense avec politique de déni par défaut.
* **Politique de Sauvegarde** : Exportation quotidienne incrémentale des bases de données et dossiers vers le volume RAID 1 d'OpenMediaVault.
* **Haute Disponibilité** : Tolérance aux pannes matérielles (RAID 1), réplication de domaine AD DS et synchronisation DFS.
* **Confidentialité & RGPD** : Protection des données clients et respect du principe du moindre privilège.
* **Charte Informatique** : Définition des règles d'utilisation des comptes, de la sécurité des mots de passe et engagements de confidentialité.

---

## 🎓 5. Cadre Académique & Crédits

Projet d'Épreuve Technique réalisé par **Nithilan SHUBASKARAN** dans le cadre du **BTS Services Informatiques aux Organisations (SIO) - Option SISR** (Session 2024–2026).
