#PARTIE 1 : MISE EN PLACE DE NIS (Network Information Service)

##1.1. Présentation de NIS
NIS (Network Information Service), anciennement appelé "Yellow Pages" (YP), est un protocole qui constitue une base de données répartie pour les fichiers de configuration UNIX. Il permet de centraliser les fichiers de configuration sur un serveur NIS pour éviter de dupliquer des fichiers sur chaque machine d'un réseau.

###Objectif : 
Un utilisateur peut se connecter sur n'importe quelle machine du réseau avec le même login/password, et les informations système (passwd, group, hosts, etc.) sont partagées.

##1.2. Services/Démons NIS

**Portmapt (ou rpcbind)**

**. Rôle** : Mise en correspondance des numéros de ports TCP/IP avec les numéros de processus RPC

**. Fichier de référence** : */etc/rpc* contient les numéros RPC réservés

**. Paquet Debian** : rpcbind

**Côté serveur**

| Démon | Rôle | Paquet |
|-------|------|--------|
| ypserv| Implémente le serveur NIS principal | nis |
| rpc.yppasswdd| Permet de changer un mot de pass sur le serveur NIS depuis un client | nis |
| rpc.ypxfrd | Accélère les trasnfers entre serveur maître et esclave| nis |

**Côté client**

| Démon | Rôle | Paquet|
|-------|------|-------|
| ypbind | Implémente le client NIS | nis |
| yp-tools | Outils de diagnostic (ypwich, ypcat, ypmatch) | nis |

##1.3. Principe de Fonctionnement 

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
                    192.168.1.x (collègues)

**Mécanisme**

1. Le serveur maître maintient les bases de données maîtresses
2. Les esclaves se synchronisent 
