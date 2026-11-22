# TP – Atelier firewalld : commandes `firewall-cmd` et rich rules

**Distribution :** AlmaLinux 8 (énoncé écrit pour CentOS 7, commandes identiques) · **Outil :** `firewall-cmd` · **Réseau d'exemple :** 192.168.122.0/24

> ⚠️ L'énoncé fournit des **commandes** et des **exemples de tâches** (avec solutions). Dans les PDF, les `--` sont affichés comme des tirets longs (`–permanent`) et les guillemets sont typographiques : **à retaper avec des `--` et des guillemets droits `"` / `'`**. Les remarques marquées **à vérifier** sont mon interprétation.

---

## 1. Théorie

### 1.1 Qu'est-ce que firewalld ?

**firewalld** est le pare-feu **dynamique** de la famille Red Hat. Il gère les règles par **zones** et par **services**, et permet de les modifier **sans couper les connexions existantes**. En AlmaLinux 8, il s'appuie sur **nftables**.

- Outil en ligne de commande : **`firewall-cmd`** ; interface graphique : `firewall-config`.

### 1.2 Zones

Une **zone** est un niveau de confiance appliqué à une interface ou à une source.

| Zone | Usage |
|------|-------|
| `drop` | tout est rejeté **silencieusement** |
| `block` | tout est rejeté avec un message ICMP |
| `public` | réseau non fiable (**zone par défaut**) : seuls les services autorisés passent |
| `work`, `home`, `internal` | réseaux de plus en plus fiables |
| `trusted` | tout est accepté |
| `dmz` | machines exposées |

- Sans `--zone=<zone>`, la commande agit sur la **zone par défaut** (ici `public`).

### 1.3 Runtime vs permanent

| | Runtime (par défaut) | Permanent |
|---|----------------------|-----------|
| Commande | `firewall-cmd --add-service=http` | `firewall-cmd --permanent --add-service=http` |
| Effet immédiat ? | **oui** | **non** (jusqu'au `--reload`) |
| Survit au reboot / `--reload` ? | **non** | **oui** |

- `--reload` : recharge la configuration **permanente** et **supprime** la config runtime non sauvegardée.
- Pour sauvegarder la config runtime : `firewall-cmd --runtime-to-permanent`.
- Méthode courante : `--permanent` puis `--reload`.

### 1.4 Services, ports, redirections

| Notion | Exemple | Remarque |
|--------|---------|----------|
| **Service** | `--add-service=http` | groupe de ports prédéfini (`firewall-cmd --get-services`) |
| **Port** | `--add-port=8080/tcp` | port/protocole explicite |
| **Redirection de port** | `--add-forward-port=port=80:proto=tcp:toport=8080` | redirige le trafic entrant vers un autre port |
| **Masquerade** | `--add-masquerade` | NAT : la machine fait office de **routeur** |
| **Rich rule** | `--add-rich-rule='...'` | règle **détaillée** (source, service, action…) |

### 1.5 Rich rules : syntaxe générale

```
rule [family="ipv4|ipv6"] [source address="IP[/masque]"] <élément> <action>
```

| Partie | Exemples |
|--------|----------|
| `family` | `"ipv4"` |
| `source address` | `"192.168.122.0/24"` |
| élément | `service name="http"`, `port port="22" protocol="tcp"`, `forward-port port="4032" protocol="tcp" to-port="22"` |
| action | `accept`, `reject`, `drop` |

- `reject` : refuse **et prévient** l'émetteur ; `drop` : ignore **silencieusement**.
- ⚠️ Les rich rules **de refus** sont évaluées **avant** les règles d'acceptation.

---

## 2. Plan du TP

**Partie A – Prise en main de `firewall-cmd`**
1. État et rechargement
2. Informations (zones, interfaces, services)
3. Ajouter services, ports, redirection
4. Mode panique
5. Option `--timeout`
6. Retour à la configuration par défaut

**Partie B – Exemples de tâches (rich rules)**
1. Site web accessible seulement depuis 192.168.122.0/24
2. Redirection de port pour une seule source
3. Samba accessible seulement depuis 192.168.122.0/24
4. Bloquer le port 80 pour une seule adresse
5. Serveur Linux en routeur

---

## 3. Pratique

### Partie A : Prise en main de `firewall-cmd`

#### A.1 État et rechargement

```bash
firewall-cmd --state
firewall-cmd --reload
```

- `--state` : renvoie `running` si firewalld fonctionne.
- `--reload` : recharge les fichiers de configuration.

#### A.2 Obtenir des informations

```bash
firewall-cmd --get-active-zones
firewall-cmd --zone=public --list-interfaces
firewall-cmd --zone=public --list-services
firewall-cmd --zone=public --list-all
```

| Commande | Rôle |
|----------|------|
| `--get-active-zones` | zones **actives** et leurs interfaces |
| `--zone=public --list-interfaces` | interfaces de la zone |
| `--zone=public --list-services` | services autorisés |
| `--zone=public --list-all` | **tout** : services, ports, forward-ports, rich rules… |

Commandes utiles en plus : `--get-default-zone`, `--get-services`, `--get-zones`.

#### A.3 Ajouter services, ports et redirection

```bash
# Services http et https
firewall-cmd --add-service=http --add-service=https
firewall-cmd --list-services

# Port 8080
firewall-cmd --add-port=8080/tcp

# Redirection de port 80 → 8080
firewall-cmd --add-forward-port=port=80:proto=tcp:toport=8080
```

- Sans `--zone`, la **zone par défaut** (`public`) est utilisée.
- Sans `--permanent`, ces règles sont **runtime** (perdues au reload/reboot).
- Syntaxe de redirection : `port=<port source>:proto=<tcp|udp>:toport=<port destination>` (ajouter `:toaddr=<IP>` pour rediriger vers une autre machine).

#### A.4 Mode panique

```bash
firewall-cmd --query-panic
firewall-cmd --panic-on
firewall-cmd --panic-off
```

Le **mode panique coupe tout le trafic réseau** (y compris les connexions existantes).

**Expérience avec 2 terminaux :**

1. Terminal 1 : `ping localhost`
2. Terminal 2 : `firewall-cmd --query-panic` (→ `no`)
3. Terminal 2 : `firewall-cmd --panic-on`
4. Observer le ping du terminal 1 : **il se bloque / échoue** (aucun paquet ne passe).
5. Terminal 2 : `firewall-cmd --panic-off` : le ping **reprend**.

- ⚠️ En SSH distant, `--panic-on` **coupe votre propre session**.

#### A.5 Option `--timeout`

```bash
# Terminal 1
watch firewall-cmd --zone=public --list-services

# Terminal 2
firewall-cmd --add-service=mysql --timeout=10s
```

- `--timeout=<durée><s|m|h>` : la règle est **supprimée automatiquement** après la durée indiquée.
- Observation : `mysql` apparaît dans le terminal 1, puis **disparaît après 10 s**.
- Très utile pour **tester sans risque** une règle (on ne se bloque pas définitivement).
- `--timeout` s'applique aux règles **runtime** (pas compatible avec `--permanent`).

#### A.6 Retour à la configuration par défaut

```bash
firewall-cmd --reload
```

- Recharge la config **permanente** : supprime tout ce qui n'avait été ajouté qu'en runtime.

---

### Partie B : Exemples de tâches avec *rich rules*

#### B.1 Site web accessible uniquement depuis 192.168.122.0/24

```bash
firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="192.168.122.0/24" service name="http" accept'
firewall-cmd --reload
```

- La règle **autorise** `http` **seulement** pour ce sous-réseau.
- ⚠️ Pour que les **autres** sous-réseaux soient réellement bloqués, `http` **ne doit pas** être autorisé globalement dans la zone (`firewall-cmd --permanent --remove-service=http`), sinon il reste ouvert à tous.
- Alternative citée par l'énoncé : `Require ip 192.168.122.0/24` côté **Apache** (valable pour un seul hôte virtuel). Le pare-feu protège **tout le serveur** au niveau du port 80.

#### B.2 Redirection de port 4030 → 22 seulement pour une source

L'énoncé parle du port **4030**, mais la commande utilise **4032** : **choisir un seul port** et l'utiliser partout.

```bash
firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="192.168.122.1" forward-port port="4032" protocol="tcp" to-port="22"'
firewall-cmd --reload
```

- Seule la machine **192.168.122.1** voit le port 4032 redirigé vers **SSH (22)**.
- Test depuis cette machine : `ssh -p 4032 user@<IP_serveur>`.

#### B.3 Samba accessible uniquement depuis 192.168.122.0/24

```bash
firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="192.168.122.0/24" service name="samba" accept'
firewall-cmd --reload
```

- L'énoncé écrit `service name="smb"` : sous firewalld, le service s'appelle **`samba`** (*à vérifier avec `firewall-cmd --get-services`*).
- Comme en B.1, retirer le service global `samba` de la zone si on veut bloquer les autres réseaux.
- (Voir le TP Samba pour la configuration du partage.)

#### B.4 Bloquer le port 80 pour 192.168.122.1 uniquement

```bash
firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="192.168.122.1" service name="http" reject'
firewall-cmd --reload
```

- Les autres adresses continuent d'accéder à `http` (à condition que le service soit autorisé dans la zone, par ex. `--add-service=http`).
- `reject` renvoie un refus explicite ; `drop` ignorerait silencieusement.

#### B.5 Serveur Linux en routeur

```bash
firewall-cmd --permanent --add-masquerade
firewall-cmd --reload
```

Activer aussi le **routage IP dans le noyau** :

```bash
echo "net.ipv4.ip_forward = 1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

- `--add-masquerade` : active le **NAT** (les paquets sortants prennent l'IP de l'interface de sortie).
- `net.ipv4.ip_forward = 1` : le noyau **retransmet** les paquets entre interfaces (sans cela, pas de routage).
- Vérifier : `firewall-cmd --query-masquerade` et `sysctl net.ipv4.ip_forward`.
- (L'énoncé ne met pas `--permanent` ; sans lui, la règle est perdue au reload.)

---

## 4. Dépannage

| Symptôme | Cause probable | Solution |
|----------|----------------|----------|
| `Error: INVALID_...` / commande non reconnue | `–permanent` copié depuis le PDF (tiret long) / guillemets typographiques | retaper `--permanent` et des guillemets droits |
| Règle ajoutée mais absente après `--reload` | ajoutée **sans** `--permanent` | refaire avec `--permanent`, ou `--runtime-to-permanent` |
| Règle ajoutée en `--permanent` mais inactive | pas de `--reload` | `firewall-cmd --reload` |
| Les autres réseaux accèdent encore au service | le service reste autorisé globalement dans la zone | `--remove-service=<service>` |
| Rich rule rejetée | erreur de syntaxe (guillemets, espaces) | simples quotes autour de la règle, doubles autour des valeurs |
| Plus de réseau du tout | `--panic-on` activé | `firewall-cmd --panic-off` (depuis la console) |
| Le routage ne marche pas | `ip_forward` à 0 | `sysctl -w net.ipv4.ip_forward=1` |
| Voir la config complète | | `firewall-cmd --list-all`, `--list-rich-rules` |
| Supprimer une rich rule | | `firewall-cmd --permanent --remove-rich-rule='...'` (même texte que l'ajout) |

---

## 5. Questions de révision

1. Qu'est-ce que firewalld ? Qu'est-ce qu'une **zone** ? Quelle est la zone par défaut ?
2. Différence entre configuration **runtime** et **permanente** ? Rôle de `--reload` ?
3. Que font `--add-service`, `--add-port` et `--add-forward-port` ?
4. Que fait le **mode panique** ? Comment le tester ?
5. À quoi sert l'option `--timeout` ?
6. Qu'est-ce qu'une **rich rule** ? Donner la syntaxe générale.
7. Comment autoriser `http` uniquement pour 192.168.122.0/24 ?
8. Comment rediriger le port 4032 vers 22 uniquement pour 192.168.122.1 ?
9. Différence entre `reject` et `drop` ?
10. Comment bloquer le port 80 pour une seule adresse ?
11. Comment transformer un serveur Linux en routeur (deux réglages à faire) ?
12. Pourquoi `--permanent` seul ne suffit-il pas à appliquer une règle immédiatement ?
