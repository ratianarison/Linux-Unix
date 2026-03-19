# DNS (Domain Name System)


## 📌 Définition

Le DNS est un service fondamental d'Internet qui traduit les noms de domaine (comme google.com) en adresses IP (comme 142.250.178.46). 


## 🌐 Principes de base

**Sans DNS**

```bash
# Pour accéder à un site, il faudrait retenir :
ping 142.250.178.46
ssh 142.250.178.46
```

**Avec DNS**

```bash
# On utilise des doms faciles à retenir :
ping google.com
ssh master.tp.local
```

## Architecture du DNS

```text
DNS a une structure hiérarchique définie comme suit :

Root (.)
.
├────  .com ( Domaine de primier niveau - TLD)
|       ├── google.com  (domaine de deuxième niveau)
|            ├── mail.google.com (FQDN)
|       ├── amazon.com
|       ├── example.com
├────  .org
|       ├── wikipedia.org
├────  .local
        ├── tp.local
             ├── master.tp.local  (FQDN)
```

## 📚 Notions Importantes

### 1. FQDN (Full Qualified Domain Name)

Le FQDN est le nom de domaine complet et unique d'une machine sur Internet. Il comprend tous les éléments de la hiérarchie DNS, du nom d'hôte jusqu'à la racine.

**Format :** 

```text
[hostname].domain.[tld].     ← Point final important
```

**Exemples :**

```bash
# FQDN valides
google.com.    # Le point final représente la racine
www.google.com.
mail.google.com.
ftp.example.org.

# Dans le projet :
master.tp.local    # FQDN du serveur
slave.tp.local
client.tp.local
```

### 2. Domaine

Un domaine est un espace de noms dans l'arbre DNS. C'est un regroupement logique de machines sous une même autorité administrative.

**Hiérarchie des domaines**

```bash
.                          # Racine
├── com                    # Domaine de premier niveau (TLD)
│   ├── google             # Domaine de deuxième niveau
│   │   ├── mail           # Sous-domaine
│   │   ├── drive
│   │   └── calendar
│   └── amazon
├── org
│   └── wikipedia
└── fr                      # TLD national
    ├── google
    ├── amazon
    └── education
        └── univ-paris      # Sous-domaine
```

**Types de domaines**

Domaine de premier niveau (TLD)

```bash
.com    # commercial
.org    # organisation
.net    # network
.fr     # France
.mg     # Madagascar
.local  # Privé
```

Domaine de deuxième niveau

```bash
google.com    # Google dans le .com
amazon.fr     # Amazon dans le .fr
tp.local      # Le projet TP dans .local
```

Sous domaine 

```bash
mail.google.com      # Service mail de google
drive.google.com     # Service drive de google
master.tp.local      # le serveur dans le domaine principal tp.local
```

### 3. Zone

Une zone est une portion de l'arbre DNS qui est gérée par une même entité andministrative. C'est l'unité de délégation et de configuration.

Domaine : Concept hiérarchique
Zone : Portion du domaine gérée par un serveur

**Exemple de zone**

```bash

; Fichier de zone pour tp.local
$ORGIN tp.local
$TTL 86400

@ IN SOA master.tp.local. admin.tp.local. (
  2024031901 ; Serial
  3600 ; Refresh
  1800 ; Retry
  604800 ; Expire
  86400; Minimum

; Serveurs de noms pour cette zone

  IN NS master.tp.local.
  IN NS slave.tp.local.

; Entregistrement dans la zone
  master IN A 192.168.1.10
  slave IN A 192.168.1.11
  client IN A 192.168.1.12
```

**Types de zones**

Zones primaires (master)

```bash
# Fichier modifiable manuellement
/var/lib/bind/tp.local.hosts
```

Zones Secondaires (slave)

```bash
# Copie automatique depuis le master
/var/lib/bind/tp.local.hosts  # read-only
```

Zone directe

```bash
# Nom - IP
tp.local 192.168.1.10
```

Zone inverse

```bash
# IP → Nom (pour la résolution inverse)
192.168.1.10 master.tp.local
```

### 4. Délégation

La délégation est le mécanisme par lequel un serveur DNS transmet l'autorité pour une partie de l'arbre DNS à un autre serveur.

**Principe**

```bash
                  Root Server
                        ↓ Délégue .com à Verisign
                  Serveurs .com
                        ↓ Délègue google.com à Google
                  Serveurs google.com (Google)
                        ↓ Délègue mail.google.com (optionnel)
                  Serveurs mail.google.com
```

**Comment ça marche techniquement**

1. Enregistrements NS dans la zone parente

```bash
; Dans la zone .com
google.com. IN NS ns1.google.com
google.com. IN NS ns2.google.com
```

2. Enregistrements "glue" (A)

```bash
; Dans la zone .com (nécessaire si le serveur est dans le domaine déléqué)
ns1.google.com. IN A 216.239.32.10
ns2.google.com. IN A 216.239.34.10
```

**Exemple de délégation (projet)**

Si on a un domaine public tp.com

```bash
; Dans la zone .com (gérée par Verisign)
tp.com IN NS master.tp.com.
tp.com IN NS slave.tp.com.

; Glue records (car master.tp.com est DANS tp.com)
master.tp.com. IN A 192.168.1.10
slave.tp.com. IN A 192.168.1.11
```

Puis dans la zone locale 

```bash
; Sur le serveur master.tp.local (zone tp.com)
$ORIGIN tp.com.

@ IN SOA master.tp.local. admin.tp.com. (...)
master IN A 192.168.1.10
slave IN A 192.168.1.11
client IN A 192.168.1.12

www IN CNAME master
```

## 📋 Types d'enregistrements DNS

Un enregistrement DNS est une instruction stockée sur un serveur DNS, dans un fichier de zone. Ces enregistrements fournissent des informations importantes sur les domaines et les noms d'hôte. Ces listes aident les serveurs DNS à envoyer les requêtes au bon endroit.

Chaque type d'enregistrement DNS a un rôle spécifique.

### 1. Enregistrement A (Address Record)

Convertir un nom de domaine en adresse IPv4 (32 bits). C'est l'enregistrement le plus fondamental.

**Syntaxe**

```text
nom. TTL IN A IPv4_address
```

**Exemples**

```bash
# Exemple simple
google.com. 300 IN A 142.250.178.46

# Plusieurs noms vers la même IP
example.com. 3600 IN A 192.168.1.10
www.example.com 3600 IN A 192.168.1.10
mail.examle.com 3600 IN A 192.168.1.10

# Plusieurs IP pour le même nom (Round-robin)
google.com. 300 IN A 142.250.178.46
google.com. 300 IN A 142.250.178.47
google.com. 300 IN A 142.250.178.48
```
### 2. Enregistrement AAAA (Quad-A Record)

Convertit un nom de domaine en adresse IPv6 (128 bits).

**Syntaxe**

```text nom. TTL AAAA IPv6_address```

**Exemples**

```bash
google.com. 300 IN AAAA 2a00:1450:810:200e
ipv6.google.com. 3600 AAAA 2001:4860:4860::8888
```

**Utilisation**

```bash
# Vérifier si un site supporte IPv6
dig AAAA google.com
```

### 3. Enregistrement CNAME (Canonical Name)

Crée un alias vers un autre domaine. Le nom canonique (officiel) reçoit les requêtes.

Un CNAME ne peut pas coexister avec d'autres enregistrements pour le même nom.

**Syntaxe**

```text alias. TTL IN CNAME canonical_name```

**Exemple**

```bash
# Alias courants
www.example.com. 3600 IN CNAME example.com.
ftp.example.com. 3600 IN CNAME example.com.
blog.example.com. 3600 IN CNAME example.com.

# Chaîne de CNAME
short.example.com. IN CNAME tiny.example.com
tiny.example.com. IN CNAME example.com
```

**Dans le projet**

```bash
# Si on veut que wwww pointe vers master :
www.tp.local. 86400 IN CNAME master.tp.local.

# Pour un service web
web.tp.local. 86400 IN CNAME master.tp.local.
```

### 4. Enregistrement MX (Mail Exchange)

Indique les serveurs de messagerie qui acceptent les emails pour le domaine.

- Plus le nombre est petit, plus la priorité est haute
- Le serveur avec la plus petite priorité est contacté en premier

**Syntaxe**

```text domaine. TTL IN MX priority mail_server```

**Exemples**

```bash
# Configuration typique

example.com. 3600 IN MX 10 mail1.example.com
example.com. 3600 IN MX 20 mail2.example.com
example.com. 3600 IN MX 30 mail3.example.com

# Google workspace (ex G Suite)
example.com.    3600    IN  MX  10  aspmx.l.google.com.
example.com.    3600    IN  MX  20  alt1.aspmx.l.google.com
example.com.    3600    IN  MX  20  alt2.aspmx.l.google.com.
example.com.    3600    IN  MX  30  alt3.aspmx.l.google.com.
example.com.    3600    IN  MX  30  alt4.aspmx.l.google.com.
```

**Dans le projet**

```bash
tp.local.       86400   IN  MX  10  mail.tp.local.
mail.tp.local.  86400   IN  A   10.237.80.115
```

### 5. Enregistrement NS (Name Server)

Déclare les serveurs DNS faisant autorité pour un domaine. C'est l'enregistrement qui permet la délégation.

**Syntaxe**

```text domaine. TTL NS dns_server```

**Exemples**

```bash
# Pour un domaine avec plusieurs serveurs DNS
example.com. 172800 IN NS ns1.example.com.
example.com. 172800 IN NS ns2.example.com.
example.com. 172800 IN NS ns3.example.com.

# Pour un sous-domaine délégué
sub.example.com. 172800 IN NS ns1.sub.example.com
sub.example.com. 172800 IN NS ns2.sub.example.com

# Dans le projet
tp.local. 86400 IN NS master.tp.local.
tp.local. 86400 IN NS master.tp.local.
```
### 6. Enregistrement PTR (Pointer)

Réalise la résolution inverse : convertit une adresse IP en nom de domaine. Utilisé pour la vérification (anti-spam, authentification).

Les enregistrements PTR sont dans des domaines spéciaux : 
- IPv4 : in-addr-arpa.
- IPv6 : ip6.rpa.

**Syntaxe**

```text ip_reverse.in-addr.arpa. TTL IN PTR nom.```

**Exemples**

```bash
# Pour l'IP 192.168.1.10
192.168.1.10.in-addr.arpa. 3600 IN PTR example.com

# Pour l'IP 8.8.8.8 (Google DNS)
8.8.8.8.in-addr.arpa. 86400 IN PTR master.tp.local

# Vérification
dig -x 8.8.8.8
nslookup
```

