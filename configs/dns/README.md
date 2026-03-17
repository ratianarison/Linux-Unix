
# PARTIE 3 : MISE EN PLACE DNS (Domain Name System)

## 3.1. Présentation du DNS 

<details>
  <summary><b>Le DNS (Domain Name System)</b></summary>

  Le DNS (Domain Name System) est un service fondamental d'Internet qui traduit les noms de domaine (comme google.com) en adresses IP (comme 142.250.178.46).
  C'est un système de nommage par base de données répartie.

  ### 🌐 Principe de base 

  **Sans DNS**

  ```bash
  # Pour accéder à un site, il faudrait retenir :
  ping 142.250.178.46
  ssh 192.168.1.10 # le serveur master
  ```
  
  **Avec DNS**

  ```bash
  # On utilise des noms faciles à retenir :
  ping google.com
  ssh user@master.tp.local
  ```

  ### 🏗️ Architecture de DNS

  #### 1. Architecture hiérarchique

  ```text
  Racine (.)
    ├── .com (domaine de premier niveau - TLD)
    │     ├── google.com
    │     ├── amazon.com
    │     └── example.com
    ├── .org
    │     └── wikipedia.org
    ├── .fr
    │     ├── google.fr
    │     └── education.gouv.fr
    └── .local (domaine privé)
          └── tp.local
                ├── master.tp.local
                ├── slave.tp.local
                └── client.tp.local
  ```

  #### 2. Types de serveurs DNS**

  **Les 4 types principaux de serveurs DNS**

  ***1. Serveur Racine (Root Server)***

  Ce sont les serveurs les plus élevés dans la hiérarchie DNS. Il y a 13 adresses IP de serveurs racines (nommées de <mark>a.root-servers.net</mark> à <mark>m.root-servers.net</mark>, gérés par 12 organisations différentes.
  Tous ces serveurs partagent le même fonction critique : ils constituent la première étape de la résolution de noms sur Internet. Lorsqu'un résolveur DNS ne connaît pas l'adresse d'un domaine, il interroge l'un de ces serveurs racines qui le redirige vers le serveur faisant autorité pour l'extension concernée (TLD).

  | Identifiant | Organisation responsable | Localisation principale | Adresse IPv4 | Adresse IPv6 |
  |-------------|--------------------------|-------------------------|--------------|--------------|
  | A (a.root-servers.net) | Verisign, Inc | Virginie, USA | 198.41.0.4 | 2001:503:ba3e::2:30 |
  | B (b.root-servers.net) | USC-ISI (Information Science Institute) | Californie, USA | 199.9.14.201 | 2001:500:200::b |
  | C (c.root-servers.net) | Cogent Communications | Virginie, USA | 192.33.4.12 | 2001:500:2::c
  | D (d.root-servers.net) | University of Maryland | Maryland, USA | 199.7.91.13 | 2001:500:2d::d
  | E (e.root-servers.net) | NASA (Ames Research Center) | Californie, USA | 192.203.230.10 |	2001:500:a8::e
  | F (f.root-servers.net) | ISC (Internet Systems Consortium, Inc.) | Palo Alto, USA | 192.5.5.241 | 2001:500:2f::f
  | G (g.root-servers.net) | U.S. Departement of Defense | Ohio, USA | 192.112.36.4 | 2001:500:12::d0d 
  | H (h.root-servers.net) | U.S. Army (U.S. Army Research Laboratory) | Maryland, USA | 198.97.190.53 | 2001:500:1::53
  | I (i.root-servers.net) | Netnod (Autonomical) | Stockholm, Suède | 192.36.148.17 | 2001:7fe::53
  | J (j.root-servers.net) | Verisign, Inc. | Virginie, USA | 192.58.128.30 | 2001:503:c27::2:30
  | K (k.root-servers.net) | RIPE NCC | Amsterdam, Pays-Bas | 193.0.14.129 | 2001:7fd::1
  | L (l.root-servers.net) | ICANN | Californie, USA | 199.7.83.42 | 2001:500:9f::42
  | M (m.root-servers.net) | WIDE Project | Tokyo, Japon | 202.12.27.33 | 2001:dc3::35

  Bien qu'il n'y a que 13 adresses IP (de A à M), il existe en réalité des centaines de serveurs physiques répartis mondialement grâce à la technologie <mark>anycast</mark>. Cela permet à un utilisateur en France d'interroger une instance du serveur "K" située physiquement en Europe plutôt qu'aux Pay-Bas.

  
  *Le fichier "Root Hints**

  Ces 13 adresses sont répertoriées dans un petit fichier texte appelé <b>named.cache</b> (ou root hints). C'est le bottin de base que tout serveur DNS télécharge au démarrage pour savoir à qui parler en premier.
  
  ***Serveur TLD (Top Level Domain)***

  Le serveur TLD est l'étape juste après le root-servers dans la hiérarchie du DNS. C'est lui qui gère une extension spécifique (comme .com, .fr, .net).

  - Un serveur TLD pour chaque extension (.com, .net, .org, .fr, etc.)
  - Connaissent les serveurs faisant autorité pour chaque domaine de leur extension
  - Gérés par des registres (Verisign pour .comme, AFNIC pour .fr)

  Si le serveur racine est le réceptionniste qui indique quel département aller voir, le serveur TLD est le chef du département. Il connaît l'emplacement exact de tous les noms de domaines enregistrés sous son extension.

  Il existe deux types de TLD : 

  - gTLD (Generic) : Les extensions générales comme .com, .org, .edu, .gov, .net
  - ccTLD (Country Code) : les extensions liées à un pays comme .fr (France), .be (belgique), .ca (Canada), .mg (Madagascar)

  

  ### 📝 Types d'enregistrement

  ```bash
  #Enregistrements courants :

  A → 192.168.1.10    # Adresse IPv4
  AAAA → 2001:db8::1  # Adresse IPv6
  CNAME → www → example.com  # Alias (canonical name)
  MX → mail.example.com      # Serveur de mail
  NS → ns1.example.com       # Serveur de noms (Name Server)
  PTR → 10.1.168.192.in-addr.arpa   # Résolution inverse (IP → nom)
  TXT → "v=spf1 include:_spf.google.com"   # Texte (SPF, vérifications)
  SOA → ...    # Start of Authority (information de zone)
  ```

  ### 🔄 Fonctionnement d'une requête DNS

  Quand on tape <mark>google.com</mark> :

  ```bash
  1. Le PC consulte le cache local
  2. Si pas en cache -> demande au resolver (ex : 8.8.8.8)
  3. Le resolver demande aux serveurs racines
  4. Racine indique : "demandez au serveurs .com"
  5. Serveurs .com indique : "demandez aux serveurs de google.com"
  6. Serveurs de google.com donnent l'IP
  7. Resolver retourne l'IP au PC
  8. Le PC se connecte à l'IP
  ```

  ### 📓 Notions importantes 

  | Terme | Description | Exemple |
  |-------|-------------|---------|
  | FQDN | Fully Qualified Domain Name (nom absolu) | master.tp.local |
  | Domaine | Sous-arbre de l'espace de nommage | tp.local |
  | Zone | Partie du domaine gérée par un serveur | zone tp.local |
  | Délegation | Transfert d'autorité à un sous-domaine | paris.tp.local délégué |

  
  
</details>
