# 🚀 Infrastructure Réseau & Services Virtuels — SBN Solution Tech

> **Projet d'Épreuve Technique — BTS SIO (Services Informatiques aux Organisations) Option SISR (Session 2024–2026)**  
> *Documentation technique d'architecture réseau, de virtualisation Proxmox VE et de déploiement de services d'entreprise.*

---

## 📋 Présentation du Projet

Ce projet a pour objectif de concevoir, déployer et administrer une infrastructure réseau d'entreprise complète, 100 % virtualisée et sécurisée pour la société **SBN Solution Tech** (spécialisée dans la vente et la réparation de matériel informatique).

L'ensemble de l'environnement est hébergé sur une plateforme d'hypervision **Proxmox VE** s'appuyant sur un serveur physique **Dell PowerEdge R730**.

---

## 🏗️ Architecture Réseau & Virtualisation

### 1. Hyperviseur & Routage
* **Hyperviseur** : Proxmox VE (basé sur Debian Linux, exploitant KVM pour les VM et LXC pour les conteneurs).
* **Pare-feu / Routeur principal** : Machine virtuelle **pfSense** (FreeBSD) assurant le routage inter-VLAN, le filtrage de sécurité et le service DHCP centralisé.

### 2. Segmentation par VLAN (802.1Q)
Le réseau interne est découpé logiquement en sous-réseaux afin d'assurer la sécurité et l'isolement des flux :
* **VLAN 10 — Direction / Comptabilité** : `192.168.10.0/24`
* **VLAN 20 — Atelier** : `192.168.20.0/24`
* **VLAN 30 — Magasin** : `192.168.30.0/24`
* **LAN Management / Infrastructure** : `192.168.1.0/24` (réseau d'administration pfSense / Proxmox)

### 3. Schéma Logique de l'Infrastructure

```text
                     [ WAN / Accès Internet ]
                                 |
                          (vmbr0 - WAN)
                                 |
                        +-----------------+
                        |   pfSense VM    | (Pare-feu & DHCP Centralisé)
                        +-----------------+
                                 |
                      (vmbr1 - Trunk VLAN-Aware)
                                 |
       +-------------------------+-------------------------+
       |                         |                         |
  [VLAN 10: Direction]     [VLAN 20: Atelier]      [VLAN 30: Magasin]
  192.168.10.0/24          192.168.20.0/24         192.168.30.0/24
   - Active Directory       - Postes Atelier        - Serveur Xibo (CMS)
   - NAS OpenMediaVault     - Serveur Dolibarr       - Postes Caisse & Players
   - Mailcow Mail Server
```

---

## 🛠️ Services & Serveurs Déployés

### 🔐 Active Directory & Infrastructure Windows Server
* **Contrôleurs de Domaine** : `SRV-DC01-D` et `SRV-DC02-M` (Domaine `sbn.solution`).
* **Service DNS** : Intégré à Active Directory pour la résolution de noms interne.
* **Gestion des Fichiers & Haute Disponibilité** :
  * Espace de noms **DFS** (`\\sbn.solution\Users\Data`).
  * **Réplication DFS** (`Users-Replication`) pour la synchronisation automatique des dossiers utilisateurs entre les deux contrôleurs de domaine.
* **Stratégies de Groupe (GPO)** :
  * Redirection automatique des dossiers utilisateurs (*Bureau, Documents, Téléchargements, Images, Musique, Vidéos*).
  * Mappage automatique des lecteurs réseau et application du fond d'écran d'entreprise.

### ✉️ Messagerie Unifiée (Mailcow)
* **Architecture** : Déploiement conteneurisé basé sur **Docker**.
* **Fonctionnalités** : Services SMTP, IMAP, ActiveSync, interface Webmail **SOGo**, protection antivirus et antispam.
* **Authentification** : Synchronisation et intégration avec l'Active Directory via LDAP.

### 💼 ERP / CRM d'Entreprise (Dolibarr)
* **Stack Technique** : serveur Linux (Debian/Ubuntu), Apache2, PHP, base de données MariaDB.
* **Fonctionnalités** : Suivi des stocks, gestion commerciale et suivi logistique et financier.
* **Partage de Fichiers** : Serveur SMB (**Samba**) associé pour le stockage et le partage des documents métier.

### 📺 Affichage Dynamique (Xibo)
* **Architecture** : Serveur **Xibo CMS** sous Docker sur Ubuntu Server (`192.168.30.14`).
* **Diffusion** : Gestion et contrôle des afficheurs en vitrine via **Xibo Player**.
* **Administration & Fichiers** : Transferts sécurisés des médias multimédias via **SFTP / FileZilla**.

### 💾 Stockage Centralisé & Sauvegarde (OpenMediaVault NAS)
* **Système NAS** : OpenMediaVault sur base Debian (`192.168.10.250`).
* **Redondance Physique** : Volume configuré en **RAID 1 (Miroir)**.
* **Partages SMB/CIFS** : Dossiers `Direction`, `Atelier`, `Magasin` et `Wallpaper` avec accès restreints par groupes d'utilisateurs (`Administrators`, `Users-A`, `Users-M`, `Users-D`).
* **Extensions (OMV-Extras)** : Integration de FileBrowser et PhotoPrism.
* **Sauvegardes** : Sauvegardes automatisées et incrémentales de la base MariaDB de Dolibarr et des données sensibles du réseau.

---

## 🖥️ Postes Clients & Validation

* **Clients Windows** : Windows 11 Professionnel intégrés au domaine `sbn.solution`, testés avec application complète des GPO.
* **Clients Linux** : Postes sous **Ubuntu Desktop** joints au domaine Active Directory pour valider la centralisation des accès et l'interopérabilité des services web (Mailcow, Dolibarr, Samba).

---

## 🔒 Sécurité & Conformité

* **Sécurité Périmétrique** : Filtrage strict des flux inter-VLAN via les règles pfSense.
* **Sauvegardes** : Plan de sauvegarde quotidienne incrémentale sur NAS dédié.
* **Confidentialité & RGPD** : Protection des données clients et respect du principe du moindre privilège.
* **Charte Informatique** : Définition des règles d'utilisation des comptes, de la sécurité des mots de passe et engagements de confidentialité.

---

## 🎓 Cadre du Projet

Projet réalisé dans le cadre de la formation **BTS Services Informatiques aux Organisations (SIO) - Option SISR** (Session 2024–2026).
