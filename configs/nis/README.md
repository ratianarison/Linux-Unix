# Partie I : MISE EN PLACE DE NIS (Network Information Service)

## 1.1. Présentations de NIS

NIS (Network Information Service), anciennement appelé *"Yellow Pages" (YP)*, est un protocole qui constitue une base de données répartie pour les fichiers de configuration UNIX. Il permet de centraliser les fichiers de configuration sur un serveur NIS pour éviter de dupliquer des fichiers sur chaque machine d'un réseau.

**Objectif** : Un utilisateur peut se connecter sur n'importe quelle machine du réseau avec le même login, et les informations système (passwd, group, hosts, etc.) sont partagées.

## 1.2. Services/Démons NIS

**Portmap (ou rpcbind)**

**. Rôle** : Mise en correspondance des numéros de ports TCP/IP avec les numéros de porcessus RPC.

**. Fichier de référence** : */etc/rpc* contient les numéros RPC réservés

**. Paquet Debian**: *rpcbind*

**Côté serveur**

| Démon | Rôle | Paquet |
|-------|------|--------|
| ypserv | Implémente le serveur NIS principal | nis |
| rpc.yppasswdd | Permet de changer un mot de passe sur le serveur NIS depuis un client | nis |
| rpc.ypxfrd | accélère le transferts entre serveur maître et esclave | nis |

**Côté client**

| Démon | Rôle | Paquet |
|-------|------|--------|
| ypbind | Implémente le client NIS | nis |
| yp-tools | Outils de diagnostic (ypwhich, ypcat, ypmatch) |

## Principe de fonctionnement

                    [Serveur NIS Maître]
                    192.168.1.10 (Debian-Master)
                           │
                ┌──────────┴──────────┐
                │                      │
        [Serveur NIS Esclave]    [Clients NIS]
        192.168.1.11             192.168.1.12
        (Debian-Slave)            (Debian-Client)
                │                      │
                └──────────┬───────────┘
                    [Autres machines]
                    192.168.1.x (Clients)

**Mécanisme**

1. Le serveur maître maintient les bases de données maîtresses
2. Les esclaves se synchronisent périodiquement avec le maître
3. Les clients interrogent le maître ou les esclaves pour obtenir les informations
4. En cas de panne du maître, les esclaves continuent de servir les clients

## 1.4. Installation des paquets nis, rpcbind, yptools

**Pour le serveur (Debian-Master et Debian-Client)**
  ```bash
    # Mise à jour des dépôts
    sudo apt update

    # Installation du serveur NIS
    sudo apt install nis rpcbind

    # Vérification des paquets installés
    dpkg -l | grep nis
  ```
**Pour le client**

  ```bash
  sudo apt update
  sudo apt install nis rpcbind yp-tools
  dpkg -L yp-tools
  ypbind --version
  ```
## 1.5. Configuration commune (Client & Serveur)

**Fichier <mark>*/etc/nsswitch.conf*</mark>**

Le fichier <mark>*/etc/nsswitch*</mark> (Name Service Switch) configure l'ordre et les sources utilisés pour résoudre différentes informations système (utlisateurs, groupes, hôtes, etc.).

  ```bash
  sudo nano /etc/nsswitch.conf
  ```
  ```bash
  passwd:    files nis     # On cherche les informations dans /etc/passwd, puis dans le maps NIS "passwd"
  group:     files nis     # d'abord dans /etc/group, puis maps NIS "group"
  shadow:    files nis     # d'abord dans /etc/shadow, puis maps NIS "shadow"

  hosts:     files dns nis # d'abord dans /etc/hosts, ensuite interrogation DNS, enfin via NIS (maps hosts)
  networks:  files nis     # d'abord /etc/networks, puis maps NIS "networks". Résultat : noms de réseau (ex: "loopback" pour 127.0.0.0)

  protocols: files nis     # d'abord /etc/protocols, puis maps NIS "protocols" => Résultat : noms de protocoles (tcp, udp, icmp, ...)
  services:  files nis     # d'abord /etc/services, puis maps NIS "services" => Résultat : noms de services (ex : "ssh" pour 22/tcp)
  ethers:    files nis     # /etc/ethers, puis maps NIS "ethers" => Résultat : Correspondance MAC <=> Hôtes (pour reverse ARP)
  rpc:       files nis     # /etc/rpc, puis maps NIS "rpc" => Résultat : noms des programmes RPC (portmapper, nfs, ...)
  netmasks:  files nis     # /etc/netmasks, puis maps NIS "netmasks" => Résultat : masques de sous-réseau par réseau

  bootparams: files nis    # /etc/bootparams, puis maps NIS "bootparams" => Résultat : Paramètres de boot pour stations sans disques
  automount:  files nis    # /etc/auto.master, puis maps NIS "auto.*" => Résultat : point de montage automatique (autofs)
  aliases:    files nis    # /etc/aliases, puis maps NIS "aliases" => Résultat : aliases mail (forwarding)
```

**Fichier <mark>*/etc/host.conf*</mark>**

Définit l'ordre de résolution des noms d'hôte (FQDN <-> adress IP).

  ```bash
  sudo nano /etc/host.conf
  ```
  ```bash
  # Ordre de résolution :
  order hosts,nis,bind
  multi on
  # hosts: consulte d'abord /etc/hosts
  # nis : interroge ensuite le serveur NIS
  # bind : utilise DNS en dernier
  # multi on : autorise plusieurs adresses IP pour un même nom
  ```

## 1.6. Configuration serveur : réseau et domain NIS

### Configuration du domaine NIS

**Fichier <mark>*/etc/defaultdomain*</mark>**

Le fichier */etc/defaultdomain* contient le nom du domaine NIS. Ce nom est un identifiant simple qui définit le groupe de machines partageant les mêmes informations. Il est important de ne pas le confondre avec le domaine DNS, ce sont deux concepts totalement distincts, même s'ils peuvent être identiques par convention.

  ```bash
  # Créer le fichier avec le nom de domaine NIS
  echo "TP-NIS" | sudo tee /etc/defaultdomain

  # Vérification
  cat /etc/defaultdomain
  ```
### Configuration réseau statique

**Fichier <mark>*/etc/network/interfaces*</mark>**

  ```bash
  sudo nano /etc/network/interfaces
  ```

**Pour Debian-Master**
  ```bash
  auto lo
  iface lo inet loopback

  auto enp0s3  # ou eth0, vérifiez avec 'ip a'
  iface enp0s3 inet static
      address 192.168.1.10
      netmask 255.255.255.0
      gateway 192.168.1.1
      dns-nameservers 8.8.8.8 8.8.4.4
  ```

**Pour Debian-Slave**

  ```bash
  auto lo
  iface lo inet loopback

  auto enp0s3  # ou eth0, vérifier avec 'ip a'
  iface enp0s3 inet static
      address 192.168.1.11
      netmask 255.255.255.0
      gateway 192.168.1.1
      dns-nameservers 8.8.8.8 8.8.4.4
  ```

### Déclaration des hôtes

**Fichier <mark>*/etc/hosts*</mark>**

  ```bash
  sudo nano /etc/hosts
  ```
Ajouté sur toutes les machines :

  ```bash
  192.168.1.10  master.tp.local master
  192.168.1.11  slave.tp.local slave
  192.168.1.12  client.tp.local client
  192.168.1.13  client.tp.local client
  ```

## 1.7. Configuration spécifique au serveur 

### Fichier <mark>*/etc/ypserv.conf*</mark>

Le fichier <mark>/etc/ypserv.conf</mark> est le fichier de configuration principal du serveur NIS (ypserv). Il contrôle le comportement du serveur NIS et définit les règles de sécurité pour l'accès aux maps NIS.

  ```bash
  sudo nano /etc/ypserv.conf
  ```

  
