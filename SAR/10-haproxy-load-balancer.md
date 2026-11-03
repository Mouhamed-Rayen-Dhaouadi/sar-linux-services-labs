# TP 10 – Répartiteur de charge HAProxy (Round Robin, Round Robin pondéré, sécurité)

**Rôles :** HAProxy `192.168.1.100` · webserver01 `192.168.1.101` · webserver02 `192.168.1.102` (même LAN `192.168.1.0/24`)
**Le serveur HAProxy sert aussi de client de test.**

---

## 1. Théorie

### 1.1 Répartition de charge et reverse proxy

- Un **répartiteur de charge** (*load balancer*) reçoit les requêtes des clients et les **distribue** entre plusieurs serveurs **backend** : aucun serveur n'est surchargé, la **disponibilité** et les **performances** augmentent.
- Un **reverse proxy** se place **devant** les serveurs : le client ne parle qu'au proxy, qui relaie vers le bon serveur et renvoie la réponse.
- **HAProxy** (*High Availability Proxy*) fait les deux ; il fonctionne en couche **4** (TCP) ou **7** (HTTP).
- Un **health check** (`check`) teste régulièrement chaque backend : un serveur en panne est **retiré automatiquement** de la rotation, puis réintégré à son retour.
- ⚠️ Le répartiteur est lui-même un **point unique de défaillance** : en production, on le double (ex. avec **keepalived** / IP virtuelle).

### 1.2 Algorithmes de répartition

| Algorithme | Principe | Cas d'usage |
|------------|----------|-------------|
| `roundrobin` | chaque serveur à tour de rôle | serveurs identiques, requêtes courtes |
| `roundrobin` + `weight` (**WRR**) | tour de rôle **proportionnel aux poids** | serveurs de **capacités différentes** |
| `leastconn` | serveur avec le moins de connexions actives | sessions longues |
| `source` | hachage de l'IP client (affinité) | garder un client sur le même serveur |
| `uri` | hachage de l'URL | caches |

Exemple WRR : poids 3 et 1 → sur 4 requêtes, 3 vont à web1 et 1 à web2 (ratio **3:1**).

### 1.3 Structure de `haproxy.cfg`

| Section | Rôle |
|---------|------|
| `global` | paramètres système (processus, journalisation, sécurité) |
| `defaults` | valeurs par défaut (mode, timeouts…) héritées par les sections suivantes |
| `frontend` | **écoute** des clients (adresse/port) et règles d'aiguillage |
| `backend` | **groupe de serveurs** + algorithme |
| `listen` | frontend + backend en un seul bloc (ici : page de statistiques) |

### 1.4 Notions de sécurité vues dans le TP

- **ACL** : condition nommée (`acl allowed_ips src 192.168.1.0/24`) réutilisable dans des règles.
- **Limitation de débit** (*rate limiting*) : une **stick-table** mémorise des compteurs par IP cliente ; au-delà d'un seuil, on répond **HTTP 429 (Too Many Requests)**. Protège contre force brute et DoS simples.
- **Masquer la version** : `stats hide-version`.

---

## 2. Plan du TP

1. Installer HAProxy
2. Configurer le **Round Robin** (frontend, backend, page de statistiques)
3. Tester la répartition
4. Passer au **Round Robin pondéré** et mesurer le ratio
5. **Sécuriser** : ACL sur la page de stats, limitation de débit

---

## 3. Pratique

### Prérequis

- Deux serveurs web (voir TP Apache) dont chaque page **s'identifie** (`Esprit1.com` sur web1, `Esprit2.com` sur web2).
- Si tu n'as pas ces pages, crée un marqueur rapide **sur chaque backend** :

```bash
# sur webserver01
echo "<h1>Esprit1.com - webserver01</h1>" | sudo tee /var/www/html/index.html
# sur webserver02
echo "<h1>Esprit2.com - webserver02</h1>" | sudo tee /var/www/html/index.html
```

⚠️ **Piège si tout tourne sur une seule VM (IP en alias)** : le TP Apache a ajouté `Listen 8080` (site `esprit4`). Cela occupe déjà le port 8080 → HAProxy ne pourra pas se lier à `192.168.1.100:8080`. Désactive `esprit4` / retire `Listen 8080` avant de continuer.

### Étape 1 – Installation

```bash
sudo apt install haproxy -y
sudo systemctl status haproxy
```

- `apt install haproxy -y` : installe (`-y` = oui à tout) ; le service démarre seul.
- `systemctl status haproxy` : doit indiquer *active (running)*.

### Étape 2 – Round Robin

```bash
sudo cp /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy.cfg.bk
sudo gedit /etc/haproxy/haproxy.cfg
```

- `cp … .bk` : **sauvegarde** avant modification (réflexe indispensable : on peut restaurer en cas d'erreur).
- Garder les sections `global` et `defaults` d'origine ; **ajouter à la fin** :

```
frontend linuxtechi
    bind 192.168.1.100:8080
    stats uri /haproxy?stats
    default_backend web-servers
    option forwardfor

backend web-servers
    balance roundrobin
    server web1 192.168.1.101:80 check
    server web2 192.168.1.102:80 check

listen stats
    bind *:8404
    stats enable
    stats uri /
    stats refresh 5s
    stats realm Haproxy\ Statistics
    stats auth linuxtechi:Pass123
```

Explication ligne par ligne :

- `frontend linuxtechi` : nom du frontend (libre).
- `bind 192.168.1.100:8080` : écoute sur cette IP et ce port (8080 au lieu de 80, comme demandé).
- `default_backend web-servers` : toutes les requêtes vont au backend `web-servers`.
- `option forwardfor` : ajoute l'en-tête **`X-Forwarded-For: <IP du client>`**, car sinon les backends ne verraient que l'IP du répartiteur.
- `backend web-servers` / `balance roundrobin` : groupe de serveurs, rotation à tour de rôle.
- `server web1 192.168.1.101:80 check` : déclare un serveur (nom, IP:port) ; `check` active le **health check**.
- `listen stats` : page de statistiques sur **son propre port 8404**, distinct du trafic (pas de conflit entre les deux `bind`).
- `bind *:8404` : écoute sur toutes les adresses ; `stats enable` : active la page ; `stats uri /` : accessible à la racine ; `stats refresh 5s` : rafraîchit toutes les 5 s ; `stats realm …` : titre de la boîte d'authentification ; `stats auth user:mdp` : identifiants (`linuxtechi` / `Pass123`).

**Valider puis redémarrer**

```bash
sudo haproxy -c -f /etc/haproxy/haproxy.cfg
sudo systemctl restart haproxy
```

- `haproxy -c -f <fichier>` : **vérifie la syntaxe** (`Configuration file is valid`). En cas d'échec (`control process exited with error code`), cette commande donne la ligne fautive ; voir aussi `journalctl -xeu haproxy.service`.

### Étape 3 – Tester le Round Robin

```bash
curl http://192.168.1.100:8080
while true; do curl http://192.168.1.100:8080; sleep 1; done
```

- `curl` : requête HTTP en ligne de commande (`sudo apt install curl` si absent).
- La boucle `while true; do … ; sleep 1; done` envoie une requête par seconde (Ctrl+C pour arrêter). Les réponses doivent **alterner** : web1, web2, web1, web2…
- Statistiques en direct : **`http://192.168.1.100:8404/`** (login `linuxtechi` / `Pass123`). Les deux serveurs sont **verts** (UP) avec des compteurs *Total* équivalents.

### Étape 4 – Round Robin pondéré (WRR)

Dans `/etc/haproxy/haproxy.cfg`, modifier le backend :

```
backend web-servers
    balance roundrobin
    server web1 192.168.1.101:80 check weight 3
    server web2 192.168.1.102:80 check weight 1
```

- Aucun nouveau mot-clé `balance` : `roundrobin` gère déjà les **poids** ; on ajoute seulement `weight N` sur chaque `server`.
- Poids par défaut = **1** (donc sans `weight`, on retrouve le Round Robin classique).

```bash
sudo haproxy -c -f /etc/haproxy/haproxy.cfg
sudo systemctl reload haproxy
```

- `reload` : applique la configuration **sans couper** les connexions en cours.

**Mesurer le ratio**

```bash
for i in $(seq 1 40); do curl -s http://192.168.1.100:8080; done | grep -c Esprit1
for i in $(seq 1 40); do curl -s http://192.168.1.100:8080; done | grep -c Esprit2
```

- `seq 1 40` : génère 1…40 ; `curl -s` : mode silencieux ; `grep -c motif` : **compte les lignes** contenant le marqueur.
- Résultat attendu : ≈ **30** réponses `Esprit1` et **10** `Esprit2` (ratio 3:1), contre 20/20 en Round Robin classique.

Version plus propre (un seul lot de 40 requêtes, comptage des deux) :

```bash
for i in $(seq 1 40); do curl -s http://192.168.1.100:8080 | grep -o "Esprit[12]"; done | sort | uniq -c
```

- `grep -o` n'affiche que le mot trouvé ; `sort | uniq -c` compte les occurrences de chaque valeur.

⚠️ Fais ce test **avant** d'activer la limitation de débit (étape 5.2) : 40 requêtes rapides déclencheraient des réponses **429**.

Sur la page de statistiques, la colonne **Total** (sessions) de web1 doit croître environ **3 fois plus vite** que celle de web2.

**Bonus : tester la tolérance aux pannes**

```bash
# sur webserver01
sudo systemctl stop apache2
```

- Dans les statistiques, web1 passe en **rouge (DOWN)** après quelques secondes ; tout le trafic va à web2. Relancer : `sudo systemctl start apache2` → web1 est réintégré.

### Étape 5 – Sécurisation

#### 5.1 Restreindre la page de statistiques

```
listen stats
    bind 192.168.1.100:8404
    acl allowed_ips src 192.168.1.0/24
    tcp-request connection reject if !allowed_ips
    stats enable
    stats uri /
    stats refresh 5s
    stats realm Haproxy\ Statistics
    stats auth linuxtechi:Pass123
    stats hide-version
```

- `bind 192.168.1.100:8404` : n'écoute plus que sur l'IP du LAN (et non `*`).
- `acl allowed_ips src 192.168.1.0/24` : définit l'ACL « IP source dans le LAN ».
- `tcp-request connection reject if !allowed_ips` : **rejette** toute connexion dont la source n'est pas dans l'ACL (`!` = négation).
- `stats hide-version` : n'affiche plus la version de HAProxy.
- ⚠️ Le **nom d'ACL doit être identique** partout (`allowed_ips` partout, jamais `allowed ips` avec espace) sinon HAProxy refuse de démarrer.

#### 5.2 Limitation de débit

Dans le frontend :

```
frontend linuxtechi
    bind 192.168.1.100:8080
    # limitation : 10 requêtes par IP sur une fenêtre de 10 s
    stick-table type ip size 100k expire 30s store http_req_rate(10s)
    http-request track-sc0 src
    http-request deny deny_status 429 if { sc_http_req_rate(0) gt 10 }
    default_backend web-servers
    option forwardfor
```

- `stick-table type ip size 100k expire 30s store http_req_rate(10s)` : table en mémoire indexée par **IP** (100 000 entrées max, entrées supprimées après 30 s d'inactivité) qui stocke le **taux de requêtes sur 10 s**.
- `http-request track-sc0 src` : enregistre chaque requête dans la table, sous l'IP **source** du client.
- `http-request deny deny_status 429 if { sc_http_req_rate(0) gt 10 }` : refuse avec **HTTP 429** si le taux dépasse **10** (`gt` = *greater than*).
- 💡 Précisément : la fenêtre est de **10 secondes**, donc « plus de 10 requêtes en 10 s par IP » (et non strictement 10 par seconde).
- 💡 La directive `stats uri /haproxy?stats` du frontend (étape 2) expose une page de stats **sans authentification** sur le port 8080 : une fois la page dédiée en place sur 8404, il vaut mieux **la supprimer** du frontend.

```bash
sudo haproxy -c -f /etc/haproxy/haproxy.cfg
sudo systemctl restart haproxy
for i in {1..20}; do curl -s -o /dev/null -w "Requête $i : %{http_code}\n" http://192.168.1.100:8080; done
```

- `curl -s -o /dev/null -w "…%{http_code}…"` : ignore le contenu et n'affiche que le **code HTTP**.
- Attendu : requêtes 1 à 10 → `200`, requêtes 11 à 20 → **`429`**.
- (La capture de l'énoncé utilise `https://192.168.1.100` : le frontend du TP est en **HTTP sur 8080**, utilise l'URL ci-dessus.)

#### Configuration finale de référence

```
frontend linuxtechi
    bind 192.168.1.100:8080
    stick-table type ip size 100k expire 30s store http_req_rate(10s)
    http-request track-sc0 src
    http-request deny deny_status 429 if { sc_http_req_rate(0) gt 10 }
    default_backend web-servers
    option forwardfor

backend web-servers
    balance roundrobin
    server web1 192.168.1.101:80 check weight 3
    server web2 192.168.1.102:80 check weight 1

listen stats
    bind 192.168.1.100:8404
    acl allowed_ips src 192.168.1.0/24
    tcp-request connection reject if !allowed_ips
    stats enable
    stats uri /
    stats refresh 5s
    stats realm Haproxy\ Statistics
    stats auth linuxtechi:Pass123
    stats hide-version
```

🔒 `stats auth` transmet les identifiants **en clair** (HTTP) : à protéger par TLS et à ne pas laisser par défaut en production.

---

## 4. Dépannage

| Symptôme | Cause | Solution |
|----------|-------|----------|
| `control process exited with error code` | erreur de configuration | `haproxy -c -f …`, `journalctl -xeu haproxy.service` |
| `Cannot bind socket` | port déjà utilisé (ex. Apache sur 8080) | `sudo ss -lntp \| grep 8080`, libérer le port |
| `unable to find required ACL` | nom d'ACL différent (espace / tiret / underscore) | harmoniser le nom |
| Un serveur reste **DOWN** | backend injoignable ou arrêté | `curl http://192.168.1.101`, pare-feu, `systemctl status apache2` |
| Page de stats : 401 | mauvais identifiants | vérifier `stats auth` |
| Page de stats : connexion refusée | ACL / mauvais `bind` | tester depuis le LAN, vérifier `ss -lntp` |
| Réponses 429 pendant le test WRR | limitation de débit active | tester avant l'étape 5.2 ou espacer les requêtes |
| Pare-feu | | `sudo ufw allow 8080/tcp && sudo ufw allow 8404/tcp` |

---

## 5. Questions de révision

1. Qu'est-ce qu'un répartiteur de charge ? Quel problème résout-il ?
2. Différence entre Round Robin et Round Robin pondéré ? Quand utiliser chacun ?
3. Rôles des sections `global`, `defaults`, `frontend`, `backend`, `listen` ?
4. À quoi servent `check` et `option forwardfor` ?
5. Pourquoi la page de statistiques est-elle sur un port séparé et restreinte par une ACL ?
6. Comment fonctionne la limitation de débit (stick-table, `track-sc0`, code 429) ?
7. Quelle commande valide la syntaxe avant de recharger ? Différence entre `reload` et `restart` ?
8. Quel est le point faible d'un seul HAProxy, et comment y remédier ?
