# Configuratoin des services communs Client - Serveur : SSH

## Configuration du serveur 

### Installation des paquets :

```bash
# Mise à jour desdépôts
sudo apt update && sudo apt upgrade -y

# Installation du serveur SSH
sudo apt install openssh-server openssh-client

# Vérification
dpkg -l | grep openssh
sudo systemctl status ssh
```

### Configuration générale - fichier `/etc/ssh/sshd_config`

**Configuration réseau du serveur**

```bash
      # CONFIGURATION RÉSEAU DU SERVEUR SSH

# Port SSH
# Par défaut SSH écoute sur le port 22.
# Bonne pratique sécurité :
Port 2222
# Réduit les scans automatiques et diminue les attaque brute-force

# Adresse d'écoute :
ListenAddress 0.0.0.0 # Toutes les interfaces réseau
ListenAddress 10.141.153.249 # Le serveur n'écoute que sur cette interface

# Protocole SSH
Protocol 2 # SSH v1 est insecure et obsolète, il faut toujours utiliser la version 2


      # CONFIGURATION DE SÉCURITÉ

# Interdire connexion root :
PermitRootLogin no
# Alternative :
PermitRootLogin prohibit-password # root uniquement par clé SSH

# Limiter les tentatives de connexions
MaxAuthTries 3 # si l'utilisateur échoue 3 fois, connexion coupée

# Délai de connexion :
LoginGraceTime 15   # Le client a 15 secondes pour s'authentifier

# Nombre de sessions simultanées
MaxSessions 5 # Empêche un utilisateur d'ouvrir trop de sessions SSH simultanées


        # METHODES D'AUTHENTIFICATION

# Authentification par mot de passe
PasswordAuthentication yes # "no" pour un serveur sécurisé

# Authentification par clé publique
PubkeyAuthentication yes # SSH va vérifier ~/.ssh/authorized_keys

# Authentification combinée
AuthenticationMethods publickey,password # Éxiger clé + mot de passe


       # GESTION DES UTILISATEURS AUTORISÉS

# Autoriser certains utilisateus :
AllowUsers admin user1 user2 # tous les autres sont réfusés.

# Bloquer certains utilisateurs :
DenyUsers guest test user3

# Bloquer groupes
DenyGroups banned 


        # GESTION DES CLÉS SSH

# Fichier des clés autorisées
AuthorizedKeysFile .ssh/authorized_keys     # Par défaut, SSH regarde /home/user/.ssh/authorized_keys

# Remarque : le dossier .ssh n'est créé que si PasswordAuthentication == no

# Vérification des permissions
StrictModes yes
# SSH vérifie ~/.ssh/authorized_keys, si permissions incorrectes : connexions réfusée

# Emplacement des clés système :
HostKey /etc/ssh/ssh_host_rsa_key
HostKey /etc/ssh/ssh_host_ed25519_key
# Ces clés identifient le serveur SSH

         # FORWARDINF ET TUNNELING

# Autoriser port forwarding
AllowTcpForwarding yes # permet ssh -L et ssh -R

# "no" pour sécurité

# X11 Forwarding
X11Forwarding yes
# Permet d'exécuter des applications graphiques via SSH

# Agent forwarding
AllowAgentForwarding yes    # Permet d'utiliser la clé locale sur un serveur distant


         # GESTION DES SESSIONS SSH

# Timeout session inactive
ClientAliveInterval 300      # 5mn sans activité -> ping SSH
ClientAliveCountMax 2        # après 2 échecs -> session coupée

# Message de connexion
Banner /etc/issue.net  # insérer le message dans ce fichier


          # CONFIGURATION SFTP

# Dans SSH :
Subsystem sftp /usr/lib/openssh/sftp-server
# Ou version plus sécurisée
Subsystem sftp internal-sftp
# moins de dépendances système et plu sécurisé


## Configuration des clients

### Configuration globale : /etc/ssh/ssh_config

```bash
# Fichier principal pour tous les utilisateurs :
Host *
  ForwardAgent no            # N'expose pas les clés locales sur un serveur distant
  ForwardX11 no              # interdit l'affichage graphique via SSH
  ServerAliveInterval 60     # ping toutes les 60 secondes pour garder la session vivante
  ServerAliveCountMax 3      # Coupe la session si pas de réponse après 3 pings
  StrictHostkeyChecking ask  # prévient si la clé du serveur change
  UserKnownHostsFile ~/.ssh/known_hosts  # Fichier qui garde la liste des serveurs connus
  LogLevel INFO              # niveau de log côté client
```


