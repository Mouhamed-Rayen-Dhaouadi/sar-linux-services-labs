# TP – Atelier collectd : supervision des ressources système (AlmaLinux)

**Distribution :** AlmaLinux 8 (`dnf`) · **Dépôt :** EPEL · **Paquets :** `collectd`, `collectd-rrdtool`, `rrdtool`, `httpd`, `stress-ng`, `iperf3` · **Hostname d'exemple :** `almalinux-node`

> ⚠️ Cette fiche suit l'énoncé de l'atelier. Les remarques marquées **à vérifier** sont mon interprétation : à confronter avec ton cours et à tester sur ta VM.

---

## 1. Théorie

### 1.1 Qu'est-ce que collectd ?

**collectd** est un **démon de collecte de métriques** système (CPU, mémoire, disque, réseau…). Il est très léger, écrit en C, et fonctionne **entièrement par plugins**.

- Il **collecte** à intervalle régulier (**10 s** par défaut) puis **écrit** les données via un plugin de sortie (ici `rrdtool`).
- Il ne dessine **pas** de graphes lui-même : il faut une interface (ici **collection3**).

### 1.2 Architecture

```
Plugins de lecture            Plugin d'écriture          Visualisation
(cpu, memory, interface,  →   rrdtool                →   collection3 (CGI Perl via Apache)
 load, processes, disk)       (fichiers .rrd)
```

### 1.3 Les plugins du TP

| Plugin | Métriques collectées |
|--------|----------------------|
| `cpu` | utilisation CPU (user, system, idle, wait…) |
| `memory` | mémoire utilisée/libre/cache/buffers |
| `interface` | trafic réseau par interface (octets, paquets, erreurs) |
| `load` | charge moyenne (1, 5, 15 min) |
| `processes` | nombre de processus par état (running, sleeping, zombie…) |
| `disk` | opérations et débit des disques |
| `rrdtool` | **écrit** les données dans des fichiers RRD |

### 1.4 RRDtool (Round Robin Database)

- Base de données **circulaire de taille fixe** : les anciennes données sont **écrasées** et **agrégées** (moyennes) au fil du temps.
- Avantage : la taille des fichiers `.rrd` **ne grossit jamais** ; adapté aux séries temporelles et aux graphes.
- Stockage : `DataDir` → `/var/lib/collectd/rrd/<hostname>/<plugin>/…`.

### 1.5 Outils de test de charge

| Outil | Rôle |
|-------|------|
| `stress-ng` | génère de la charge CPU, mémoire, disque… |
| `iperf3` | génère du trafic réseau (mode serveur `-s` / client `-c`) |

---

## 2. Plan du TP

1. Préparation (mise à jour, EPEL)
2. Installation de collectd et des plugins
3. Configuration de `/etc/collectd.conf`
4. Plugin `rrdtool` et répertoire RRD
5. Démarrage du service
6. Interface web (collection3 + Apache)
7. Tests de charge (CPU, RAM, disque, réseau)
8. Diagnostic des erreurs

---

## 3. Pratique

### Étape 1 : Préparation

```bash
sudo dnf update -y
sudo dnf install -y epel-release
```

- collectd n'est pas dans les dépôts de base : il vient d'**EPEL** (*Extra Packages for Enterprise Linux*).

### Étape 2 : Installation

```bash
sudo dnf install -y collectd collectd-plugins rrdtool collectd-rrdtool
ls /usr/lib64/collectd/
```

- `ls /usr/lib64/collectd/` : liste les **plugins installés** (fichiers `.so`).
- *À vérifier* : sur EL8, plusieurs plugins sont dans des **sous-paquets séparés** (ex. `collectd-disk`, `collectd-rrdtool`). Si `collectd-plugins` n'existe pas ou si un plugin manque : `dnf search collectd`, puis installer le sous-paquet (`sudo dnf install -y collectd-disk`).

### Étape 3 : Configuration

```bash
sudo vi /etc/collectd.conf
```

```
Hostname "almalinux-node"
LoadPlugin cpu
LoadPlugin memory
LoadPlugin interface
LoadPlugin load
LoadPlugin processes
LoadPlugin disk
```

- `Hostname` : nom sous lequel les données sont enregistrées (nom du dossier RRD).
- `LoadPlugin <nom>` : charge un plugin.
- Autre paramètre utile : `Interval 10` (secondes entre deux collectes).

### Étape 4 : Plugin RRDTool

```
LoadPlugin rrdtool
<Plugin rrdtool>
  DataDir "/var/lib/collectd/rrd"
</Plugin>
```

- `DataDir` : dossier de stockage des fichiers `.rrd`.
- ⚠️ Les directives `LoadPlugin` doivent être **décommentées** (sans `#`) ; si une ligne `LoadPlugin` existe déjà dans le fichier, éviter de la **dupliquer**.

### Étape 5 : Répertoire RRD

```bash
sudo mkdir -p /var/lib/collectd/rrd
sudo chown -R root:root /var/lib/collectd
sudo chmod -R 755 /var/lib/collectd
```

- `755` : le propriétaire écrit, les autres **lisent** (utile pour qu'Apache lise les RRD).
- collectd tourne en `root` par défaut, d'où `root:root`.

### Étape 6 : Démarrer collectd

```bash
sudo collectd -t
sudo systemctl enable --now collectd
```

- `collectd -t` : **teste la configuration** (syntaxe, plugins) **sans démarrer** le démon. Silence = OK ; sinon il affiche l'erreur.
- `enable --now` : démarre **maintenant** et **au boot**.
- Vérifier : `systemctl status collectd` et `ls /var/lib/collectd/rrd/almalinux-node/` (des dossiers par plugin doivent apparaître au bout de quelques secondes).

### Étape 7 : Interface web

```bash
sudo dnf install -y httpd collectd-web
sudo rsync -av /usr/share/collectd/collection3/ /var/www/html/collectd
sudo systemctl enable --now httpd
```

URL : `http://IP/collectd`

- **collection3** est un front-end **CGI Perl** qui lit les fichiers RRD et dessine les graphes.
- `rsync -av` : copie récursive (`-a` archive, `-v` verbeux) ; le slash final de la source copie **le contenu** du dossier.
- *À vérifier* : `collectd-web` n'est pas toujours disponible sur EL8 ; collection3 nécessite l'exécution de **CGI** par Apache (`ScriptAlias`/`Options +ExecCGI`) et des modules **Perl** (`perl-RRD`, `perl-CGI`…).
- Si la page est blanche ou le script s'affiche en texte brut : l'exécution CGI n'est pas activée.
- Si l'accès distant échoue : ouvrir le **pare-feu** (`firewall-cmd --permanent --add-service=http && firewall-cmd --reload`).
- Si Apache ne peut pas lire `/var/lib/collectd` : voir **SELinux** (`ausearch -m avc`, `restorecon`, booléens).

### Étape 8 : Tests de charge

```bash
sudo dnf install -y stress-ng
```

**8.1 Stress CPU**

```bash
stress-ng --cpu 4 --timeout 30s
```

- Lance 4 workers CPU pendant 30 s → pic sur les graphes **CPU** et **load**.

**8.2 Stress RAM**

```bash
stress-ng --vm 2 --vm-bytes 70% --timeout 30s
```

- 2 workers occupant **70 % de la mémoire** → pic sur le graphe **memory**.

**8.3 Stress disque**

```bash
stress-ng --hdd 1 --timeout 30s
```

- 1 worker d'écriture disque → pic sur le graphe **disk**.

**8.4 Stress réseau**

```bash
# Terminal 1
iperf3 -s
# Terminal 2
iperf3 -c 127.0.0.1 -t 20
```

- `iperf3 -s` : mode **serveur** ; `iperf3 -c <IP> -t 20` : mode **client**, durée 20 s.
- ⚠️ `iperf3` n'est pas installé par l'énoncé : `sudo dnf install -y iperf3`.
- Avec `127.0.0.1`, le trafic passe par l'interface **loopback `lo`** : regarder le graphe de `lo` (pas `eth0`).

**Observer les pics** dans l'interface collectd (le graphe se rafraîchit selon l'intervalle de collecte).

---

## 4. Dépannage (diagnostic des erreurs)

| Symptôme | Cause probable | Solution |
|----------|----------------|----------|
| `collectd -t` affiche une erreur | syntaxe / plugin inexistant | corriger `/etc/collectd.conf` |
| `Could not find plugin xxx` | plugin non installé | installer le sous-paquet (`dnf search collectd`) |
| Service `failed` | erreur de config, droits | `systemctl status collectd`, `journalctl -u collectd` |
| Pas de fichiers `.rrd` | `rrdtool` non chargé, `DataDir` incorrect | vérifier `LoadPlugin rrdtool`, droits du dossier |
| Données dans le mauvais dossier | `Hostname` différent | lister `/var/lib/collectd/rrd/` |
| Page web introuvable / 403 | Apache arrêté, contexte SELinux, pare-feu | `systemctl status httpd`, `firewall-cmd`, `ls -Z` |
| Page affiche le code Perl | CGI non activé | configurer `ExecCGI`/`ScriptAlias` |
| Graphes vides | collectd vient de démarrer | attendre quelques cycles (10 s chacun) |
| Pas de pic réseau sur `eth0` | test sur loopback | observer l'interface `lo` |
| Outils de diagnostic | | `rrdtool info fichier.rrd`, `rrdtool lastupdate fichier.rrd`, `journalctl -u collectd`, `/var/log/messages` |

---

## 5. Questions de révision

1. Qu'est-ce que collectd ? Pourquoi dit-on qu'il fonctionne « par plugins » ?
2. À quoi servent les plugins `cpu`, `memory`, `interface`, `load`, `processes`, `disk` ?
3. Qu'est-ce que RRDtool ? Pourquoi la taille des fichiers reste-t-elle constante ?
4. Rôle du plugin `rrdtool` et de la directive `DataDir` ?
5. À quoi sert `collectd -t` ?
6. Quel est l'intervalle de collecte par défaut ? Comment le changer ?
7. Rôle de **collection3** et d'**Apache** dans la visualisation ?
8. Comment générer une charge CPU, RAM, disque et réseau ?
9. Pourquoi observer l'interface `lo` avec `iperf3 -c 127.0.0.1` ?
10. Comment diagnostiquer une erreur de collectd (outils et fichiers de logs) ?
11. Pourquoi EPEL est-il nécessaire ?
