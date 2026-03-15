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

Le fichier */etc/ypserv.conf* est le fichier de configuration principal du serveur NIS. Il contrôle le comportement du serveur NIS et définit les règles de sécurité pour l'accès aux maps NIS.

Avant de modifier ce fichier, il est préférable de voir le manuel ypserv.conf(5)

  ```bash
  man ypserv.conf
  ```
<details>
  <summary><i>Fichier /etc/ypserv.conf</i></summary>
  
  ```markdown
  YPSERV.CONF(5)                NIS Reference Manual               YPSERV.CONF(5)

  NAME
    ypserv.conf - configuration file for ypserv and rpc.ypxfrd

  DESCRIPTION
    ypserv.conf is an ASCII file which contains some options for ypserv. It also contains a list
    of rules for special host and map access for ypserv and rpc.ypxfrd. This file will be read by
    ypserv and rpc.ypxfrd at startup, or when receiving a SIGHUP signal.

    There is one entry per line. If the line is a option line, the format is :

          option: argument

    The line for an access rule has the format :

          host:domain:map:security

    All rules are tried one by one. If no match is found, access to a map is allowed.

    Following options exist :

          files: 30
    This option specifies, how many database files should be cached by ypserv. If 0 specified, caching is disabled.
    Decreasing this number is only possible, if ypserv is restarted.

          trusted_master: server

    If this options is set on a slave server, new maps from the host server will be accepted as master. The default
    is, that no trusted master is set and new maps will not be accepted.

    Example :

          trusted_master: master.tp.local

          slp: [yes|<no>|domain]
    If this option is enabled and SLP supported compiled in, the NIS server registers on a SLP server. If the variable
    is set to domain, an attribute domain with a comma separated list of supported domainnames is set. Else this
    attribute will not be set. The default is "no" (disabled). (SLP : Service Location Protocol)

          xfr_check_port: [<yes>|no]
    With this option enabled, the NIS master server have to run on a port < 1024. The default is "yes" (enabled)

    The field descriptions for the access rule lines are :

          host
    IPv4 only address. Wildcards are allowed. This rules are ignored for IPv6, which means it is better to not use
    this option at all anymore

          Examples :
                131.234. = 131.234.0.0/255.255.0.0
                131.234.214.0/255.255.254.0

          domain
    specifies the domain, for which this rule should be applied. AN asterix as wildcard is allowed.

          map
    name of the map, or asterisk for all maps.

          security
    one of none, port, deny :

    none : always allows access
    port : allow access if from port 1024. Otherwise do not allow access.
    deny : deny access to this map

  ```
</details>

**Configuration du serveur master**

  ```bash
  sudo nano /etc/ypserv.conf
  ```
  ```bash
  # Configuration commune entre master et slave :

  # Options générales :
  dns: yes    # Le serveur interrogera le DNS pour touver ses clients qui n'apparaissent pas dans les maps hosts.
  xfr_check_port: yes  # Vérification des ports < 1024 pour les transferts.

  # Règles d'accès :
  # host:domain:map:security
  192.168.1.0/24:*:*:none      #autoriser tout le réseau local

  # Protéger les mots de passe (shadow) via username et uid
  *:*:shadow.byname:port
  *:*:shadow.byuid:port

  # Refuser tout le reste
  *:*:*:deny

  # Configuration spécifique pour le serveur slave :

  trusted_master: master.tp.local
  ```

**Vérification et rechargement**

  ```bash
  # Redémarrer le service
  sudo systemctl restart ypserv

  # Vérifier les logs
  sudo journalctl -u ypserv -f
  ```
### Fichier <mark>*/var/yp/Makefile*</mark>

Le fichier */var/yp/Makefile* est le cœur de la génération des maps NIS. C'est un makefile qui contrôle comment les bases de données sources sont converties en maps NIS (fichiers.db) utilisables par les clients.

<details>
  <summary>Explication</summary>
  
  ### 1. Rôle principal
  
  Ce makefile :
   - Lit les fichiers source (/etc/passwd, /etc/group, /etc/hosts, etc.)
   - Génère les maps NIS (fichier.db dans <mark>/var/yp/[domaine]/</mark>
   - Met à jour le serveur quand les sources changent
   - Propagé aux slaves (optionnel)

  ### 2. Structure du fichier 

  **Variables de configuration (en haut du fichier)**

  ```makefile
  # Répertoire des maps
  YPDIR=/var/yp
  # Domaine NIS
  DOMAIN=`domainname`
  # Serveur slaves (pour propagation)
  NOPUSH=true
  # Ou pour activer la propagation
  #NOPUSH=false
  ```
  **Fichiers sources à inclure**
  ```makefile
  #Quels fichiers transformer en maps
  all: passwd group hosts rpc services netid protocols netgrp \
       mail shadow publickey networks ethers bootparams printcap \
       amd.home auto.master auto.home auto.local passwd.adjunct \
       netmasks timezone locale netgroup
  ```

  **Cibles de génération**
  
  ```makefile
  # Cible par défaut
  all: passwd group hosts ...

  # Comment générer chaque map
  passwd:passwd.time
  passwd.time: $(YPSRCDIR)/passwd
    @echo "Updating passwd..."
    @$(MAKE)-C $(YPDIR) passwd
  ```

### 3. Exemple concret de sections importantes 

**Définition des fichiers sources**

```makefile
# Où trouver des fichiers sources
YPSRCDIR=/etc

# Fichiers à inclure
YPPWDDIR=/etc

# Maps à générer à partir de /etc/passwd
PASSWD=$(YPPWDDIR)/passwd
SHADOW=$(YPPWDDIR)/shadow
GROUP=$(YPPWDDIR)/group
HOSTS=$(YPPWDDIR)/hosts
NETWORKS=$(YPPWDDIR)/networks
PROTOCOLS=$(YPPWDDIR)/networks
SERVICES=$(YPPWDDIR)/services
```

**Règles de génération pour passwd

```makefile
passwd.time: $(PASSWD) $(SHADOW)
  @echo "Bulting passwd maps..."
  @$(AWK) -f $(YPDIR)/mkpasswd.awk $(PASSWD) > $(YPDIR)/passwd.tmp
  @$(DBLOAD) -i $(YPDIR)/passwd.tmp -o $(YPDIR)/$(DOMAIN)/passwd.byname \
    -e $(YPDIR)/$(DOMAIN)/passwd.byuid
  @rm -f $(YPDIR)/passwd.tmp
  @touch passwd.time
  @echo "passwd maps built successfully."
```

### 4. Commandes courantes avec le Makefile

**Reconstruire tous les maps**
```bash
cd /var/yp
sudo make
```

**Reconstruire une map spécifique**
```bash
cd /var/yp
sudo make passwd
sudo make hosts
sudo make group
```

**Forcer la reconstruction même su rien a changé**
```bash
cd /var/yp
sudo make -B
#ou
sudo make clean all
```
**Voir ce qui va être fait**
```bash
cd /var/yp
sudo make -n
```

### 5. Personnalisations courantes

**Ajouter des maps personnalisées**

```makefile
# Ajouter des propres maps
all: passwd group hosts ... mes_maps
mes_maps: mes_maps.time
mes_maps.time:/etc/mes_fichiers
  @echo "Bulting custom maps ..."
  @cp /etc/mes_fichiers $(YPDIR)/mes_maps.txt
  @$(DBLOAD) -i $(YPDIR)/mes_maps.txt -o $(YPDIR)/$(DOMAIN)/mes_maps.byname
  @touch mes_maps.time
```

**Exclure certains utilisateurs**

```makefile
# Dans mkpasswd.awk ou en modifiant la règle passwd
# Exclure les utilisateurs (UIS < 1000)
awk -F: '$3 >= 1000 {print}' /etc/passwd > /tmp/passwd.nis
```

**Activer la propagation vers les slaves**

```makefile
# Changer en false pour activer la push
NOPUSH=false
# Spécifier les slaves
SLAVES="192.168.1.11 192.168.1.12"
```
### 6. Exemple de fichier typique simplifié

```makefile

# /var/yp/Makefile - Version simplifiée
#
DIR=/var/yp
DOM=`domainname`
NOPUSH= false

all: passwd group hosts services protocols netgroup

passwd: passwd.time
passwd.time: /etc/passwd /etc/shadow
  @echo "Making passwd maps..."
  @umask 077;\
  awk 'BEGIN {FS="."}{print $$1,$$0}' /etc/passwd | \
  $(DBLOAD) -i --o $(DIR)/$(DOM)/passwd.byname;\
  awk 'BEGIN {FS=":"}{print $$3,$$0}' /etc/passwd | \
  $(DBLOAD) -i --o $(DIR)/$(DOM)/passwd.byuid
  @touch passwd.time

group: group.time
  @echo "Making group maps..."
  @umask 077;\
  awk 'BEGIN {FS=":"}{print $$1,$$0}' /etc/group | \
  $(DBLOAD) -i --o $(DIR)/$(DOM)/group.byname;\
  awk 'BEGIN {FS=":"}{print $$3,$$0}' /etc/group | \
  $(DBLOAD) -i --o $(DIR)/$(DOM)/group.byuid
  @touch group.time

# Règles similiaires pour autres maps
clean:
  rm -f *.time
```

### 7. Points importants à retenir

1. Après chaque modification d'un fichier source (/etc/passwd, etc.), il faut exécuter <mark>make</mark> dans <mark>/var/yp</mark>.
2. L'ordre des maps dans la cible <mark>all</mark> détermine l'ordre de génération
3. Les fichiers.time sont des marqueurs qui indiquent quand chaque map a été générée pour la dernière fois.
4. Les maps sont stockées dans <mark>*/var/yp/[domainname]/*</mark> avec des extensions comme .byname, .byuid, etc.
5. La variable NOPUSH contrôle si les maps sont automatiquement poussées aux slaves

### 8. Vérification après génération

```makefile
# Voir les maps générées
ls -la /var/yp/$(domainname)/

# Voir le contenu d'une map
ypcat passwd
ypcat hosts
```
</details>

```bash
sudo nano /var/yp/Makefile
```
```makefile
# autorisé le PUSH vers les slaves
NOPUSH = false

# Domaines à servir  (par défaut toutes les maps)
all: passwd group hosts networks protocols services rpc ethers netgroup bootparams aliases

# Répertoire des fichiers sources
YPSRCDIR = /etc
YPWDDIR = /etc

# Maps à générer à partir des fichiers
PASSWD: $(YPSRCDIR)/passwd
GROUP: $(YPSRCDIR)/passwd
#...
```

## 1.8. Initialisation et lancement du serveur NIS

### Sur le serveur maître (Debian-Master)

#### Étape 1 : Configuration du nom de domaine NIS

  ```bash
  # Définir le nom de domaine NIS pour la session courante
  sudo nisdomainname TP-NIS

  # Rendre permanent
  echo "TP-NIS" | sudo tee /etc/defaultdomain

  # Vérification
  nisdomainname
  domainname # Affiche le domaine NIS
  ```

#### Étape 2 : Vérification/Mise à jour <mark>*/etc/hosts*</mark>

  ```bash
  cat /etc/hosts
  # Doit contenir master, slave, client
  ```

#### Étape 3 : Vérification des fichiers de configuration

  ```bash
  # Vérifier host.conf (ordre de résolution des noms : order hosts,nis,bind \ multi on
  cat /etc/host.conf

  # Vérifier nsswitch.conf
  grep -E "passwd|group|shadow|hosts" /etc/nsswitch.conf

  # Vérifier ypserv.conf
  cat /etc/ypserv.conf
  ```

#### Étape 4 : Vérification des scripts de démarrage

  ```bash
  # Vérifier que les services existent
  ls -la /etc/init.d/ | grep -E "ypserv|yppasswdd|ypxfrd"

  # ou avec systemd (Debian moderne)
  ls -la /lib/systemd/system/ | grep -E "ypserv|yppasswdd|ypxfrd"
  ```

#### Étape 5 : Lancer les services 

  ```bash
  # Démarrer les services
  sudo systemctl start ypserv
  sudo systemctl start yppaswdd
  sudo systemctl start ypxfrd

  # Activer au démarrage
  sudo systemctl enable ypserv
  sudo systemctl enable yppasswdd
  sudo systemctl enable ypxfrd

  # Vérifier le statut
  sudo systemctl status ypserv
  sudo systemctl status yppasswdd
  sudo systemctl status ypxfrd

  # Vérifier que les services RPC sont enregistrés
  rpcinfo -p | grep yp
  ```

#### Étape 6 : Initialiser le serveur maître

  ```bash
  # Initialiser les bases NIS
  sudo /usr/lib/yp/ypinit -m
  ```

### Sur le serveur esclave

#### Étape 1 : Installer comme client NIS d'abord

  ```bash
  sudo apt install nis rpcbind
  
  # Configuration du domain
  sudo nisdomainname TP-NIS
  echo "TP-NIS" | sudo tee /etc/defaultdomain

  # Configurer /etc/hosts
  sudo nano /etc/hosts
  # Ajouter les mêmes entrées

  # Tester la connexion au maître
  ping master
  # ou
  ping master.tp.local
  ```

#### Étape 2 : Configurer comme serveur (étapes identiques au maître)

  ```bash
  # Configurer nsswitch.conf, host.conf, ypserv.conf (comme le maître)

  # Démarrer les services
  sudo systemctl start ypserv
  sudo systemctl enable ypserv
  ```

#### Initialisation du serveur esclave

  ```bash
  # Initialiser en tant qu'esclave du maître 
  sudo /usr/lib/yp/ypinit -s master.tp.local
  ```

#### Étape 4 : Configuration finale sur le maître

  ```bash
  # Sur Debian-Master, éditer /var/yp/Makefile
  sudo nano /var/yp/Makefile
  # Modifier: NOPUSH=false

  # Configurer ypbind pour le serveur maître
  sudo nano /etc/yp.conf
  # Ajouter : domain TP-NIS server localhost
  # Ajouter : domain TP-NIS 127.0.0.1
  # Ajouter : domain TP-NIS IP_DU_SERVEUR

  # Mettre à jour /var/yp/ypservers
  sudo nano /var/yp/ypservers
  # Ajouter master.tp.local
  # Ajouter: slave.tp.local

  # Forcer la mise à jour des maps
  cd /var/yp
  sudo make
  ```

## 1.9. Configuration client

### Sur Debian-Client

