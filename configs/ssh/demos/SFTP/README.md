# Serveur de transfert et sauvegarde sécurisé

## Contexte 

Une entreprise possède un serveur Debian central qui sert à :

- Recevoir les fichiers utilisateurs
- stocker les projets
- effectuer des sauvegardes automatiques

Les administrateurs utilisent SCP pour les sauvegardes et les employés utilisent SFTP pour déposer leurs fichiers.

Pour renforcer la sécurité, l'entreprise applique les règles suivantes :

- Les utilisateurs SFPT sont limitée à leur dossier
- Ils ne peuvent pas accéder au reste du système
- Les sauvegardes sont automatiquement via cron
- Tous les transferts sont chiffrés via SSH

## Architecture du système

```text
Administrateur (Kali)
        │
        │ SCP (sauvegarde)
        ▼
Serveur Debian
        │
        ├── /srv/backups
        │
        ├── /srv/sftp
        │      ├── user1
        │      ├── user2
        │      └── user3
        │
        └── /srv/scripts
```
## Utilisateurs du système

| Utilisateur | Rôle           | Accès           |
| ----------- | -------------- | --------------- |
| admin       | administrateur | SSH + SCP       |
| user1       | employé        | SFTP uniquement |
| user2       | employé        | SFTP uniquement |
| user3       | employé        | SFTP uniquement |
| guest       | invité         | refusé          |
| test        | test           | refusé          |


## Création des dossiers SFTP

**Créer le dossier principal**

```bash
sudo mkdir -p /srv/sftp
```

**Créer les dossiers utilisateurs :**

```bash
sudo mkdir /srv/sftp/user1
sudo mkdir /srv/sftp/user2
sudo mkdir /srv/sftp/user3
```

**Permissions**

```bash
sudo chown root:root /srv/sftp
sudo chmod 755 /srv/sftp
```

Puis : 

```bash
sudo chown user1:sftpusers /srv/sftp/user1
sudo chown user2:sftpusers /srv/sftp/user2
sudo chown user3:sftpusers /srv/sftp/user3
```

## Configuration SFTP Jail

Modification : /etc/ssh/sshd_config

Configuration :

```bash
Subsystem sftp internal-sftp

Match Group sftpusers
ChrootDirectory /srv/sftp
ForceCommand internal-sftp
X11Forwarding no
AllowTcpForwarding no
```

## Sauvegarde automatique avec SCP

Les administrateurs doivent sauvegarder les données importantes.

Sur la machine de l'administrateur : 

Créer un dossier sauvegarde : /home/admin/backups

Script du sauvegarde :

```bash
nano backups.sh
```

Script : 

```bash
#!/bin/bash

DATE=$(date +%F)

scp -r /home/admin/data admin@192.168.1.10:/srv/backups/backup-$DATE
```

Rendre exécutable :

```bash
chmod +x bakcups.sh
```

## Automatisation avec cron

Ouvrir cron :

```bash
crontab -e
```
Ajouter :

```bash
0 2 * * * /home/admin/backup.sh
# Temps : 0 = 0 minute, 2 = 2 heures, * = chaque jour
# Donc : sauvegarde tous les jours à 2h00
```

## Exemple de transfert SFTP

Connexion : 

```bash
sftp user1@server
```

Envoyer un fichier : 

```bash put rapport.pdf```

Télécharger : 

```bash 
get document.txt
```

Lister :

```bash 
ls
```

## Exemple de transfert SCP administrateur 

Envoyer un projet : 

```bash 
scp -r projet admin@server:/serv/backups
```

