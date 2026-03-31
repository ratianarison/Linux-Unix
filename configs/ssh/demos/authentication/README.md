# Scénario 1 : Accès SSH à un serveur d'entreprise

## Contexte

Une petite entreprise a un serveur Debian utilisé pour stocker des fichiers internes et administrer les systèmes.

Les administrateurs veulent permettre la connexion SSH uniquement à certains utilisateurs autorisés.

Pour renforcer la sécurité : 

1. Seuls certains comptes peuvent se connecter via SSH
2. Les comptes sans mot de passe sont automatiquement refusés
3. Certains comptes existent dans le système mais ne sont pas autorisés à utiliser SSH

## Utilisateurs du système

| Utilisateurs | Rôle | Accès SSH |
|--------------|------|-----------|
| admin | Administrateur système | autorisé |
| user1 | Employé IT | autorisé |
| user2 | Employé IT | autorisé |
| user3 | Employé IT | autorisé |
| guest | Compte invité | refusé |
| test | Compte de test interne | refusé |

**Création des utilisateurs**

```bash
sudo useradd -m -s /bin/bash admin
sudo useradd -m -s /bin/bash user1
sudo useradd -m -s /bin/bash user2
sudo useradd -m -s /bin/bash user3
sudo useradd -m -s /bin/bash guest
sudo useradd -m -s /bin/bash test
```

**Vérification**

```bash
tail /etc/passwd
```

**Configuration de mots de passe**

```bash
# Attribuer un mot de passe pour chaque utilisateur sauf user3
sudo passwd admin
sudo passwd user1
sudo passwd user2
sudo passwd guest
sudo passwd test
```

## Politique de sécurité appliquée

**1. Authentification par mot de passe obligatoire**

Dans le fichier `/etc/ssh/sshd_config`, on active : 

```bash
PasswordAuthentication yes
PermitEmptyPasswords no
PubkeyAuthentication no # pour le moment
StrictModes no # pour le moment
```

**2. Autoriser uniquement certains utilisateurs**

Toujours dans `sshd_config` : 

```bash
AllowUsers admin user1 user2 user3 z

# ces utilisateurs peuvent se connecter
# tous les autres utilisateurs sont refusés même s'ils existent
```

## Comportement du serveur 

**Cas 1 : admin se connecte**

```bash
ssh admin@server
# accès accepté si mot de passe correcte
```

**Cas 2 : Guest connecte**

```bash
ssh guest@server
# refusé, car guest n'est pas dans AllowUsers
```

**Cas 3 : user3 se connecte**

```bash
ssh user3@server
# refusé car user3 n'a pas de mot de passe
```

---

# Scénario 2 : Authentification SSH par clé publique

## Contexte 

L'entreprise héberge des scripts d'administration et des fichiers sensibles.

Pour améliorer la sécurité, l'adminitrateur décide de : 

- désactiver l'authentification par mot de passe
- utiliser l'authentification par clé publique SSH
- autoriser uniquement certains utilisateurs.

Les connexions SSH se font depuis une machine utilisée par les administrateurs.

## Génération des clés sur le client 

Sur la machine de l'administrateur :

```bash
ssh-keygen -t ed25519 -C "admin@server"
# ED25519 est plus moderne que RSA
# Résultat :
#        ~/.ssh/id_ed25519
#        ~/.ssh/id_ed25519.pub
# La clé privée ne doit jamais être partagée
```

## Copie de la clé publique vers le serveur

```bash
ssh-copy-id admin@server
# La clé sera ajoutée dans : /home/admin/.ssh/authorized_keys
```
**Structure des clés sur le serveur**

Dans le compte utilisateur : 

```text
/home/admin/
        │
        └── .ssh
             │
             └── authorized_keys
```

Exemple de contenu : 

```text ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAA... admin@server```

## Configuration SSH du serveur : fichier /etc/ssh/sshd_config

```bash
PubkeyAuthentication yes
PasswordAuthentication yes
PermitEmptyPassword no
StrictModes yes
AllowUsers admin user1 user2 user3
DenyUsers guest test
```

**Redémarrer SSH**

```bash
sudo systemctl restart ssh
sudo systemctl restart sshd
```
