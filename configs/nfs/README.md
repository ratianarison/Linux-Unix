# PARTIE 2 : MISE EN PLACE DE NFS (Network File System)

## 2.1. Présentation de NFS

NFS (Network File System) est un système de fichiers réseau qui fournit un accès réseau transparent aux fichiers d'un serveur. Développé par Sun Microsystems en 1984, c'est un protocole ouvert et portable sur de nombreux environnements.

**Objectif** : Permettre à des clients à acceder à des fichiers situés sur un serveur distant comme s'ils étaient locaux.

## 2.2. Services/Démons NFS

### Portmap (rpcbind)

- **Rôle** : Mise en correspondance des numéros de ports TCP/IP avec les numéros de processus RPC
- **Fichier de référence** : <mark>/etc/rpc</mark> pour les numéros réservés

**Côté serveur**

| Démon | Rôle | Paquet |
|-------|------|--------|
| rpc.nfsd | Implémente la partie utilisateur du protocole NFS | nfs-kernel-server |
| rpc.mountd | Gère les demandes de montage des clients | nfs-kernel-server |
| rpc.statd | Gère le verrouillage des fichiers (NSM - Network Status Monitor) | nfs-common |
| rpc.lockd | Gère le verrouillage des fichiers (NLM - Network Lock Manager) | nfs-common |

**Côté client**

| Démon | Rôle | Paquet |
|-------|------|--------|
| rpc.statd | Gestion des verrous côté client | nfs-common |
| rpc.lockd | Gestion des verrous côté client | nfs-common |

## 2.3. Principe de fonctionnement 

                          [Serveur NFS]
                         192.168.1.10
                        (Debian-Master)
                               │
                       Répertoires exportés
                    ┌──────────┼──────────┐
                 /home     /opt/share    /mnt/disk
                    │          │            │
           ┌────────┴──┐   ┌───┴────┐  ┌────┴────┐
           │           │   │        │  │         │
      [Client 1]  [Client 2]  [Client 3]  [Collègues]
     192.168.1.12 192.168.1.11 192.168.1.x 192.168.1.x
    (Debian-Client) (Debian-Slave)  (Client test)  (Machines externes)

**Mécanisme**

1. Le serveur exporte des répertoires via <mark>/etc/exports</mark>
2. Les clients montent ces répertoires avec la commande <mark>mount</mark>
3. Les accès sont contrôlés par les permissions NFS et les options d'export
4. Les UID/GID sont préservés (important avec NIS)

## 2.4. Installation des paquets (Debian)

### Sur le serveur (Debian-Master)

  ```bash
  # Mise à jours des paquets
  sudo apt install

  # Installation du serveur NFS
  sudo apt install nfs-kernel-server nfs-common rpcbind

  # Vérification des paquets installés
  dpkg -l | grep nfs
  dpkg -L nfs-kernel-server | grep -E "(bin|sbin)"
  ```
### Sur les clients (Debian-Slave, Debian-Client)

  ```bash
  sudo apt update
  sudo apt install nfs-common rpcbind
  dpkg -L nfs-common
  ```

## 2.5. Configuration du serveur NFS 

### Fichier <mark>/etc/exports</mark> 

<details>
  
  <summary>Qu'est-ce que le fichier /etc/exports</summary>


  Le fichier <mark>/etc/exports</mark> est le fichier de configuration principal du serveur NFS. Il définit quels répertoires sont partagés, avec quels clients, et avec quelles options de securité/permissions.

  ### Structure d'une ligne**

  ```bash
  # Format général
  /response/to/share  client1(options) client2(options) ...

  # Exemple :
  /home/nfs  192.168.1.0/24(rw,syn,no_subtree_check) *.tp.local(ro)
  ```

  ### Options courantes**

   Option | Type | Description 
   -------|------|-------------
   ro | accès | Read-Only : accès en lecture seule
   rw | accès | Read-Write : accès en lecture/écriture
   sync | comportement | Les écritures sont synchronisées
   async | comportement | Écritures asynchrones 
   no_subtree_check | comportement | Désactive la vérification des sous-arborescences
   subtree_check | comportement | Active la vérification
   root_squash | sécurité | Transforme l'UID root (0) en nobody (65534)
   no_root_squash | sécurité | Les accès root restent root
   all_squash | sécurité | Convertit tous les UID/GID en anonymes
   anonuid=xxx | sécurité | Spécifie l'UID pour les users anonymes
   anongid=xxx | sécurité | Spécifie le GID pour les users anonymes
   secure | network | Restreint les connexions aux ports < 1024 (par défaut)
   insecure | network | Autorise les connexions depuis des ports > 1024

</details>

  ```bash
  sudo nano /etc/exports
  ```
  ```text
  # Export du fichier /home pour tous les clients du réseau (Read/Write)
  /home    192.168.1.0/24(rw,sync,no_subtree_check,secure)

  # Export d'un repértoire partagé en lecture seule pour tous
  /opt/share  192.168.1.0/24(ro,sync,no_subtree_check,secure)

  # Export spécifique pour un client particulier (avec droits root)
  /mnt/disk   192.168.1.12(rw,sync,no_root_squash,no_subtree_check)

  # Export public (anonyme) pour tous
  /var/public  *(ro,sync,all_squash,anonuid=65534,anongid=65534)

  ```
**Aplication de la configuration**

  ```bash
  # Appliquer les changements
  sudo exportfs -ra

  # Vérifier les exports actuels
  sudo exportfs -v

  # Afficher ce qui est exporté
  showmount -e localhost
  ```

## 2.6. Lancement de NFS sur le serveur

### Vérificatoin des script de démarrage 

  ```bash
  # Vérifier que les services existent
  ls -la /etc/init.d/ grep -E "nfs|rpc"
  # ou avec systemd
  ls -la /lib/systemd/system/ | grep -E "nfs|rpc"
  ```

### Démarrage des services

  ```bash
  # Démarrer les services NFS
  sudo systemctl start nfs-kernel-server
  sudo systemctl start rpcbind

  # Activer au démarrage
  sudo systemctl enable nfs-kernel-server
  sudo systemctl enable rpcbind

  # Vérifier le statut
  sudo systemctl status nfs-kernel-server
  sudo systemctl status rpcbind

  # Vérifier que les services RPC sont enregistrés
  rpcinfo -p | grep -E "nfs|mountd|rpcbind"
  ```

### Surveillance de connexion

  ```bash
  # Voir quelles machines clientes montent quelles sources
  showmount -a

  # Voir les exports actuels via le systeme de fichiers proc
  cat /proc/fs/nfs/exports

  # surveillance en temps réel des connexions (si nfsstat est disponible)
  watch -n 2 nfsstat -c
  ```
<details>
  
  <summary>REMARQUE : Le fichier <mark><i>/proc</i></mark></summary>

  Le système de fichier /proc (pour "process information") est un système de fichiers virtuels présent sur tous les systèmes Linux/UNIX. Il n'existe pas physiquement sur le disque dur, mais est créé en mémoire par le noyau au démarrage.

  ### Caractéristiques Fondamentales

  **📁 C'est un pseudo-système de fichiers**

  - Il n'occupe aucun espace disque
  - Les fichiers sont générés dynamiquement à la demande
  - Il est monté automatiquement au boot sur <mark>/proc</mark>

  **🔍 Fonction principale**

  Il fournit une interface pour obtenir des informations sur :
  - L'état du système et du noyau
  - Les processus en cours d'exécution
  - La configuration matérielle
  - Les statistiques système

  ### Structure du répertoire /proc

  ```bash
  ls /proc
  ```
  **

  **1. Répertoires numérotés - Un par processus**

  ```bash
  /proc/1/    # Informations sur le processus PID 1 (init/systemd)
  /proc/1234  # Informations sur le processis PID 1234
  ```

  Chaque repértoire de processus contient :
  - *cmdline* : commande complète avec arguments
  - *cwd* : lien vers le repértoire de travail
  - *environ* : variables d'environnement
  - *exe* : lien vers l'exécutable
  - *fd/* : descripteurs de fichiers ouverts
  - *maps* : zones mémoire mappées
  - *status* : état détaillé du processus

  **2. Fichiers d'informations système**

  ```bash
  /proc/cpuinfo      # Infos détaillées sur le CPU
  /proc/meminfo      # Statistiques mémoire
  /proc/version      # Version du noyau
  /proc/uptime       # Temps depuis le démarrage
  /proc/loadavg      # Charge moyenne du système
  /proc/mounts       # Points de montage actuels
  /proc/filesystems  # Systèmes de fichiers supportés
  /proc/partitions   # Partitions détectées
  /proc/devices      # Périphériques
  /proc/interrupts   # Statistiques des interruptions
  /proc/iomem        # Carte mémoire E/S
  /proc/ioports      # Ports E/S utilisés
  ```

  **3. Fichiers de configuration dynamique**

  ```bash
  /proc/sys/  # Paramètres configurables du noyau
  ```

  C'est la partie plus importante pour l'administration

  ### Exemples d'utilisation

  **Obtenir des informations système**

  ```bash
  # Voire les infos CPU
  cat /proc/cpuinfo | head -10

  # Voir la mémoire dispinble
  cat /proc/meminfo | grep MemTotal

  # Voir la version du noyau
  cat /proc/version

  # Voir les partitions
  cat /proc/partitions

  # Voir la charge système
  cat /proc/loadavg
  ```

  **Explorer un processus spécifique**

  ```bash
  # Trouver le PID d'un processus (ex : bash, ici : 871)
  pgrep bash

  # Voir les infos du processus
  ls -la /proc/871
  cat /proc/871/status
  cat /proc/871/cmdline
  cat /proc/871/environ | tr '\0''\n' #variables d'environnemnt
  ```

  **Modifier des paramètres noyau (via /proc/sys)**

  ```bash
  # Voir si le forwarding IP est activé
  cat /proc/sys/net/ipv4/ip_forward

  # L'activer temporairement
  echo 1 | sudo tee /proc/sys/net/ipv4/ip_forward

  # Voir la taille max des fichiers
  cat /proc/sys/fs/file-max

  # Changer le hostname temporairement
  echo "new-name" | sudo tee /proc/sys/kernel/hostname
  ```

  ### Fichiers importants dans /proc/sys

  ```bash
  /proc/sys/kernel/hostname  #nom de la machine
  /proc/sys/kernel/domainname # nom de domaine NIS
  /proc/sys/kernel/osrelease  # Version du noyau
  /proc/sys/kernel/ostype   # type d'OS

  /proc/sys/net/ipv4/ip_forward  # Routage IP
  /proc/sys/net/ipv4/icmp_echo_ignore_all # Réponse ping

  /proc/sys/fs/file-max  # maximum de fichiers ouverts
  /proc/sys/fs/inotify/max_user_watches # surveillace fichiers
  ```
  ### Importances dans ce projet

  **Configuration du domaine NIS**

  ```bash
  # Le domaine NIS est stocké ici temporairement
  cat /proc/sys/kernel/domainname
  # correspond à la commande domainname

  # Pour le changer temporairement
  echo "TP-NIS" | sudo tee /proc/sys/kernel/domainname
  ```

  **Surveillance des services NFS**

  ```bash
  # Voir les exports NFS actuels
  cat /proc/fs/nfs/exports

  # Voir les clients NFS connectés
  cat /proc/fs/nfs/clients/*/info
  ```

  **Debug**

  ```bash
  # Voir les processus NIS/NFS
  ls /proc/ | grep -E "ypserv|ypbind|nfs" | xargs -I {} cat /proc/{}/status
  ```

  ### Particularité

  **📝 Lecture seule vs Lecture/ecriture**
  
  - La plupart des fichiers dans /proc sont en lecture seule
  - Ceux dans /proc/sys peuvent souvent être modifiés

  **🔄 Temporaire**

  Les modifications via /proc/sys sont perdues au redémarrage. Pour les rendre permanentes :

  ```bash
  # Editer /etc/sysctl.conf
  sudo nano /etc/sysctl.conf
  net.ipv4.ip_forward=1
  #puis appliquer
  sudo sysctl -p
  ```

  **📊 Statistique en temps réel**

  ```bash
  # Voir les statistique réseau en direct
  watch -n 1 cat /proc/net/dev

  # Surveiller les interruptions
  watch -n 1 cat /proc/interrupts
  ```
  ### 
</details>

## 2.7. Configuration et utilisation côté client

### Montage manuel d'une ressource NFS

  ```bash
  # Syntaxe : mount -t nfs serveur:chemin_distant point_de_montage

  # Création d'un point de montage

  sudo mkdir -p /mnt/home

  # Montage de /home du serveur

  sudo mount -t nfs 192.168.1.10:/home /mnt/home

  # Vérification du montage
  df -h | grep nfs
  mount | grep nfs
  ```

  **Options de montage importantes**

  | Option | Description |
  |--------|-------------|
  | hard | Montage avec écriture bloquante (attend indéfiniment) |
  | soft | Montage avec écriture bloquante (timeout après un temps) |
  | intr | Montage interruptible (utile avec hard en cas de blocage) |
  | fg | Montage en forground (premier plan) |
  | bg | Montage en background (arrière plan) - utile si serveur lent |
  | rsize/wsize | Taille des buffers de lecture/écriture |
  | timeo | timeout en dixième de secondes |
  | retrans | Nombre de retransmissions avant timeout |

### Montage automatique avec /etc/fstab

  ```bash
  sudo nano fstab
  ```
Ajouter les lignes : 

  ```text
  192.168.1.10:/home  /mnt/home  nfs rw,soft,intr,bg 0 0
  192.168.1.10:/opt/share  /mnt/share nfs ro,soft,bg 0 0
  ```
tester le montage automatique

  ```bash
  # Tester la configuration fstab
  sudo mount -a

  # Vérifier
  mount | grep nfs
  ```
<details>
  <summary>Qu'est-ce que le fichier <mark><i>/etc/fstab</i></mark></summary>

  Le fichier <mark><i>/etc/fstab</i></mark> (pour File System Table) est un fichier de configuratoin système qui définit comment et où les systèmes de fichiers doivent être montés automatiquement au démarrage.

  #### Structure d'une ligne

  Chaque ligne du fichier /etc/fstab représente un système de fichiers à monter et suit ce format 

  ```bash
  <device> <point_de_montage> <type> <options> <dump> <pass>
  ```

  **Détail des 6 champs**

  1. <device> : Le périphérique ou système de fichiers à monter
    - <mark>/dev/sda1</mark> (partition disque)
    - <mark>UUID=1234-5678</mark> (identifiant unique universel - recommandé)
    - <mark>LABEL=MonDisque</mark> (étiquette de volume)
    - <mark>serveur:/partage</mark> (pout NFS)
    - <mark>proc, tmpfs</mark> (systemes virtuels)

  2. <point_de_montage> : Où monter le système de fichiers
    - <mark>/, /home, /boot, /mnt/data</mark>, etc.

  3. <type> : Type de système de fichiers
    - <mark>ext4, xfs, btrfs</mark> (systèmes Linux)
    - <mark>ntfs, vfat</mark> (systèmes Windows)
    - <mark>nfs, nfs4</mark> (partages réseau)
    - <mark>swap</mark> (espace d'échange)
    - <mark>proc, sysfs, tmpfs</mark> (virtuels)

  4. <options> : Options de montage (séparées par des virgules)
    - <mark>default</mark> : Options par défaut (rw,suid,dec,exec,auto,nouser,async)
    - <mark>ro/rw</mark> : Lecture seule / lecture-écriture
    - <mark>nosuid</mark> : Ignorer les bits suid
    - <mark>noexec</mark> : Interdire l'exécution de programmes
    - <mark>nodev</mark> : ne pas interpréter les fichiers de périphériques
    - <mark>noauto</mark> : ne pas monter automatiquement au démarrage
    - <mark>user/nouser</mark> : permettre/interdire aux utilisateurs de monter
    - <mark>quota</mark> : activer les quotas
    - <mark>noatime</mark> : ne pas mettre à jours les temps d'accès

  5. <dump> : utiliser par la commande dump pour les sauvegardes
    - <mark>0</mark> : ne pas sauvegarder
    - <mark>1</mark> : sauvegarder

  6. <pass> : Ordre de vérification par fsck au démarrage
    - <mark>0</mark> : ne pas vérifier
    - <mark>1</mark> : vérifier en premier (pour la partition root <mark>/</mark>)
    - <mark>2</mark> : vérifier après (pour les autres partitions)

  #### Commandes essentiels 

  **Vérifier le fichier**

  ```bash
  cat /etc/fstab
  # Voir les UUIDs des partitions
  sudo blkid
  ```

  **Monter selon fstab**

  ```bash
  # Monter tout ce qui est dans fstab (sauf noauto)
  sudo mount -a

  # Monter une entrée spécifique
  sudo mount /home
  sudo mount /mnt/data
  ```

  **Tester avant d'écrire**

  ```bash
  #Vérifier la syntaxe du fichier
  sudo findmnt --verify
  #ou
  sudo mount -a 2>&1 | grep -i error
  ```

  #### Exemple : Configuraton de /etc/fstab sur les clients pour monter automatiquement les partages NFS :

  **Pour montage NFS permanent**

  ```bash
  # /etc/fstab sur les clients

  # Montage du répertoire home partagé via NIS/NFS
  master.tp.local:srv/nfs/home  /home/nfs  nfs rw,hard,intr,noatime,rsize=32768,wsize=32768 0 0
  # Montage d'un repértoire public
  master.tp.local:/srv/nfs/public /mnt/public nfs ro,soft,intr 0 0
  ```

  **Avec autofs**

  Au lieu de montages permanents dans /etc/fstab, on utilise souvent autofs avec NIS : 

  ```bash
  # /etc/auto.master
  /net /etc/auto.nfs

  # /etc/auto.nfs
  home  -rw,soft,intr  master.tp.local:/srv/nfs/home
  ```
</details>

## 2.8.Sécurité NFS

⚠️ NFS n'est pas un protocole très sécurisé (surnommé "No File Security")

### Vulnérabilités principales 

1. Authentification faible : basée uniquement sur l'adresse IP (spoofing possible)
2. Identification par UID : Usurpation possible si un client a un utilisateur avec le même UIS
3. Root squashing : Par défaut, root est mappé à nobody (mais contournable)
4. Transactions non cryptées : Tout circule en clair sur le réseau

### Recommandations de sécurit

1. Utilisation en réseau isolé
2. Restrictions avec /etc/hosts.deny et /etc/hosts.allow
    ```bash
    sudo nano /etc/hosts.deny
    ```
    ```text
    # Refuser tout accès NFS par défaut
    rpcbind:ALL
    mountd:ALL
    nfsd:ALL
    statd:ALL
    lockd:ALL
    ```
    ```bash
    sudo nano /etc/hosts.allow
    ```
    ```text
    # Autoriser uniquement le réseau local
    rpcbind:192.168.1.0/24
    mountd:192.168.1.0/24
    nfsd:192.168.1.0/24
    statd:192.168.1.0/24
    lockd:192.168.1.0/24
    ```
3. Options de sécurité dans /etc/exports
    ```bash
    sudo nano /etc/exports
    ```
    ```text
    # Options sécurisées
    /home 192.168.1.0/24(rw,sync,root_squash,no_subtree_check)
    /opt/share 192.168.1.0/24(ro,syn,root_squash,no_subtree_check)

    # Eviter absolument (sauf cas très scpécifiques) :
    # no_root_squash # garde les droits root - DANGEREUX
    # insecure # autorise les ports > 1024

