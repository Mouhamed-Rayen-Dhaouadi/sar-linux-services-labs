# TP 03 – Serveur DNS BIND 9 sous Ubuntu 22.04 (maître, esclave, failover)

**Zone gérée :** `esprit1.com` + zone inverse `1.168.192.in-addr.arpa`
**Réseau :** `192.168.1.0/24` · **Serveur maître :** `192.168.1.100` · **Esclave (alias IP) :** `192.168.1.102`

---

## 1. Théorie

### 1.1 Le DNS

Le **DNS** (*Domain Name System*) traduit des **noms** (`www.esprit1.com`) en **adresses IP** (`192.168.1.100`) et inversement. C'est une base de données **hiérarchique et distribuée** : racine `.` → TLD (`.com`) → domaine (`esprit1`) → hôtes (`www`).

- **FQDN** : nom complet, terminé par un point (`www.esprit1.com.`).
- **Zone** : portion de l'espace de noms gérée par un serveur. Fichier de zone = ses enregistrements.
- **Serveur faisant autorité** (*authoritative*) : répond pour ses zones. **Résolveur récursif** : cherche la réponse à la place du client (et la met en cache).
- **Forwarder** : serveur vers lequel on **transmet** les requêtes qu'on ne sait pas résoudre (ex. `1.1.1.1`).

### 1.2 Types d'enregistrements (RR)

| Type | Signification | Exemple |
|------|---------------|---------|
| **SOA** | *Start of Authority* : paramètres de la zone (serial, refresh, retry, expire, TTL négatif) | `@ IN SOA esprit1.com. root.esprit1.com. (...)` |
| **NS** | *Name Server* : serveur de noms de la zone | `@ IN NS bindserver.esprit1.com.` |
| **A** | nom → IPv4 | `www IN A 192.168.1.100` |
| **AAAA** | nom → IPv6 | |
| **CNAME** | *Canonical Name* : alias vers un autre nom | `ftp IN CNAME www` |
| **MX** | *Mail Exchanger* : serveur de messagerie | `@ IN MX 10 mail.esprit1.com.` |
| **PTR** | *Pointer* : IP → nom (zone inverse) | `100 IN PTR www.esprit1.com.` |

> L'énoncé écrit « CN » ; l'abréviation correcte est **CNAME**.

### 1.3 Zone directe et zone inverse

- **Directe** : nom → IP (`esprit1.com`).
- **Inverse** : IP → nom. Le réseau `192.168.1.0/24` devient `1.168.192.in-addr.arpa` (octets **inversés**) et l'hôte `192.168.1.100` s'écrit `100`.

### 1.4 Maître / esclave et AXFR

- **Maître** (*primary*) : détient la zone modifiable.
- **Esclave** (*secondary*) : **copie** la zone automatiquement par **transfert de zone** :
  - **AXFR** = transfert complet ; **IXFR** = incrémental.
  - Déclenché si le **serial** du maître est **supérieur** à celui de l'esclave → **toujours incrémenter le serial** après modification d'un fichier de zone.
  - Le maître doit autoriser l'esclave (`allow-transfer`).
- Intérêt : **redondance**. Si le maître tombe, l'esclave répond avec la copie reçue.

### 1.5 Côté client : `systemd-resolved`

Sur Ubuntu 22.04, `/etc/resolv.conf` est un **lien** généré par `systemd-resolved` : on ne l'édite pas. On configure `/etc/systemd/resolved.conf` (global) et **Netplan** (par interface, prioritaire).

---

## 2. Plan du TP

**Partie 1 : serveur maître**
1. Mises à jour · 2. Pare-feu · 3. Installation de BIND 9
4. `named.conf.options` · 5. `named.conf.local` · 6–8. Dossier et fichiers de zone
9. Vérification de syntaxe · 10. Redémarrage · 11. Tests `dig` · 12. Client

**Partie 2 : serveur esclave (même VM, alias IP)**
13. Alias IP · 14. `allow-transfer` sur le maître · 15. 2ᵉ instance BIND (esclave)

**Partie 3 : bascule (failover)**
16. Client avec 2 DNS · 17. Service systemd de l'esclave · 18–21. Test de panne

---

## 3. Pratique

### Partie 1 : serveur maître

#### Étape 1 – Mises à jour

```bash
sudo su
apt update
apt upgrade -y
```

- `sudo su` : ouvre un shell root.
- `apt update` : rafraîchit la **liste** des paquets disponibles.
- `apt upgrade -y` : installe les mises à jour (`-y` = répondre oui).

#### Étape 2 – Désactiver le pare-feu (labo)

```bash
sudo ufw disable
```

- `ufw disable` : désactive le pare-feu UFW pour que le port 53 ne soit pas bloqué. 🔒 En production, on **autorise** plutôt `53/tcp` et `53/udp`.

#### Étape 3 – Installer BIND 9

```bash
sudo apt install -y bind9 bind9utils bind9-doc dnsutils
sudo systemctl status bind9
```

- `bind9` : le serveur ; `bind9utils` : outils (`named-checkconf`, `named-checkzone`, `rndc`) ; `bind9-doc` : documentation ; `dnsutils` : `dig`, `nslookup`.
- `systemctl status bind9` : état du service (doit être *active (running)*).

#### Étape 4 – `/etc/bind/named.conf.options`

```bash
sudo gedit /etc/bind/named.conf.options
```

```
options {
        directory "/var/cache/bind";
        dnssec-validation auto;
        listen-on-v6 { any; };
        allow-query { localhost; LAN; };
        forwarders { 1.1.1.1; };
        recursion yes;
};

acl LAN {
        192.168.1.0/24;
};
```

- `acl LAN { … }` : **liste de contrôle d'accès** nommée = notre réseau local.
- `allow-query { localhost; LAN; }` : seuls ces clients peuvent interroger le serveur.
- `forwarders { 1.1.1.1; }` : les requêtes hors de nos zones sont transmises à `1.1.1.1`.
- `recursion yes` : autorise les requêtes récursives.
- `directory` : dossier de travail/cache de `named`.

```bash
sudo named-checkconf /etc/bind/named.conf.options
sudo named -V
sudo ss -lnptu | grep named
sudo systemctl restart named
sudo systemctl enable named
```

- `named-checkconf` : vérifie la **syntaxe** ; **aucun affichage = OK**.
- `named -V` : affiche version et options de compilation.
- `ss -lnptu | grep named` : sockets en écoute (`-l` listening, `-n` numérique, `-p` processus, `-t` TCP, `-u` UDP) → port **53**.
- `restart` : relance pour appliquer ; `enable` : démarrage automatique au boot.

#### Étape 5 – `/etc/bind/named.conf.local` (déclaration des zones)

```
zone "esprit1.com" IN {
        type master;
        file "/etc/bind/zones/esprit1.com";
};

zone "1.168.192.in-addr.arpa" IN {
        type master;
        file "/etc/bind/zones/esprit1.com.rev";
};
```

- `type master` : on est maître de cette zone.
- `file` : emplacement du fichier de zone.

```bash
sudo named-checkconf
```

#### Étape 6 – Dossier des zones

```bash
sudo mkdir /etc/bind/zones
```

#### Étape 7 – Zone directe

```bash
sudo cp /etc/bind/db.local /etc/bind/zones/esprit1.com
sudo gedit /etc/bind/zones/esprit1.com
```

(`db.local` sert de modèle.) Contenu :

```
$TTL    604800
@       IN      SOA     esprit1.com. root.esprit1.com. (
                        2         ; Serial
                        604800    ; Refresh
                        86400     ; Retry
                        2419200   ; Expire
                        604800 )  ; Negative Cache TTL
;
@          IN      NS      bindserver.esprit1.com.
bindserver IN      A       192.168.1.100
www        IN      A       192.168.1.100
```

- `$TTL` : durée de vie (secondes) des réponses dans les caches.
- `@` : représente le nom de la zone (`esprit1.com.`).
- SOA : `esprit1.com.` = serveur primaire ; `root.esprit1.com.` = e-mail de l'admin (`root@esprit1.com`, le `@` devient un point) ; **Serial** (incrémenter à chaque modif) ; **Refresh** (fréquence de vérification par l'esclave) ; **Retry** (nouvel essai si échec) ; **Expire** (l'esclave arrête de répondre après ce délai sans contact) ; **Negative Cache TTL** (durée de cache d'un « n'existe pas »).
- ⚠️ Un nom **sans point final** est complété avec le domaine (`www` → `www.esprit1.com.`) ; un nom **avec** point final est absolu.

#### Étape 8 – Zone inverse

```bash
sudo cp /etc/bind/db.127 /etc/bind/zones/esprit1.com.rev
sudo gedit /etc/bind/zones/esprit1.com.rev
```

```
$TTL    604800
@       IN      SOA     esprit1.com. root.esprit1.com. (
                        1         ; Serial
                        604800    ; Refresh
                        86400     ; Retry
                        2419200   ; Expire
                        604800 )  ; Negative Cache TTL
;
@       IN      NS      bindserver.esprit1.com.
100     IN      PTR     bindserver.esprit1.com.
100     IN      PTR     www.esprit1.com.
```

- `100` = **dernier octet** de `192.168.1.100`, relatif à `1.168.192.in-addr.arpa.`.
- `PTR` : associe l'IP à un nom (noter le **point final** sur les FQDN cibles).

#### Étape 9 – Vérifier les zones

```bash
sudo named-checkzone esprit1.com /etc/bind/zones/esprit1.com
sudo named-checkzone 1.168.192.in-addr.arpa /etc/bind/zones/esprit1.com.rev
```

- `named-checkzone <nom-de-zone> <fichier>` : vérifie le fichier ; attendu : `loaded serial N` puis **OK**.
- 💡 Pour la zone inverse, le nom correct est `1.168.192.in-addr.arpa` (l'énoncé utilise l'étiquette `esprit1.com.rev`, qui fonctionne aussi pour la simple vérification de syntaxe).

#### Étape 10 – Redémarrer

```bash
sudo systemctl restart named
sudo systemctl restart bind9
```

#### Étape 11 – Tester

```bash
dig a www.esprit1.com @192.168.1.100
dig -x 192.168.1.100 @192.168.1.100
```

- `dig a <nom> @<serveur>` : requête de type A auprès d'un serveur précis.
- Attendu : `status: NOERROR`, drapeau **`aa`** (*authoritative answer*), `ANSWER SECTION` avec `192.168.1.100`.
- `dig -x <IP>` : **résolution inverse** (PTR) → `bindserver.esprit1.com.` et `www.esprit1.com.`.

#### Étape 12 – Configuration du client

**a) Global : `systemd-resolved`**

```bash
sudo gedit /etc/systemd/resolved.conf
```

```
[Resolve]
DNS=192.168.1.100
Domains=esprit1.com
```

```bash
sudo systemctl restart systemd-resolved
```

- `DNS=` : serveur DNS à interroger ; `Domains=` : **domaine de recherche** (taper `www` équivaut à `www.esprit1.com`).

**b) Par interface : Netplan** (prioritaire)

```bash
sudo gedit /etc/netplan/01-network-manager-all.yaml
```

```yaml
network:
  version: 2
  renderer: NetworkManager
  ethernets:
    ens33:
      dhcp4: no
      addresses:
        - 192.168.1.100/24
        - 192.168.1.101/24
        - 192.168.1.102/24
      routes:
        - to: default
          via: 192.168.1.2
      nameservers:
        addresses:
          - 192.168.1.100
          - 8.8.8.8
        search:
          - esprit1.com
```

- `addresses` : IP principale + **alias** (`.101`, `.102`) sur la même carte.
- `routes … to: default via …` : passerelle par défaut.
- `nameservers` : serveurs DNS de cette interface + domaine de recherche.

```bash
sudo netplan apply
ip addr show ens33
resolvectl status
```

- `netplan apply` : applique la configuration.
- `ip addr show ens33` : affiche les IP de l'interface.
- `resolvectl status` : doit montrer `192.168.1.100` et `esprit1.com` dans **Global** et **Link**, avec *Current DNS Server: 192.168.1.100*.

**c) Test sans `/etc/hosts`**

```bash
sudo gedit /etc/hosts      # supprimer la ligne www.esprit1.com
```

Ouvrir le navigateur sur `http://esprit1.com` : la résolution doit passer par le DNS (et non plus par `/etc/hosts`, qui est prioritaire sur le DNS).

---

### Partie 2 : serveur esclave

> Une seule VM : l'esclave est une **seconde instance de BIND** qui n'écoute que sur l'alias `192.168.1.102`.

#### Étape 13 – Vérifier l'alias

```bash
ip a show ens33
```

`192.168.1.102/24` doit apparaître (créé dans Netplan).

#### Étape 14 – Autoriser le transfert sur le maître

```bash
sudo gedit /etc/bind/named.conf.local
```

```
zone "esprit1.com" IN {
        type master;
        file "/etc/bind/zones/esprit1.com";
        allow-transfer { 192.168.1.102; };
};
```

```bash
sudo named-checkconf
sudo systemctl restart named
```

- `allow-transfer { IP; }` : **seule** cette adresse peut demander l'AXFR (🔒 évite que n'importe qui télécharge toute la zone).

#### Étape 15 – Seconde instance (esclave)

```bash
sudo mkdir -p /etc/bind-slave
sudo cp -r /etc/bind/db.empty /etc/bind-slave/
sudo mkdir -p /var/cache/bind-slave
sudo chown bind:bind /var/cache/bind-slave
sudo gedit /etc/bind-slave/named.conf
```

- Crée une arborescence **séparée** (config + cache) pour l'esclave ; `chown bind:bind` donne l'écriture à l'utilisateur du service.

```
options {
    directory "/var/cache/bind-slave";
    pid-file "/run/named/named-slave.pid";
    listen-on { 192.168.1.102; };
    listen-on-v6 { none; };
    allow-query { any; };
    recursion no;
    transfer-source 192.168.1.102;
};

controls { };

zone "esprit1.com" IN {
    type slave;
    masters { 192.168.1.100; };
    file "/var/cache/bind-slave/esprit1.com.slave";
};
```

- `pid-file` distinct : évite le conflit avec l'instance maître.
- `listen-on { 192.168.1.102; }` : n'écoute que sur l'alias.
- `recursion no` : l'esclave ne fait que répondre pour sa zone.
- `controls { };` : **désactive le canal `rndc`** (127.0.0.1:953) partagé avec le maître ; sinon `rndc stop` sur le maître arrêterait aussi l'esclave.
- `transfer-source 192.168.1.102` : force l'IP **source** de la requête de transfert. Sans cela, le noyau utilise l'IP primaire `.100` et le maître répond **REFUSED** (il n'autorise que `.102`).
- `type slave` + `masters` : la zone est copiée depuis le maître ; `file` : copie locale.

**AppArmor** (Ubuntu) : `named` est confiné à certains chemins ; il faut autoriser les nouveaux.

```bash
sudo gedit /etc/apparmor.d/usr.sbin.named
```

Ajouter à côté des lignes `/etc/bind/**` et `/var/cache/bind/**` :

```
/etc/bind-slave/ r,
/etc/bind-slave/** rk,
/var/cache/bind-slave/ rw,
/var/cache/bind-slave/** rwk,
/run/named/** rwk,
```

```bash
sudo apparmor_parser -r /etc/apparmor.d/usr.sbin.named
sudo named-checkconf /etc/bind-slave/named.conf
sudo named -c /etc/bind-slave/named.conf -u bind
```

- `r` lecture, `w` écriture, `k` verrouillage (*lock*).
- `apparmor_parser -r` : recharge le profil.
- `named -c <conf> -u bind` : lance `named` avec **cette** configuration, sous l'utilisateur `bind`. Au premier plan, pour voir les erreurs (Ctrl+C pour arrêter).

```bash
sudo ls -l /var/cache/bind-slave/
dig www.esprit1.com @192.168.1.102
sudo journalctl | grep named
```

- Le fichier `esprit1.com.slave` doit exister (zone transférée) ; `dig` sur `.102` renvoie `192.168.1.100`.
- Si absent : lire les journaux.

---

### Partie 3 : test de bascule

#### Étape 16 – Client avec les deux serveurs

`/etc/systemd/resolved.conf` :

```
[Resolve]
DNS=192.168.1.100 192.168.1.102
Domains=esprit1.com
```

Netplan :

```yaml
nameservers:
  addresses:
    - 192.168.1.100
    - 192.168.1.102
  search:
    - esprit1.com
```

```bash
sudo systemctl restart systemd-resolved
sudo netplan apply
resolvectl status
```

#### Étape 17 – Service systemd de l'esclave

```bash
sudo gedit /etc/systemd/system/named-slave.service
```

```ini
[Unit]
Description=BIND DNS Slave Server
Documentation=man:named(8)
After=network.target

[Service]
Type=forking
ExecStart=/usr/sbin/named -u bind -c /etc/bind-slave/named.conf
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

- `After=network.target` : démarre après le réseau.
- `Type=forking` : `named` se détache en arrière-plan.
- `ExecStart` : commande de lancement avec la config de l'esclave.
- `Restart=on-failure` : relance en cas de plantage.
- `WantedBy=multi-user.target` : démarrage au boot quand `enable`.
- ⚠️ Ne **pas** copier `named.service` ni ajouter `Alias=bind9.service` : conflit avec le maître.

```bash
sudo systemctl daemon-reload
sudo systemctl enable named-slave
sudo systemctl start named-slave
sudo systemctl status named-slave --no-pager
sudo journalctl -u named-slave -n 50 --no-pager
```

- `daemon-reload` : relit les fichiers d'unités ; `enable` : au boot ; `start` : maintenant ; `journalctl -u <unité> -n 50` : 50 dernières lignes de log du service.

#### Étape 18 – Les deux serveurs répondent

```bash
dig www.esprit1.com @192.168.1.100
dig www.esprit1.com @192.168.1.102
```

Les deux doivent renvoyer `192.168.1.100`.

#### Étape 19 – Simuler la panne du maître

```bash
sudo systemctl stop named
sudo systemctl status named
```

#### Étape 20 – Résolution côté client pendant la panne

```bash
resolvectl flush-caches
nslookup www.esprit1.com
dig www.esprit1.com @192.168.1.102
```

- `flush-caches` : vide le cache local (sinon on teste le cache, pas le serveur).
- Le client essaie `.100` (indisponible), puis bascule sur `.102` après un délai : la réponse doit rester correcte. `dig @192.168.1.102` teste l'esclave directement et immédiatement.

#### Étape 21 – Restaurer le maître

```bash
sudo systemctl start named
sudo systemctl status named
dig www.esprit1.com @192.168.1.100
```

> L'esclave ne peut répondre pendant la panne que parce qu'il a **déjà reçu la zone par AXFR** avant l'arrêt du maître.

---

## 4. Dépannage

| Symptôme | Cause | Solution |
|----------|-------|----------|
| `named-checkzone` : erreur de syntaxe | point final oublié, parenthèse | relire le fichier de zone |
| Modification non visible sur l'esclave | **serial non incrémenté** | augmenter le serial, `sudo rndc reload` |
| Transfert `REFUSED` | `allow-transfer` / `transfer-source` manquants | voir étapes 14–15 |
| `permission denied` au démarrage de l'esclave | AppArmor | ajouter les chemins + `apparmor_parser -r` |
| `rndc stop` coupe aussi l'esclave | `controls { };` absent | l'ajouter dans la config esclave |
| `dig` fonctionne mais le navigateur non | entrée `/etc/hosts` ou cache | `resolvectl flush-caches`, vérifier `/etc/hosts` |
| `SERVFAIL` | fichier de zone invalide | `journalctl -u named`, `named-checkzone` |

---

## 5. Questions de révision

1. Différence entre zone directe et zone inverse ? Pourquoi les octets sont-ils inversés ?
2. Rôle de chaque champ du SOA ?
3. À quoi servent `forwarders`, `recursion`, `allow-query` et `acl` ?
4. Qu'est-ce qu'un transfert AXFR ? Quelle condition le déclenche ?
5. Pourquoi `transfer-source` et `controls { }` sont-ils nécessaires sur une même VM ?
6. Pourquoi ne modifie-t-on pas `/etc/resolv.conf` directement sous Ubuntu 22.04 ?
7. Que se passe-t-il côté client si le maître tombe ? Qu'est-ce qui garantit que l'esclave peut répondre ?
