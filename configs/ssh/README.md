
# 🔐 SSH (Secure Shell)

## 1. 📍 Définition et Rôle

SSH est un protocole de communication sécurisé qui remplace les anciens protocoles non sécurisés comme telnet, rlogin, rsh ou FTP.
Il permet d'administrer des machines à distances, de transférer des fichiers et d'éxecuter des commandes de façon chiffrée et authentifiée.

**Pourquoi SSH est indispensable ?**

Problème des anciens protocoles | Solution SSH
--------------------------------|-------------
Mots de passe en clair sur le réseau | Tout est chiffré
Pas d'authentification du serveur | Clés hôtes pour vérifier le serveur
Pas d'authentification forte du client | Clés publiques/privés
Données transférées en clair | Chiffrement de bout en bout

## 2. 🏗️ Architecture SSH

**Modèle Client-serveur**

```text
┌─────────────────┐                    ┌─────────────────┐
│   CLIENT        │                    │   SERVEUR       │
│                 │                    │                 │
│  ssh client     │◄───── SSH ────────►│  sshd (daemon)  │
│  (local)        │    Port 22         │  (distant)      │
└─────────────────┘                    └─────────────────┘
```

**Composant**

Composant | Rôle | Fichier/Commande
----------|------|-----------------
Client SSSH | Se connecte aux serveurs | <mark>ssh</mark> commande
Serveur SSH (sshd) | Accepte les connexions | <mark>/etc/ssh/sshd_config</mark>
Agent SSH | Gère les clés privées | <mark>ssh-agent</mark>
SCP | Copie de fichiers sécurisée | <mark>scp</mark>
SFTP | Transfert de fichiers interactif | <mark>sftp</mark>

## 3. 📌 Fonctionnement technique 

### 3 Phases d'une connexion SSH

Une connexion SSH se déroule en trois phases distinctes qui s'enchaînent pour établir un canal de communication sécurisé. 

**🔑 Phase 1 : Négociation et établissement du chiffrement**

<b>Objectif</b> : Établir un canal chiffré entre le client et le serveur, sans qu'aucune information sensible (password, commands) ne circule en clair.

*Étape 1 : Ouverture de la connexion TCP*

```bash
Client(port aléatoire 54321) ──SYN──► Serveur (port 22)
Client ◄──SYN-ACK── Serveur
Client ──ACK──► Serveur
# Connection établie
```
*Étape 2 : Négociation des algorithmes**

```bash
Client  ──────────────────────────────────────────► Serveur
"Je supporte :"
- Chiffrement symétrique : AES256, ChaCha20, 3DES
- Échange de clés : Diffie-Hellmn, ECDH, Curve25519
- Authentification : RSA, FCDSA, Ed25519
- Compression : zlib, nonz

Client ◄────────────────────────────────────────── Serveur
"Choisissons :
- Chiffrement : AES256
- Échange de clés : Curve25519
- Authentification : Ed25519
- Compression : none"
```

*Étape 3 : Échange de clés (Key Exchange)*

C'est l'étape le plus critique. Le client et le serveur vont créer une clé de session secrète sans jamais l'envoyer sur le réseau.

```bash
# Algorithme Diffie-Hellman (simplifié)
Client                                    Serveur
   │                                         │
   │  Génère un nombre privé a                │  Génère un nombre privé b
   │  Calcule g^a mod p                       │  Calcule g^b mod p
   │                                         │
   │  ──────────── g^a mod p ──────────────► │
   │                                         │
   │  ◄─────────── g^b mod p ─────────────── │
   │                                         │
   │  Calcule (g^b)^a mod p = g^ab mod p     │  Calcule (g^a)^b mod p = g^ab mod p
   │                                         │
   └───────────────────┴─────────────────────┘
              CLÉ DE SESSION IDENTIQUE !
```

La clé de session <mark>g^a mod p</mark> est connue uniquement du client et du serveur, jamais transmise.

*Étape 4 : Vérification de la clé hôte (Host Key)*

```bash
Client ◄────────────────────────────────────────── Serveur
"Voici ma clé publique hôte (fingerprint)"

Client vérifie :
~/.ssh/known_hosts
├── Si la clé est déjà connue : comparaison
├── Si clé nouvelle : demande de confirmation
├── Si clé différente : ALERTE (possible attaque MITM)

# Exemple de message
The autheticity of host 'master (192.168.1.10)' can't be established
ED2559 key fingerprint is SHA256:xxxxxxxxxxxxxxxxxxxx.
Are you sure you want to continue connecting (yes/no/[fingerprint])?
```

*Étape 5 : Canal chiffré établi**

```text

TEMPS 0 : Connexion TCP
┌─────────────────────────────────────────────────────────────────┐
│ Client                 TCP 3-way handshake              Serveur │
└─────────────────────────────────────────────────────────────────┘

TEMPS 1 : Négociation
┌─────────────────────────────────────────────────────────────────┐
│ "AES256,ChaCha20,DH,Curve25519,RSA,ECDSA,Ed25519"               │
│ ◄─────────────────────────────────────────────────────────────  │
│ "AES256,Curve25519,Ed25519"                                     │
└─────────────────────────────────────────────────────────────────┘

TEMPS 2 : Échange de clés (Clé de session créée
┌─────────────────────────────────────────────────────────────────┐
│ g^a mod p ───────────────────────────────────────────────────►  │
│ ◄─────────────────────────────────────────────────── g^b mod p  │
│                 La clé de session est établie                   │
└─────────────────────────────────────────────────────────────────┘

TEMPS 3 : Vérification hôte
┌─────────────────────────────────────────────────────────────────┐
│ ◄─────────────────────────────────────────────────── Clé hôte   │
│ ✓ Vérification fingerprint                                      │
└─────────────────────────────────────────────────────────────────┘

TEMPS 4 : CANAL CHIFFRÉ OPÉRATIONNEL
```

**🆔 PHASE 2 : Authentification du client**

*Objet :* Prouver que le client est bien celui qui prétend être. Plusieurs méthodes possibles.

*Méthode 1 : Authentification par mot de passe*

```bash
Client ◄────────────────────────────────────────── Serveur
"Veuillez vous identifier"

Client (chiffré) ─────────────────────────────────► Serveur
"Username: x"
"password: *******"

Serveur vérifie :
/etc/passwd + /etc/shadow

Client ◄────────────────────────────────────────── Serveur
"Authentification réussie" ou "Échec"
```

*Méthode 2 : Authentification par clé publique*

```bash
Étape 2.1 : Client annonce sa méthode
Client ──────────────────────────────────────────► Serveur
"Je souhaite m'authentifier avec ma clé publique"

Étape 2.2 : Serveur envoie un défi (challenge)
Client ◄────────────────────────────────────────── Serveur
"Voici un message chiffré avec votre clé publique"
"Message: 'abc123def456'

Étape 2.3 : Client déchiffre et renvoie
Client ──────────────────────────────────────────► Serveur
"Voici le message déchiffré: 'abc123def456'"
"Signature de ce message avec ma clé privée"

Étape 5 : Serveur vérifie
Serveur:
1. Vérifie le message déchiffré(correct ?)
2. Vérifie la signature (valide ?)
3. Vérifie que la clé publique est dans ~/.ssh/authorized_keys

Client ◄────────────────────────────────────────── Serveur
"Authentification réussie"
```

*Méthode 3 : Authentification par Challenge - Response (Kerberos, etc.)*

```bash
# Utilisé dans les environnements avec Kerberos (Active Directory)
Client ◄────────────────────────────────────────── Serveur
"Défi Kerberos"
Client ──► Serveur "Réponse Kerberos"
# Vérification par le KDC (Key Distribution Center)
```

*Méthode 4 : Authentification par clé hôte*

```bash
# Utilisée pour les scripts automatisés
# Le client s'authentifie avec la clé hôte du serveur
# Réseré à des cas spécifiques
```

*Ordre des méthodes d'authentification*

```bash
# Dans /etc/ssh/sshd_config
# L'ordre peut être configuré
AuthenticationMethoods pubkey,password
# Force l'authentification par clés puis mot de passe (double facteur)
```

## 💬 Phase 3 : Session chiffré

**Objectif :** Échanger des commandes, des données et des fichiers de manière sécurisée et interactive.

**Types de sessions**

*1. Session Shell interactive*

```bash
Client                           Serveur
   │                                │
   │  ─── "ls -la" (chiffré) ────►  │
   │                                │ Exécute ls
   │  ◄─── "fichier1 fichier2" ───  │
   │                                │
   │  ─── "cd /etc" (chiffré) ───►  │
   │                                │
   │  ─── "cat passwd" (chiffré) ─► │
   │  ◄─── Contenu du fichier ────  │
```

*2. Session d'exécution de commande unique*

```bash
# Côté client
ssh user@server "ls -la /var/yp"

# Ce qui se passe :
  # 1. Phase 1 et 2 complètes
  # 2. Commande "ls -la /var/yp" est envoyée chiffrée
  # 3. Serveur exécute et renvoie la sortie
  # 4. Connexion se ferme immédiatement
```

*3. Session de transfert de fichiers (SFTP)*

```bash
Client                            Serveur
   │                                 │
   │  ─── "open" (chiffré) ───────►  │
   │  ◄─── "SFTP server ready" ───   │
   │                                 │
   │  ─── "ls" (chiffré) ─────────►  │
   │  ◄─── "fichier1 fichier2" ───   │
   │                                 │
   │  ─── "get fichier1" (chiffré) ► │
   │  ◄─── [DONNÉES BINAIRES] ────   │
```

*4. Session de tunnel (port forwarding)*

```bash
# Tunnel local
ssh -L 8080:localhost:80 user@serveur

# Client (local)    SSH Client    SSH Serveur    Serveur Web
#     │               │              │              │
#     │──:8080───────►│              │              │
#     │               │──chiffré────►│              │
#     │               │              │──:80────────►│
#     │               │              │◄──Réponse────│
#     │               │◄──chiffré────│              │
#     │◄──Réponse─────│              │              │
```

*Caractéristique de la session chiffrée*

Chiffrement symétrique :

```text
La même clé sert à chiffrer et déchiffrer

Donnée + Clé Session → Donnée chiffrée
Donnée chiffrée + Clé Session → Donnée originale
```

Intégrité des données (MAC)

```text
Message Authentication Code
Garantit que les données n'ont pas été modifiées
[message] + [clé session] →  HMAC
Si HMAC ne correspond pas → session coupée
```

Compression (optionnelle)

```text
Pour améliorer les performances sur réseaux lents
Donnée originale → Compression → Cheffrement → Envoi
```

Canaux multiples dans une session

```text
Une seule connexion SSH peut transporter plusieurs canaux :

┌─────────────────────────────────────────────────┐
│               CONNEXION SSH                     │
├─────────────────────────────────────────────────┤
│  Canal 1 : Shell interactif (stdin/stdout)      │
│  Canal 2 : Transfert SFTP (fichiers)            │
│  Canal 3 : Tunnel local 8080 → serveur:80       │
│  Canal 4 : Tunnel distant (reverse)             │
│  Canal 5 : Agent forwarding                     │
└─────────────────────────────────────────────────┘
```
<details>
<summary>Exemple</summary>
  
## 🔍 Connexion SSH en mode verbose

```bash
$ ssh -vvv user@master.tp.local

# PHASE 1
debug1: Connecting to master.tp.local [10.237.80.115] port 22.
debug1: Connection established.
debug1: identity file /home/user/.ssh/id_rsa type 0
debug1: Local version string SSH-2.0-OpenSSH_9.2
debug1: Remote protocol version 2.0, remote software version OpenSSH_9.2
debug1: compat_banner: match: OpenSSH_9.2 pat OpenSSH*
debug2: fd 3 setting O_NONBLOCK
debug1: Authenticating to master.tp.local:22 as 'user'
debug1: SSH2_MSG_KEXINIT sent
debug1: SSH2_MSG_KEXINIT received
debug2: kex_parse_kexinit: curve25519-sha256,ecdh-sha2-nistp256...
debug2: kex_parse_kexinit: aes256-gcm@openssh.com,chacha20-poly1305@openssh.com...
debug1: kex: algorithm: curve25519-sha256
debug1: kex: host key algorithm: ssh-ed25519
debug1: kex: server->client cipher: chacha20-poly1305@openssh.com MAC: <implicit> compression: none
debug1: kex: client->server cipher: chacha20-poly1305@openssh.com MAC: <implicit> compression: none
debug1: expecting SSH2_MSG_KEX_ECDH_REPLY
debug1: SSH2_MSG_KEX_ECDH_REPLY received
debug1: Server host key: ssh-ed25519 SHA256:xxxxxxxxxxxxxxxxxxxx
debug1: Host 'master.tp.local' is known and matches the ED25519 host key.
debug1: rekey out after 134217728 blocks
debug1: SSH2_MSG_NEWKEYS sent
debug1: expecting SSH2_MSG_NEWKEYS
debug1: SSH2_MSG_NEWKEYS received
debug1: rekey in after 134217728 blocks

# PHASE 2
debug1: Will attempt key: /home/user/.ssh/id_rsa RSA SHA256:yyyyyyyyyy
debug1: SSH2_MSG_EXT_INFO received
debug1: kex_input_ext_info: server-sig-algs=<ssh-ed25519,sk-ssh-ed25519@openssh.com,ssh-rsa,rsa-sha2-256,rsa-sha2-512,ssh-dss,ecdsa-sha2-nistp256,ecdsa-sha2-nistp384,ecdsa-sha2-nistp521>
debug1: SSH2_MSG_SERVICE_ACCEPT received
debug1: Authentications that can continue: publickey,password
debug1: Next authentication method: publickey
debug1: Offering public key: /home/user/.ssh/id_rsa RSA SHA256:yyyyyyyyyy
debug1: Server accepts key: /home/user/.ssh/id_rsa RSA SHA256:yyyyyyyyyy
debug1: Authentication succeeded (publickey).

# PHASE 3
debug1: channel 0: new [client-session]
debug1: Entering interactive session.
debug1: pledge: filesystem
debug1: client_input_global_request: rtype hostkeys-00@openssh.com want_reply 0
debug1: Sending environment.
debug1: Sending env LANG = en_US.UTF-8
debug1: Sending command: /bin/bash
Last login: Sat Mar 21 10:00:00 2026 from 10.237.80.247
user@master:~$
```
</details>
