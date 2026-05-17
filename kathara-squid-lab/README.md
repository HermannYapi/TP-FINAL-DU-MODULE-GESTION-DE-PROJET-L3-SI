# TP RÉSEAU — Configuration du Caching via Proxy Squid (Mode Mandataire)
## Rapport & Documentation Complète — Kathara Lab

---

## TABLE DES MATIÈRES

1. [Objectif du TP](#1-objectif-du-tp)
2. [Architecture du réseau](#2-architecture-du-réseau)
3. [Tableau des adresses IP](#3-tableau-des-adresses-ip)
4. [Rôle des ACL Squid](#4-rôle-des-acl-squid)
5. [Rôle du NAT sur R1](#5-rôle-du-nat-sur-r1)
6. [Structure des fichiers Kathara](#6-structure-des-fichiers-kathara)
7. [Lancement du Lab](#7-lancement-du-lab)
8. [Commandes de tests](#8-commandes-de-tests)
9. [Interprétation des logs](#9-interprétation-des-logs)
10. [Conclusion](#10-conclusion)

---

## 1. OBJECTIF DU TP

Ce TP met en pratique la configuration d'un **Proxy Cache de type mandataire (forward proxy)**
via **Apache Squid** dans un réseau simulé avec **Kathara**.

### Compétences visées :
- Déployer Squid en mode **proxy mandataire** (clients configurent le proxy explicitement)
- Activer et optimiser le **cache** (RAM + disque) pour réduire la consommation de bande passante
- Implémenter un **filtrage d'accès par ACL** (blacklist de domaines et mots-clés)
- Configurer une **exception d'accès total** pour un hôte spécifique (PC1)
- Mettre en place le **NAT** sur un routeur pour l'accès à Internet
- Analyser les logs Squid : **TCP_HIT** (cache) vs **TCP_MISS** (serveur réel)

---

## 2. ARCHITECTURE DU RÉSEAU

```
=========================
||     INTERNET        ||
||  nginx Web Server   ||
||   10.0.0.2/24       ||
=========================
          |
    10.0.0.0/24
          |
    eth1  |
     +---------+
     |   R1    |   → NAT MASQUERADE (POSTROUTING sur eth1)
     | Routeur |   → IP Forwarding activé
     +---------+
     | eth0
192.168.2.1/24
          |
    Réseau PROXY-R1 (192.168.2.0/24)
          |
192.168.2.100
     eth1
 +-----------+
 |   PROXY   |   → Squid écoute sur 3128
 |   Squid   |   → Cache RAM 256MB + Disque 1Go
 +-----------+   → Blacklist + Exception PC1
     eth0
192.168.1.100
(= Gateway de tous les clients)
          |
     +---------+
     | SWITCH  |
     +---------+
  ─────────────────────────────────────
  |          |          |             |
PC1        PC2        PC3           PC4
192.168.1.10  192.168.1.20  192.168.1.30  192.168.1.40
[EXCEPTION]   [FILTRÉ]      [FILTRÉ]      [FILTRÉ]
```

### Flux de trafic :
```
Client (PC) → [Proxy Squid :3128] → [R1 NAT] → [INTERNET nginx]
                    ↕
                [Cache HIT]  ← Réponse depuis le cache (rapide)
                [Cache MISS] → Requête transmise à Internet
```

---

## 3. TABLEAU DES ADRESSES IP

| Machine    | Interface | Adresse IP        | Réseau           | Rôle                        |
|------------|-----------|-------------------|------------------|-----------------------------|
| PC1        | eth0      | 192.168.1.10/24   | 192.168.1.0/24   | Client — ACCÈS TOTAL        |
| PC2        | eth0      | 192.168.1.20/24   | 192.168.1.0/24   | Client — Filtrage actif     |
| PC3        | eth0      | 192.168.1.30/24   | 192.168.1.0/24   | Client — Filtrage actif     |
| PC4        | eth0      | 192.168.1.40/24   | 192.168.1.0/24   | Client — Filtrage actif     |
| PROXY eth0 | eth0      | 192.168.1.100/24  | 192.168.1.0/24   | Gateway clients + Port 3128 |
| PROXY eth1 | eth1      | 192.168.2.100/24  | 192.168.2.0/24   | Lien vers R1                |
| R1         | eth0      | 192.168.2.1/24    | 192.168.2.0/24   | Passerelle Proxy → WAN      |
| R1         | eth1      | 10.0.0.1/24       | 10.0.0.0/24      | Interface Internet (NAT)    |
| INTERNET   | eth0      | 10.0.0.2/24       | 10.0.0.0/24      | Serveur Web nginx           |

### Configuration réseau des clients :

| Client | IP             | Masque        | Gateway (Proxy) | Proxy HTTP       |
|--------|----------------|---------------|-----------------|------------------|
| PC1    | 192.168.1.10   | 255.255.255.0 | 192.168.1.100   | 192.168.1.100:3128 |
| PC2    | 192.168.1.20   | 255.255.255.0 | 192.168.1.100   | 192.168.1.100:3128 |
| PC3    | 192.168.1.30   | 255.255.255.0 | 192.168.1.100   | 192.168.1.100:3128 |
| PC4    | 192.168.1.40   | 255.255.255.0 | 192.168.1.100   | 192.168.1.100:3128 |

---

## 4. RÔLE DES ACL SQUID

### Qu'est-ce qu'une ACL ?

Une **ACL (Access Control List)** est une règle nommée qui identifie un élément du trafic :
- une adresse IP source (`src`)
- un domaine de destination (`dstdomain`)
- un motif dans l'URL (`urlpath_regex`)
- une méthode HTTP (`method`)
- un numéro de port (`port`)

> ⚠️ **Important** : Une ACL seule ne fait rien. C'est la directive `http_access allow/deny`
> qui utilise l'ACL pour prendre une décision.

### Tableau des ACL définies dans ce TP :

| Nom ACL              | Type          | Valeur(s)                                      | Rôle                          |
|----------------------|---------------|------------------------------------------------|-------------------------------|
| `lan_clients`        | src           | 192.168.1.0/24                                 | Tout le réseau clients        |
| `localhost`          | src           | 127.0.0.1/32                                   | Loopback                      |
| `Safe_ports`         | port          | 80, 443, 21, 70...                             | Ports HTTP légitimes          |
| `SSL_ports`          | port          | 443                                            | HTTPS uniquement              |
| `CONNECT`            | method        | CONNECT                                        | Tunnels SSL                   |
| `pc1_exempt`         | src           | 192.168.1.10/32                                | PC1 — exception totale        |
| `clients_filtres`    | src           | 192.168.1.20, .30, .40                         | PC2, PC3, PC4                 |
| `blacklist_youtube`  | dstdomain     | .youtube.com, .ytimg.com, .googlevideo.com...  | Blocage YouTube               |
| `blacklist_social`   | dstdomain     | .facebook.com, .instagram.com, .tiktok.com...  | Blocage réseaux sociaux       |
| `blacklist_streaming`| dstdomain     | .netflix.com, .twitch.tv                       | Blocage streaming             |
| `blacklist_adult`    | dstdomain     | .pornhub.com, .xvideos.com, .xnxx.com          | Blocage contenu adulte        |
| `blacklist_misc`     | dstdomain     | .roblox.com, .discord.com                      | Blocage divers                |
| `blacklist_keywords` | urlpath_regex | /porn/, /xxx/, /torrent, /crack/...            | Mots-clés interdits dans URL  |

### Logique des règles `http_access` (ordre de traitement) :

```
Requête entrante
      │
      ▼
[1] Port non sûr ? ──── OUI ──→ DENY (bloqué)
      │ NON
      ▼
[2] CONNECT vers non-SSL ? ─ OUI ──→ DENY
      │ NON
      ▼
[3] Localhost ? ────────── OUI ──→ ALLOW (accès admin)
      │ NON
      ▼
[4] PC1 (192.168.1.10) ? ─ OUI ──→ ALLOW (exception totale !)
      │ NON
      ▼
[5] Client filtré + YouTube ? OUI → DENY
      │ NON
[5] Client filtré + Social ?  OUI → DENY
      │ NON
[5] Client filtré + Stream ?  OUI → DENY
      │ NON
[5] Client filtré + Adult ?   OUI → DENY
      │ NON
[5] Client filtré + Misc ?    OUI → DENY
      │ NON
[5] Client filtré + Keyword ? OUI → DENY
      │ NON
      ▼
[6] Client filtré, site OK → ALLOW
      │
      ▼
[7] LAN général → ALLOW
      │
      ▼
[8] Tout le reste → DENY
```

### Pourquoi `dstdomain` avec le point (`.youtube.com`) ?

Le point devant le domaine signifie **tous les sous-domaines** :
- `.youtube.com` bloque : `www.youtube.com`, `m.youtube.com`, `accounts.youtube.com`, etc.
- Sans le point, seul le domaine exact serait bloqué.

---

## 5. RÔLE DU NAT SUR R1

### Pourquoi le NAT est-il indispensable ?

Les clients du LAN utilisent des **adresses IP privées** (RFC 1918) :
- `192.168.1.x` → non routable sur Internet
- `192.168.2.x` → non routable sur Internet

Quand le Proxy Squid envoie une requête à Internet (`10.0.0.2`), la requête porte
l'adresse source `192.168.2.100`. Si le serveur Internet répond à cette adresse,
la réponse ne peut pas être routée (adresse privée).

Le **NAT MASQUERADE** sur R1 résout ce problème :

```
AVANT NAT (à la sortie de R1 vers Internet) :
  Source : 192.168.2.100  →  Traduit en : 10.0.0.1
  Dest   : 10.0.0.2              Dest   : 10.0.0.2

RETOUR (depuis Internet) :
  Source : 10.0.0.2        →  Traduit en : 192.168.2.100
  Dest   : 10.0.0.1              Dest   : 10.0.0.1
```

### Commandes iptables utilisées :

```bash
# Activer le forwarding IP
echo 1 > /proc/sys/net/ipv4/ip_forward

# NAT MASQUERADE : tout ce qui sort par eth1 est masqué derrière 10.0.0.1
iptables -t nat -A POSTROUTING -o eth1 -j MASQUERADE

# Autoriser le trafic établi en retour
iptables -A FORWARD -i eth1 -o eth0 -m state --state RELATED,ESTABLISHED -j ACCEPT

# Autoriser le trafic sortant
iptables -A FORWARD -i eth0 -o eth1 -j ACCEPT
```

### Différence entre NAT et MASQUERADE :
- **SNAT (Static NAT)** : remplace la source par une IP fixe → pour une IP externe fixe
- **MASQUERADE** : remplace la source par l'IP courante de l'interface → idéal pour IP dynamique ou simulation

---

## 6. STRUCTURE DES FICHIERS KATHARA

```
kathara-squid-lab/
│
├── lab.conf                    ← Topologie du lab (connexions réseau)
│
├── PC1.startup                 ← Config PC1 (exception)
├── PC2.startup                 ← Config PC2 (filtré)
├── PC3.startup                 ← Config PC3 (filtré)
├── PC4.startup                 ← Config PC4 (filtré)
├── PROXY.startup               ← Installation + démarrage Squid
├── R1.startup                  ← Config routeur + NAT
├── INTERNET.startup            ← Installation + démarrage nginx
│
└── PROXY/
    └── etc/
        └── squid/
            └── squid.conf      ← Configuration complète Squid
```

---

## 7. LANCEMENT DU LAB

```bash
# Se placer dans le répertoire du lab
cd kathara-squid-lab/

# Lancer tout le lab
kathara lstart

# Lancer une machine spécifique
kathara lstart PC1
kathara lstart PROXY

# Ouvrir un terminal dans une machine
kathara connect PC1
kathara connect PROXY

# Arrêter tout le lab
kathara lclean

# Supprimer complètement le lab
kathara lclean --all
```

---

## 8. COMMANDES DE TESTS

### 8.1 Vérification de la connectivité de base

```bash
# Depuis PC1 : ping vers le Proxy (gateway)
ping -c 3 192.168.1.100

# Depuis PC1 : ping vers R1
ping -c 3 192.168.2.1

# Depuis PC1 : ping vers Internet
ping -c 3 10.0.0.2

# Depuis PROXY : ping vers R1
ping -c 3 192.168.2.1

# Depuis PROXY : ping vers Internet
ping -c 3 10.0.0.2
```

### 8.2 Test du Cache (HIT / MISS)

```bash
# Depuis PC1 ou PC2 — Premier accès (→ TCP_MISS attendu)
curl -x http://192.168.1.100:3128 http://10.0.0.2/ -v

# Deuxième accès identique (→ TCP_HIT attendu si cache actif)
curl -x http://192.168.1.100:3128 http://10.0.0.2/ -v

# Observer les logs en temps réel sur PROXY
tail -f /var/log/squid/access.log

# Rechercher les HIT dans les logs
grep "TCP_HIT" /var/log/squid/access.log

# Rechercher les MISS dans les logs
grep "TCP_MISS" /var/log/squid/access.log
```

### 8.3 Test de filtrage (PC2, PC3, PC4 bloqués)

```bash
# Depuis PC2 — Tentative d'accès à YouTube (→ doit être BLOQUÉ)
curl -x http://192.168.1.100:3128 http://www.youtube.com/ -v

# Depuis PC2 — Tentative Facebook (→ BLOQUÉ)
curl -x http://192.168.1.100:3128 http://www.facebook.com/ -v

# Depuis PC2 — Tentative Netflix (→ BLOQUÉ)
curl -x http://192.168.1.100:3128 http://www.netflix.com/ -v

# Depuis PC2 — Accès site autorisé (→ doit fonctionner)
curl -x http://192.168.1.100:3128 http://10.0.0.2/ -v
curl -x http://192.168.1.100:3128 http://www.wikipedia.org/ -v
```

### 8.4 Test de l'exception PC1 (accès total)

```bash
# Depuis PC1 — YouTube (→ doit être AUTORISÉ car PC1 est exempt)
curl -x http://192.168.1.100:3128 http://www.youtube.com/ -v

# Depuis PC1 — Facebook (→ AUTORISÉ)
curl -x http://192.168.1.100:3128 http://www.facebook.com/ -v

# Depuis PC1 — Accès serveur local (→ AUTORISÉ)
curl -x http://192.168.1.100:3128 http://10.0.0.2/ -v
```

### 8.5 Test de filtrage par mots-clés

```bash
# Depuis PC2 — URL avec mot-clé interdit dans le chemin (→ BLOQUÉ)
curl -x http://192.168.1.100:3128 "http://10.0.0.2/adult/content" -v
curl -x http://192.168.1.100:3128 "http://10.0.0.2/files/torrent" -v

# URL normale (→ AUTORISÉ)
curl -x http://192.168.1.100:3128 "http://10.0.0.2/test1/" -v
```

### 8.6 Statistiques et gestion du cache

```bash
# Sur le PROXY — Afficher les statistiques du cache
squidclient -h 127.0.0.1 mgr:info

# Statistiques mémoire
squidclient -h 127.0.0.1 mgr:mem

# Statistiques IO (entrées/sorties)
squidclient -h 127.0.0.1 mgr:io

# Afficher les objets en cache
squidclient -h 127.0.0.1 mgr:objects | head -50

# Compter les HIT vs MISS
echo "=== Statistiques Cache ==="
echo "TCP_HIT  : $(grep -c TCP_HIT  /var/log/squid/access.log)"
echo "TCP_MISS : $(grep -c TCP_MISS /var/log/squid/access.log)"
echo "DENIED   : $(grep -c DENIED   /var/log/squid/access.log)"
```

### 8.7 Test du NAT sur R1

```bash
# Sur R1 — Afficher les connexions NAT actives
iptables -t nat -L -v -n

# Voir les connexions en cours (avec conntrack)
conntrack -L 2>/dev/null || cat /proc/net/ip_conntrack | head -20

# Vérifier les règles de forwarding
iptables -L FORWARD -v -n
```

### 8.8 Vérification de l'état de Squid

```bash
# Sur PROXY — Vérifier que Squid tourne
ps aux | grep squid

# Vérifier que le port 3128 est ouvert
ss -tlnp | grep 3128
netstat -tlnp | grep 3128

# Tester la configuration
squid -k check

# Recharger la configuration sans redémarrage
squid -k reconfigure

# Voir les logs système de Squid
tail -50 /var/log/squid/cache.log
```

---

## 9. INTERPRÉTATION DES LOGS

### Format d'un log Squid (`access.log`) :

```
1700000000.123    45 192.168.1.20 TCP_MISS/200 1234 GET http://10.0.0.2/ - DIRECT/10.0.0.2 text/html
│               │   │             │             │   │   │                   │
│               │   │             │             │   │   │                   └─ Action (DIRECT=serveur réel)
│               │   │             │             │   │   └─ URL demandée
│               │   │             │             │   └─ Méthode HTTP
│               │   │             │             └─ Taille de la réponse (octets)
│               │   │             └─ Résultat Cache / Code HTTP
│               │   └─ IP du client
│               └─ Durée de traitement (ms)
└─ Timestamp Unix
```

### Codes de résultat importants :

| Code         | Signification                                               |
|--------------|-------------------------------------------------------------|
| `TCP_HIT`    | Réponse servie depuis le **cache Squid** (rapide !)         |
| `TCP_MISS`   | Cache absent → requête **transmise au serveur réel**        |
| `TCP_DENIED` | Requête **refusée par une règle ACL** (blacklist, etc.)     |
| `TCP_TUNNEL` | Tunnel HTTPS (CONNECT) établi                               |
| `TCP_REFRESH_HIT` | Objet en cache **rafraîchi** (validation ETag/If-Modified) |
| `TCP_REFRESH_MISS`| Cache expiré, **nouveau téléchargement**               |

### Exemple de log pour le TP :

```log
# PC2 tente YouTube → BLOQUÉ
1700000001.000   10 192.168.1.20 TCP_DENIED/403 1234 GET http://www.youtube.com/ - NONE/- text/html

# PC1 accède YouTube → AUTORISÉ (exception)
1700000002.000   45 192.168.1.10 TCP_MISS/200 54321 GET http://www.youtube.com/ - DIRECT/142.250.x.x text/html

# PC2 accède 10.0.0.2 → MISS (première fois)
1700000003.000   50 192.168.1.20 TCP_MISS/200 2048 GET http://10.0.0.2/ - DIRECT/10.0.0.2 text/html

# PC3 accède 10.0.0.2 → HIT (déjà en cache !)
1700000004.000    2 192.168.1.30 TCP_HIT/200  2048 GET http://10.0.0.2/ - NONE/- text/html
```

---

## 10. CONCLUSION

### Récapitulatif des fonctionnalités implémentées :

| Fonctionnalité            | Statut | Détail                                            |
|---------------------------|--------|---------------------------------------------------|
| Proxy Mandataire Squid    | ✅     | Port 3128, gateway clients = 192.168.1.100        |
| Cache RAM                 | ✅     | 256 MB, politique LRU                             |
| Cache Disque              | ✅     | 1 Go, politique LFUDA, /var/spool/squid           |
| Blacklist domaines        | ✅     | YouTube, Social, Streaming, Adult, Misc           |
| Blacklist mots-clés URL   | ✅     | /porn/, /xxx/, /torrent, /crack/, etc.            |
| Exception PC1             | ✅     | ACL pc1_exempt → http_access allow (avant tout)  |
| NAT sur R1                | ✅     | iptables MASQUERADE sur eth1                      |
| IP Forwarding Proxy       | ✅     | /proc/sys/net/ipv4/ip_forward = 1                 |
| Logs détaillés            | ✅     | /var/log/squid/access.log + cache.log             |
| Serveur Web (INTERNET)    | ✅     | nginx sur port 80, page de test                   |

### Bénéfices du Proxy Cache dans un réseau d'entreprise :

1. **Économie de bande passante** : Les ressources déjà téléchargées (images, CSS, JS, vidéos)
   sont servies depuis le cache local → réduit le trafic sortant de 40 à 70%.

2. **Amélioration des performances** : La latence pour un `TCP_HIT` est de quelques ms,
   contre plusieurs dizaines de ms pour un `TCP_MISS`.

3. **Contrôle d'accès** : Le proxy est le point central de contrôle de tous les accès Web.
   Les ACL permettent un filtrage fin par IP, domaine, URL, heure, etc.

4. **Sécurité** : En masquant les IP internes (proxy agit comme intermédiaire),
   les serveurs externes ne voient jamais les adresses des postes clients.

5. **Auditabilité** : Tous les accès sont journalisés avec l'IP source, l'URL, le code HTTP,
   la taille et le résultat du cache → traçabilité complète.

---

*TP réalisé avec Kathara — Simulation réseau sous Linux*
*Configuration : Proxy Squid + NAT iptables + nginx*
