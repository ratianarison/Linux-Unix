
# SSH (Secure Shell)

## 17.1. Présentations de SSH

SSH est un protocole de communication sécurisé qui permet d'établir une connexion chiffré entre un client et un serveur. Il remplace les protocles non sécurisés comme telnet.

**Objectif** : Administration à distance sécurisée des machines, transfert de fichiers sécurisé, et tunnels chiffrés.

**Principales fonctionnalités**

- Connexion à distane : ssh user@host
- Transfert de fichiers : scp, rsync, sftp
- Tunnels chiffrés : redirection de ports
- Forwarding X11 : Affichage d'applications graphiques distantes

## 17.2. Services et Démons SSH

| Composant | Rôle | Paquet |
|-----------|------|--------|
| sshd | Serveur SSH (écoute les connexions) | openssh-server |
| ssh | Client SSH (se connecte aux serveurs) | openssh-client |
| ssh-keygen | Génération des clés SSH | openssh-client |
| ssh-agent | Gestionnaire d'agent d'authentification | openssh-client |
| scp/sftp | Transfert sécurisé de fihiers | openssh-client |


## 17.3. Architecture du Projet


```text
                    [Serveur SSH]
                   Debian-Master
                   192.168.1.10
                   (sshd actif)
                         │
        ┌────────────────┼────────────────┐
        │                │                │
[Admin]            [Admin]           [Admin]
Hôte Kali       Debian-Slave      Debian-Client
192.168.1.13     192.168.1.11      192.168.1.12
(ssh client)    (ssh client)      (ssh client)
```


## 17.4. Installation des Paquets 


### Sur le serveur 


```bash
# Mise à jour
sudo apt update

# Installation du serveurs SSH
sudo apt install openssh-server openssh-client

# Vérification
dpkg -l | grep openssh
systemctl status ssh
```

### Sur les clients 

```bash
# Mise à jour des dépôts
sudo apt update
sudo apt upgrade

# Les clients n'ont besoin que du client SSH
sudo apt install openssh-client

# Vérification
ssh -V # affiche la version
```

## 17.5. Configuration du serveur SSH

### Fichier principal : <mark>*/etc/ssh/sshd_config*</mark>

<details>
  <summary>/etc/ssh/sshd_config</summary>

  Le fichier /etc/ssh/sshd_config est le fichier de configuration principal du serveur SSH (SSH daemon). Il contrôle comment les clients peuvent se connecter, quelles méthodes d'authentification sont autorisées, et quelles options de sécurtié sont appliquées.

  **Emplacement**

  ```bash
  /etc/ssh/sshd_config  # fichier de configuration du serveur -> configure qui accepte les connexions
  /etc/ssh/ssh_config   # fichier de configuration du client -> configure qui initie les connexions
  ```

  **Structure du fichier**

  ```bash
  # Format : Option Valeur
  Port 22
  PermitRootLogin prohibit-password
  PasswordAuthentication yes
  PubkeyAuthentication yes
  ```

  **Options essentielles**

  ```bash
  # Port d'écoute (default is 22)
  Port 22
  # On peut changer pour améliorer la sécurité

  # Adresses IO d'écoute (toutes par défaut)
  ListenAdress 0.0.0.0  #IPv4
  ListenAddress ::   #IPv6

  # Version du protocole (SSH2 seulement recommandé)
  Protocol 2
  ```

  **Authentification**

  ```bash
  # Autoriser la connexion root
  PermitRootLogin prohibit-password # Root avec clé seulement
  # PermitRootLogin no  # Interdit totalement root
  # PermitRootLogin yes  # Autorise root

  # Authentification par mot de passe
  PasswordAuthentication yes   # Autoriser les mots de passe
  # PasswordAuthentication no # Désactiver (plus sécurisé)

  # Authentification par clé publique
  PubkeyAuthentication yes  # Autoriser les clés SSH

  # Authentification par clé GSSAPI
  GSSAPIAuthentication yes

  # Authentification par challenge (multi-facteurs)
  ChallengeResponseAithentication no
  ```

**Sécurité**

```bash
# Empêcher l'accès à des utilisateurs spécifiques
DenyUsers root baduser
DenyGroups badgroup

# Autoriser uniquement certains utilisateurs
AllowUsers x admin user1 user2
AllowGroups ssh-users

# Temps d'inactivité avant déconnexion (secondes)
ClientAliveInterval 300
ClientAliveCountMax 2

# Nombre maximum de tentatives d'authentification
MaxAuthTries 3

# Limiter les connexions simultanées
MaxSessions 1
MaxStartups 10:30:60 # 10 Connexions normales, puis 30% drop, max 60
```

**Journaux et débogage**

```bash
# Niveau de logs (QUIET, FATAL, ERROR, INFO, VERBOSE, DEBUG)
LogLevel INFO

# Fichier de logs (souvent géré par syslog
SyslogFacility AUTH
```

<details>
<summary>LogLevel INFO</summary>
  
  Ces deux directives contrôlent comment et où le serveur  SSH enregistre ces logs.
  Une bonne configuration de journalisation est essentielle pour la surveillance, le débogage et la détection d'attaques.

  **LogLevel - Niveau de détail des logs**

  LogLevel détermine la quantité d'informations que le serveur SSH écrit dans les logs.

  Hiérarchie des niveaux (du plus silencieux au plus verbeux)

  ```text

  QUIET   → Aucun log (sauf erreurs critiques)
    ↓
  FATAL   → Erreurs fatales seulement
    ↓
  ERROR   → Erreurs uniquement
    ↓
  INFO    → Informations normales (défaut)
    ↓
  VERBOSE → Informations détaillées
    ↓
  DEBUG   → Informations de débogage (très verbeux)  
  DEBUG1  → Niveau 1 de débogage
  DEBUG2  → Niveau 2 (encore plus détaillé)
  DEBUG3  → Niveau 3 (maximum de détails)  
  ```

  Exemples de logs par niveau

  **INFO (défaut)**

  ```bash
  # Connexion réussie
  Accepted password for x from 
  ```
</details>

</details>
