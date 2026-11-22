# TP – Atelier sysstat et `sar` : surveiller le système

**Distributions :** AlmaLinux 8 / RHEL / CentOS (`yum`/`dnf`) et Debian/Ubuntu (`apt`) · **Paquet :** `sysstat` · **Outils :** `sar`, `sa1`, `sa2`, `iostat`, `mpstat`

> ⚠️ Cette fiche suit l'énoncé de l'atelier. Les remarques marquées **à vérifier** sont mon interprétation : à confronter avec ton cours et à tester sur ta VM.

---

## 1. Théorie

### 1.1 Qu'est-ce que sysstat ?

**sysstat** est une suite d'outils de **supervision des performances** : elle **collecte** régulièrement l'activité du système et la **stocke** pour pouvoir l'analyser **a posteriori** (historique).

| Outil | Rôle |
|-------|------|
| **`sar`** (*System Activity Reporter*) | **affiche** les statistiques (en direct ou depuis l'historique) |
| `sadc` | collecteur de bas niveau (*System Activity Data Collector*) |
| `sa1` | script qui lance `sadc` à intervalle régulier (**collecte**) |
| `sa2` | script qui produit le **rapport journalier** |
| `iostat` | statistiques CPU et I/O disque |
| `mpstat` | statistiques **par cœur** CPU |
| `pidstat` | statistiques **par processus** |

### 1.2 Fonctionnement

```
cron / timer systemd  →  sa1 (sadc)  →  /var/log/sa/saXX   (binaire, 1 fichier par jour)
                                       └→ sar -f /var/log/sa/saXX   (lecture)
```

- `XX` = **jour du mois** (ex. `sa15` = le 15). Les fichiers sont **binaires** et lus avec `sar -f`.
- Sans sysstat, `sar` ne peut afficher que des mesures **en direct** ; avec la collecte, on peut **revenir en arrière** (diagnostic d'un ralentissement passé).

### 1.3 Options de `sar` (de l'énoncé)

| Option | Affiche | Colonnes importantes |
|--------|---------|----------------------|
| `sar -u` | **CPU** | `%user`, `%system`, `%iowait`, `%idle` |
| `sar -r` | **Mémoire** | `kbmemfree`, `kbmemused`, `%memused`, `kbcached` |
| `sar -W` | **Swap** (pages échangées) | `pswpin/s`, `pswpout/s` |
| `sar -b` | **I/O disque global** | `tps`, `bread/s`, `bwrtn/s` |
| `sar -d` | **I/O par disque** | `tps`, `await`, `%util` |
| `sar -A` | **tout** | |

Options complémentaires utiles : `sar -q` (charge moyenne), `sar -n DEV` (réseau), `sar -d -p` (noms de disques lisibles), `sar -S` (utilisation du swap).

### 1.4 Syntaxe générale

```
sar [options] [intervalle [nombre]]  [-f fichier]  [-s heure_début] [-e heure_fin]
```

| Exemple | Effet |
|---------|-------|
| `sar -u` | CPU **de la journée** (historique) |
| `sar -u 2 5` | **en direct** : 5 mesures espacées de 2 s |
| `sar -u -f /var/log/sa/sa15` | CPU du jour **15** |
| `sar -u -s 10:00:00 -e 12:00:00` | CPU entre 10h et 12h |

### 1.5 Interpréter les valeurs

| Constat | Interprétation |
|---------|----------------|
| `%iowait` élevé | le CPU **attend le disque** (goulot d'étranglement I/O) |
| `%user` élevé | applications consommatrices de CPU |
| `%system` élevé | beaucoup d'appels noyau (I/O, réseau) |
| `%idle` faible | CPU saturé |
| `%memused` élevé + `pswpin/s`/`pswpout/s` > 0 | **manque de RAM** : le système échange avec le swap → ralentissement |
| `%util` ≈ 100 % ou `await` élevé (`-d`) | disque **saturé** |
| `tps`, `bread/s`, `bwrtn/s` élevés (`-b`) | forte activité disque |

---

## 2. Plan du TP

1. Installer sysstat
2. Activer le collecteur
3. Modifier la fréquence de collecte (10 min → 2 min)
4. Utiliser `sar` (CPU, mémoire, swap, I/O)
5. Lire l'historique (`/var/log/sa/saXX`)
6. Diagnostiquer un ralentissement

---

## 3. Pratique

### Étape 1 : Installation

```bash
# Debian / Ubuntu
sudo apt-get install -y sysstat

# CentOS / AlmaLinux / RHEL
sudo yum install -y sysstat
```

### Étape 2 : Activer le collecteur

**Debian / Ubuntu** : éditer `/etc/default/sysstat` et mettre :

```
ENABLED="true"
```

**RHEL / CentOS / AlmaLinux** : démarrer le service :

```bash
sudo systemctl enable --now sysstat
```

- `enable --now` : démarre immédiatement **et** au boot.

### Étape 3 : Modifier la fréquence de collecte

**Recommandation :** passer de **10 minutes** à **2 minutes**.

**Debian / Ubuntu** : `/etc/cron.d/sysstat`

```
*/2 * * * * root command -v debian-sa1 > /dev/null && debian-sa1 1 1
```

**CentOS / RHEL / AlmaLinux** :

```
*/2 * * * * root /usr/lib64/sa/sa1 1 1
```

```bash
sudo systemctl restart sysstat
```

| Élément | Signification |
|---------|---------------|
| `*/2 * * * *` | cron : **toutes les 2 minutes** |
| `root` | utilisateur qui exécute la commande |
| `sa1 1 1` | 1 mesure d'une durée de 1 seconde |
| `debian-sa1` | version Debian de `sa1` |

> 💡 **À vérifier sur AlmaLinux 8 :** sysstat y utilise des **timers systemd** (`sysstat-collect.timer`, toutes les 10 min) et non plus `/etc/cron.d/sysstat`. Si la ligne cron ne change rien :
> ```bash
> sudo systemctl edit sysstat-collect.timer
> ```
> ```
> [Timer]
> OnCalendar=
> OnCalendar=*:00/2
> ```
> ```bash
> sudo systemctl restart sysstat-collect.timer
> systemctl list-timers | grep sysstat
> ```

> 💡 Durée de conservation de l'historique : `HISTORY=28` (jours) dans `/etc/sysconfig/sysstat` (RHEL) ou `/etc/sysstat/sysstat` (Debian).

### Étape 4 : Utiliser `sar`

```bash
sar -u      # CPU
sar -r      # Mémoire
sar -W      # Swap
sar -b      # I/O disque (global)
sar -d      # I/O par disque
sar -A      # Tout afficher
```

- Sans fichier ni intervalle, `sar` lit l'**historique du jour** (`/var/log/sa/saXX`).
- Pour une mesure **immédiate** : `sar -u 2 5`.
- ⚠️ Juste après l'installation, il n'y a **pas encore d'historique** : attendre 1 à 2 cycles de collecte.

**Exemple de lecture (`sar -u`) :**

```
12:00:01   CPU   %user   %nice   %system   %iowait   %steal   %idle
12:02:01   all    3.10    0.00      1.20      0.05     0.00    95.65
```

### Étape 5 : Emplacement et lecture de l'historique

```bash
ls /var/log/sa/
sar -u -f /var/log/sa/sa15
```

- `/var/log/sa/saXX` : statistiques **journalières** (binaires), `XX` = jour du mois.
- `/var/log/sa/sarXX` : rapport **texte** (si `sa2` est actif).
- `-f` : lit un **fichier** d'historique au lieu du jour courant.
- Autres jours : `sar -r -f /var/log/sa/sa10`, etc.

### Étape 6 : Diagnostiquer un ralentissement (méthode)

1. **Repérer la plage horaire** du problème : `sar -u -s 14:00:00 -e 16:00:00`.
2. **CPU** (`-u`) : `%iowait` ou `%user` élevé ?
3. **Mémoire / swap** (`-r`, `-W`) : `%memused` proche de 100 % et `pswpin/s`/`pswpout/s` non nuls ?
4. **Disque** (`-b`, `-d`) : `tps` élevé, `%util` proche de 100 % ?
5. **Charge** : `sar -q` (`ldavg-1/5/15`, `runq-sz`).
6. Recouper avec l'activité (cron, sauvegarde, pic d'utilisateurs).

Pour **reproduire** une charge et l'observer : `stress-ng --cpu 4 --timeout 60s` (voir le TP collectd).

---

## 4. Dépannage

| Symptôme | Cause probable | Solution |
|----------|----------------|----------|
| `Cannot open /var/log/sa/saXX: No such file` | pas encore de collecte / jour sans données | attendre un cycle, `systemctl status sysstat`, vérifier le timer/cron |
| `sar` n'affiche que « Linux … » et rien d'autre | historique vide | attendre ; ou utiliser `sar -u 2 5` |
| La fréquence ne change pas (AlmaLinux 8) | collecte par **timer systemd**, pas par cron | surcharger `sysstat-collect.timer` |
| Pas de collecte sur Debian | `ENABLED="false"` | mettre `ENABLED="true"` et redémarrer |
| Fichier du mois précédent écrasé | rotation sur 28 jours | adapter `HISTORY` |
| Noms de disques illisibles (`dev8-0`) | affichage par numéros | `sar -d -p` |
| Date/heure inattendue | fuseau horaire | `timedatectl` |
| Logs | | `journalctl -u sysstat`, `systemctl list-timers` |

---

## 5. Questions de révision

1. Qu'est-ce que sysstat ? Quelle différence entre `sar`, `sa1` et `sa2` ?
2. Où sont stockées les statistiques ? Que signifie `saXX` ?
3. Comment activer la collecte sous Debian/Ubuntu ? sous RHEL/AlmaLinux ?
4. Comment passer la collecte de 10 à 2 minutes ? Que signifie `*/2 * * * *` ?
5. Quelle option de `sar` pour le CPU, la mémoire, le swap, l'I/O disque global, l'I/O par disque ?
6. Comment lire les statistiques du jour 15 ?
7. Que signifie un `%iowait` élevé ?
8. Comment reconnaître un **manque de mémoire** avec `sar` ?
9. Comment retrouver ce qui s'est passé sur une plage horaire passée ?
10. Pourquoi `sar` n'affiche-t-il rien juste après l'installation ?
11. Quelle différence entre `sar -u` et `sar -u 2 5` ?
