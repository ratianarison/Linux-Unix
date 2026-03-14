# Projet d'Administration Système UNIX

## 📋 Description du Projet 
Mise en place d'un réseau UNIX complet avec une architecture virtualisée sous VirtualBox, comprenant des services réseau essentiels, une gestion centralisée des utilisateurs et des mécanismes de sécurité avancés.

## 🖥️ Infrastructure Matérielle et Logicielle

### Architecture de virtualisation

-**Hyperviseur**: Virtualbox

-**Mode Réseau**: Bridge adapter pour toutes les machines virtuelles

-**Hôte Physique**:Kali Linux (machine hôte)

### Machine Virtuelles

| Machine | Rôle | OS | Interface Graphique|
|---------|------|----|---------------|
| **Debian-Master** | Serveur principal (NIS Master, NFS, DNS, LDAP, SAMBA, etc.) | Debian | Non |
| **Debian-Slave** | Serveur secondaire (NIS Slave, DNS Slave, backup) | Debian | Non |
| **Debian-Client** | Client pour tester les services | Debian | Oui |
| **ParrotOS** | Machine de test/sécurité/audit | ParrotOS | Oui |

## 🎯 Objectifs du Projet

### 1. Services d'Annuaire et d'Authentification

-**NIS(Network Information Service)** - Gestion centralisée des utilisateurs (Master/Slave)

-**LDAP (Lightweight Directory Access Control)** - Annuaire centralisé pour l'authentification

### 2. Partage de Fichiers et Stockage

-**NFS (Network File System)** - Partage de fichiers entre machines UNIX

-**SAMBA** - Interopérabilité avec systèmes Windows

### 3. Services Réseau Fondamentaux 

-**DNS (Domain Name System)** - Résolution de noms (Master/Slave)

-**DHCP (Dynamic Host Configuration Protocol)** - Attribution automatique des adresses IP

-**NTP (Network Time Protocol)** - Synchronisation temporelle

### 4. Services de Communication

-**SMTP/POP/IMAP** - Serveur de messagerie complet

-**SSH** - Accès sécurisé à distance

### 5. Sécurité et Surveillance

-**Pare-feu** - Filtrage réseau et sécurité

-**Syslog** - Journalisation centralisée 

-**Chiffrement** - Sécurisation des communications

-**Système de sécurisation des données** - Politiques et mécanismes

### 6. Services d'Impression et Automatisation

-**CUPS**- Système d'impression

-**cron** - Planification de tâches

### 7. Administration et Maintenance

-**Sauvegarde et Restauration** - Protection des données

-**Réseau et connectivité** - Configuration et dépannage

## Structure du Projet 
