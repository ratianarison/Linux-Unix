
# Configuration des services communs Serveur-Clients : SSH

## 1. Introduction

SSH est le protocole standard pour l'administration sécurisée à distance. SSH permet :

- L'administration centralisée des serveurs depuis n'importe quel client
- L'exécution de commandes à distances (exemple : pour la gestion de NIS et NFS)
- Le transfert sécurisé de fichiers via SCP et SFTP
- La création de tunnels chiffrés pour sécuriser les services.

Cette documentation décrit la mise en place d'une infrastructure SSH permettant une administration fluide et sécurisée entre le serveur et les clients.

## 2. Architecture du laboratoire 

```text
┌─────────────────────────────────────────────────────────────────────────┐
│                          RÉSEAU LOCAL (10.141.153.0/24)                  │
│                                                                         │
│  ┌─────────────────────┐     ┌─────────────────────┐                    │
│  │   DEBIAN-MASTER     │     │    DEBIAN-SLAVE     │                    │
│  │   (Serveur NIS)     │     │   (Serveur NIS)     │                    │
│  │                     │     │                     │                    │
│  │  • IP:10.141.153.245│     │  • IP:10.141.153.246│                    │
│  │  • sshd (port 22)   │     │  • sshd (port 22)   │                    │
│  │  • NIS, NFS, MariaDB│     │  • NIS slave        │                    │
│  └──────────┬──────────┘     └──────────┬──────────┘                    │
│             │                           │                               │
│             └───────────┬───────────────┘                               │
│                         │                                               │
│                         │                                               │
│             ┌───────────┴───────────┐                                   │
│             │                       │                                   │
│  ┌──────────▼──────────┐  ┌─────────▼──────────┐                        │
│  │    KALI CLIENT      │  │   POSTE ADMIN      │                        │
│  │    (Hôte)           │  │   (Optionnel)      │                        │
│  │                     │  │                    │                        │
│  │  • IP:10.141.153.247│  │  • IP:10.141.153.x │                        │
│  │  • ssh client       │  │  • ssh client      │                        │
│  │  • NIS client       │  │                    │                        │
│  └─────────────────────┘  └────────────────────┘                        │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

## 3. Installation des paquets 

### 3.1. Sur le serveur :

```bash
# Mise à jour des dépôts
sudo apt update && sudo apt upgrade -y

# Installation du serveur SSH
sudo apt install openssh-server -y

# Vérification :
dpkg -l | grep openssh-server
```

### 3.2. Sur les clients 

```bash
# Mise à jour des dépôts
sudo apt install && sudo apt upgrade -y

# Installation du client SSH
sudo apt install openssh-client -y

# Vérification :
ssh -V
```

## 4. Configuration du serveur SSH

### 4.1. Fichier de configuration principal

Le fichier de configuration principal du serveur SSH est /etc/ssh/sshd_config.

```bash
# Sauvegarde de la configuration d'origine
sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak

# Édition du fichier de configuration :
sudo nano /etc/ssh/sshd_config
```

## 4.2. Exemple de configuration :

```bash
# /etc/ssh/sshd_config

# Port d'écoute (default is 22)
Port 22

# Protocole (SSH2 uniquement)
Protocol 2

# Adresse d'écoute :
ListenAddress 0.0.0.0
Listen ::

# Authentification
PermitRootLogin prohibit-password # Root uniquement avec clé
PubkeyAuthentication yes  # Authentification par clé)
PasswordAuthentication yes # Authentification par mot de passe

# Sécurité :
MaxAuthTries 3 # Nombre maximal de tentatives
MaxSessions 10 # Sessions maximales
ClientAliveInterval 300 # Keepalive (5mn)
ClientAliveCountMax 2 # Nombre de keepalive avant déconnexion.

# Journalisation
LogLevel INFO
SyslogFacility AUTH

# Forwarding

X11Forwarding no  # désactivé pour sécurité
AllowTcpForwarding yes # Autoriser les tunnels

# SFTP 
Subsystem sftp /usr/lib/openssh/sftp-server

# PAM
UsePAM yes
```

