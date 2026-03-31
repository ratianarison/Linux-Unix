# Scénario 4 : Accès sécurisé aux services internes via SSH Tunneling

## Contexte 

Une entreprise possède une infrastructure réseau interne protégée par un pare-feu.

Certains services critiques ne sont accessibles que depuis le réseau interne, par exemple : 

- serveur web interne
- base de données
- interface d'administration
- NIS

Les administrateurs travaillent à distance et utilisent SSH tunneling et port forwarding pour accéder à ces services de manières sécurisée.

## Architecture du réseau 

```text
Administrateur (Kali Linux)
        │
        │ SSH
        ▼
Serveur Bastion (Debian SSH)
        │
        │ réseau interne
        ▼
Serveur interne (NIS)
```

## Types de tunneling SSH utilisés 

L'entreprise utilise trois techniques :

| Type               | Nom | Utilisation                  |
| ------------------ | --- | ---------------------------- |
| Local forwarding   | -L  | accéder à un service interne |
| Remote forwarding  | -R  | exposer un service local     |
| Dynamic forwarding | -D  | proxy SOCKS sécurisé         |

## Local Port Forwarding (SSH -L)

**Objectif** : Permettre l'administrateur d'accéder à un serveur web interne innaccessible depuis internet.

Le serveur web interne : 

```text
10.0.0.20:80
```

**Commande SSH** : 

```bash
ssh -L 8080:10.0.0.20:80 admin@server
```

**Fonctionnement** : 

```text
Navigateur local
     │
localhost:8080
     │
     ▼
Tunnel SSH
     │
     ▼
Serveur Bastion
     │
     ▼
10.0.0.20:80 (serveur web interne)
```

**Accès au site** :

Dans le navigateur : 

```text
http://localhost:8080
```

Le trafic passe dans le tunnel SSH chiffré.

## Remote Port Forwarding (SSH -R)

**Objectif** : Permettre au serveur distant d'accéder à un service local.

Situation : 
  Un développeur teste une application web sur sa machine : 
  ```text
  localhost:5000
  ```
  Il veut que le serveur puisse y accéder.

**Commande**

```bash
ssh -R 9000:localhost:5000 admin@server
```

**Fonctionnement**

```text
Serveur Bastion
     │
localhost:9000
     │
     ▼
Tunnel SSH
     │
     ▼
Machine Kali
localhost:5000
```

Le serveur peut maintenant accéder à localhost:5000.

## Dynamic Port Forwarding (SSH -D)

**Objectif** : Créer un proxy SOCKS sécurisé.

Utilisation :
- Navigation sécurisée
- Contourner certaines restrictions réseau
- analyser le trafic via tunnel SSH.

