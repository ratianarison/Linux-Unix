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
