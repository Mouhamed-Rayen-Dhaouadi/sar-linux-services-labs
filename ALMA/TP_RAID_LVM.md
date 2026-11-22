# TP – Atelier RAID et LVM : stockage logiciel sous Linux

**Distribution :** AlmaLinux 8 (`dnf`) · **Outils :** `mdadm` (RAID logiciel), `lvm2` (volumes logiques) · **Disques :** 4 disques virtuels supplémentaires (`sdb` 1 Go, `sdc` 1,2 Go, `sdd` 1,5 Go, `sde` 2 Go)

> ⚠️ L'énoncé ne contient que des **consignes** (pas de solution). Les réponses aux questions sont **mon interprétation** : à confronter avec ton cours et à tester sur ta VM.

---

## 1. Théorie

### 1.1 RAID

**RAID** (*Redundant Array of Independent Disks*) regroupe plusieurs disques en **un seul volume** pour gagner en **performance**, en **tolérance de panne**, ou les deux.

- **RAID logiciel** : géré par le noyau Linux via `mdadm`. Le volume apparaît sous `/dev/mdX`.
- ⚠️ **RAID ≠ sauvegarde** : le RAID protège d'une panne de disque, pas d'une suppression, d'un virus ou d'une erreur humaine.

### 1.2 Niveaux RAID

| Niveau | Principe | Disques min. | Capacité utile | Tolérance de panne | Remarque |
|--------|----------|:---:|----------------|--------------------|----------|
| **RAID 0** | *Striping* : données réparties en bandes | 2 | somme des disques | **aucune** | rapide, **1 panne = tout perdu** |
| **RAID 1** | *Mirroring* : copie identique | 2 | **1 disque** (le plus petit) | n-1 disques | lecture rapide, 50 % de perte |
| **RAID 5** | Striping + **1 parité** | 3 | (n-1) × petit disque | **1 disque** | bon compromis |
| **RAID 6** | Striping + **2 parités** | 4 | (n-2) × petit disque | **2 disques** | plus sûr, **écriture plus lente** |
| **RAID 10** | miroir de bandes (1+0) | 4 | n/2 | 1 par miroir | performant et sûr |

- Avec des disques de tailles différentes, RAID 1/5/6 limitent la capacité au **plus petit disque** de chaque membre.

### 1.3 LVM

**LVM** (*Logical Volume Manager*) ajoute une couche d'abstraction souple entre les disques et le système de fichiers.

```
Disques / RAID (/dev/md0)  →  PV  →  VG  →  LV  →  Système de fichiers  →  Montage
```

| Élément | Nom | Rôle | Commandes |
|---------|-----|------|-----------|
| **PV** | Physical Volume | disque ou RAID préparé pour LVM | `pvcreate`, `pvdisplay`, `pvremove` |
| **VG** | Volume Group | pool regroupant un ou plusieurs PV | `vgcreate`, `vgdisplay`, `vgremove` |
| **LV** | Logical Volume | « partition » souple découpée dans le VG | `lvcreate`, `lvdisplay`, `lvremove` |

- Avantage : redimensionner un volume (`lvextend`), ajouter des disques au VG, faire des snapshots.
- **RAID sous LVM** : le RAID assure la **protection**, LVM la **souplesse**.

### 1.4 États d'un RAID (`/proc/mdstat`)

| Indication | Signification |
|-----------|---------------|
| `[UU]` | tous les disques sont **actifs** (Up) |
| `[U_]` | un disque manquant/défaillant → **dégradé** |
| `(F)` | disque marqué **Failed** |
| `recovery` / `resync` | **reconstruction** en cours |

---

## 2. Plan du TP

1. Préparer l'environnement (disques, `mdadm`, `lvm2`)
2. **RAID 0** + LVM + tests de performance + panne
3. **RAID 1** + LVM + panne + remplacement + reconstruction
4. **RAID 6** + LVM (XFS) + double panne

---

## 3. Pratique

### Partie 1 : Préparation

Ajouter **4 disques virtuels** dans l'hyperviseur : `sdb` (1 Go), `sdc` (1,2 Go), `sdd` (1,5 Go), `sde` (2 Go). Vérifier :

```bash
lsblk
```

Installer les outils :

```bash
sudo dnf install -y mdadm lvm2
```

---

### Partie 2 : RAID 0 (Striping)

#### 2.1 Création avec 2 disques

```bash
sudo mdadm --create /dev/md0 --level=0 --raid-devices=2 /dev/sdb /dev/sdc
```

| Option | Rôle |
|--------|------|
| `--create /dev/md0` | crée le périphérique RAID `md0` |
| `--level=0` | niveau RAID 0 |
| `--raid-devices=2` | nombre de disques actifs |

#### 2.2 Vérification

```bash
cat /proc/mdstat
sudo mdadm --detail /dev/md0
```

- Attendu : `md0 : active raid0 sdc[1] sdb[0]`, capacité ≈ somme des 2 disques, état `clean`.

#### 2.3 Tests de performance

```bash
# Écriture : 1 Go de zéros
sudo dd if=/dev/zero of=/dev/md0 bs=1M count=1024

# Lecture
sudo hdparm -tT /dev/md0
```

- `dd` : `if` = source, `of` = destination, `bs=1M` = taille de bloc, `count=1024` → 1024 × 1 Mo = **1 Go**.
- `/dev/zero` : flux infini d'octets nuls.
- `hdparm -t` : lecture **disque** (sans cache) ; `-T` : lecture depuis le **cache**.
- ⚠️ Ici `dd` écrit **directement sur le périphérique** (pas dans un fichier) : cela écrase son contenu. C'est sans conséquence ici car LVM n'est pas encore créé. `hdparm` peut nécessiter `sudo dnf install -y hdparm`.

#### 2.4 LVM sur RAID 0

```bash
# Physical Volume
sudo pvcreate /dev/md0
sudo pvdisplay

# Volume Group
sudo vgcreate vg_raid0 /dev/md0
sudo vgdisplay vg_raid0

# Logical Volume (80 % de l'espace)
sudo lvcreate -l 80%VG -n lv_data vg_raid0
sudo lvdisplay

# Formater et monter
sudo mkfs.ext4 /dev/vg_raid0/lv_data
sudo mkdir -p /mnt/raid0
sudo mount /dev/vg_raid0/lv_data /mnt/raid0

# Vérifier
df -h /mnt/raid0
```

- `-l 80%VG` : **80 % de l'espace total du VG** (en extents) ; `-n lv_data` : nom du LV. (`-L 500M` donnerait une taille fixe.)
- `-l 100%FREE` (utilisé plus loin) : **tout l'espace libre** restant.
- Chemin du LV : `/dev/<VG>/<LV>` (= `/dev/mapper/vg_raid0-lv_data`).

#### 2.5 Marquer un disque comme défaillant

```bash
sudo mdadm --manage /dev/md0 --fail /dev/sdc
cat /proc/mdstat
sudo mdadm --detail /dev/md0
```

**Qu'observe-t-on ? Quelle solution ?** *(à vérifier sur ta VM)*

- RAID 0 n'a **aucune redondance** : les données sont réparties sur les 2 disques, donc la perte de **un seul** disque rend **tout le volume inutilisable**.
- Selon le noyau, mdadm peut même **refuser** de marquer le disque défaillant (`Device or resource busy`), car le tableau ne peut pas fonctionner sans lui.
- **Solutions** :
  - **Prévention** : utiliser un RAID **avec redondance** (1, 5, 6, 10) ;
  - **Sauvegardes** régulières (le RAID 0 ne se « répare » pas : il faut recréer le volume et **restaurer** depuis la sauvegarde) ;
  - RAID 0 réservé aux données **temporaires/non critiques** (cache, calcul).

---

### Partie 3 : RAID 1 (Mirroring)

Le RAID 1 **duplique** les données sur plusieurs disques : la panne d'un disque ne détruit aucune donnée.

#### 3.1 Nettoyer le RAID 0 précédent

```bash
sudo umount /mnt/raid0
sudo lvremove /dev/vg_raid0/lv_data
sudo vgremove vg_raid0
sudo pvremove /dev/md0
sudo mdadm --stop /dev/md0
sudo mdadm --zero-superblock /dev/sdb /dev/sdc
```

> Toujours démonter et supprimer **de haut en bas** : montage → LV → VG → PV → RAID → superblock.

| Commande | Rôle |
|----------|------|
| `umount` | démonte le système de fichiers |
| `lvremove` / `vgremove` / `pvremove` | supprime LV / VG / PV |
| `mdadm --stop` | arrête le RAID |
| `mdadm --zero-superblock` | efface les **métadonnées RAID** des disques (pour les réutiliser) |

#### 3.2 Création du RAID 1

```bash
sudo mdadm --create /dev/md1 --level=1 --raid-devices=2 /dev/sdb /dev/sdc
```

- Capacité = **le plus petit disque** (≈ 1 Go). Le reste de `sdc` est inutilisé.
- Synchronisation initiale : suivre avec `cat /proc/mdstat`.

#### 3.3 LVM sur RAID 1

```bash
sudo pvcreate /dev/md1
sudo vgcreate vg_raid1 /dev/md1
sudo lvcreate -l 100%FREE -n lv_secure vg_raid1

sudo mkfs.ext4 /dev/vg_raid1/lv_secure
sudo mkdir -p /mnt/raid1
sudo mount /dev/vg_raid1/lv_secure /mnt/raid1
```

#### 3.4 Test de défaillance

```bash
# Données de test (100 Mo aléatoires)
sudo dd if=/dev/urandom of=/mnt/raid1/testfile bs=1M count=100
md5sum /mnt/raid1/testfile          # noter l'empreinte (avant panne)

# Simuler la panne de sdc
sudo mdadm --manage /dev/md1 --fail /dev/sdc
sudo mdadm --detail /dev/md1
```

**Vérifier l'intégrité des données. Qu'observe-t-on ? Interprétation.**

- `md5sum` donne **la même empreinte** : le fichier est **intact et accessible**.
- `mdadm --detail` : état **`clean, degraded`**, `sdc` marqué **`faulty`** (`[U_]` dans `/proc/mdstat`).
- **Interprétation** : chaque donnée existe en **double** (un exemplaire par disque) ; `sdb` contient une copie complète, donc le service **continue** sans interruption. Le volume est cependant **fragilisé** : une 2ᵉ panne = perte de données → il faut **remplacer vite** le disque.

#### 3.5 Remplacement du disque

```bash
# Retirer le disque défaillant
sudo mdadm --manage /dev/md1 --remove /dev/sdc

# Ajouter le disque de remplacement
sudo mdadm --manage /dev/md1 --add /dev/sdc
```

**Surveiller la reconstruction :**

```bash
watch -n 1 cat /proc/mdstat
sudo mdadm --detail /dev/md1
```

- `/proc/mdstat` affiche `recovery = xx %` avec la vitesse et le temps estimé.
- Fin : retour à `[UU]`, état `clean`.
- Vérifier ensuite : `md5sum /mnt/raid1/testfile` (même empreinte).

| Commande | Rôle |
|----------|------|
| `--fail` | marque un disque comme défaillant (simulation de panne) |
| `--remove` | retire le disque du RAID (il doit être `faulty`) |
| `--add` | ajoute un disque ; déclenche la **reconstruction** |

---

### Partie 4 : RAID 6 (double parité)

Le RAID 6 tolère la perte de **2 disques**. Capacité = **(n − 2) × plus petit disque**, **4 disques minimum**. Plus sûr que RAID 5, mais **écriture plus lente** (double calcul de parité).

#### 4.1 Nettoyer le RAID 1

```bash
sudo umount /mnt/raid1
sudo lvremove /dev/vg_raid1/lv_secure
sudo vgremove vg_raid1
sudo pvremove /dev/md1
sudo mdadm --stop /dev/md1
sudo mdadm --zero-superblock /dev/sdb /dev/sdc
```

#### 4.2 Création du RAID 6

```bash
sudo mdadm --create /dev/md6 --level=6 --raid-devices=4 /dev/sdb /dev/sdc /dev/sdd /dev/sde

sudo mdadm --detail /dev/md6
cat /proc/mdstat
```

- Capacité : (4 − 2) × 1 Go (plus petit disque) ≈ **2 Go**.
- Une synchronisation initiale démarre : attendre la fin pour des tests propres.

#### 4.3 LVM et système de fichiers XFS

```bash
sudo pvcreate /dev/md6
sudo vgcreate vg_raid6 /dev/md6
sudo lvcreate -l 100%FREE -n lv_critical vg_raid6

sudo mkfs.xfs /dev/vg_raid6/lv_critical
sudo mkdir -p /mnt/raid6
sudo mount /dev/vg_raid6/lv_critical /mnt/raid6
```

- `mkfs.xfs` : système de fichiers **XFS** (par défaut sur RHEL/AlmaLinux 8, performant avec gros fichiers). Contrairement à ext4, un XFS **ne peut pas être réduit**.

#### 4.4 Test de double panne

```bash
sudo dd if=/dev/urandom of=/mnt/raid6/important.dat bs=1M count=200
md5sum /mnt/raid6/important.dat      # noter l'empreinte

# Panne de 2 disques
sudo mdadm --manage /dev/md6 --fail /dev/sdc
sudo mdadm --manage /dev/md6 --fail /dev/sdd

sudo mdadm --detail /dev/md6
cat /proc/mdstat
md5sum /mnt/raid6/important.dat      # comparer
```

**Données toujours accessibles ? Interprétation.**

- Oui : le fichier est **accessible et identique** ; le RAID est **dégradé** (`[UU__]`, 2 disques `faulty`).
- **Interprétation** : la **double parité** (P et Q) permet de recalculer les données de **2 disques manquants** à partir des 2 restants.
- ⚠️ Plus aucune marge : une **3ᵉ panne** détruirait le volume. Il faut **remplacer** les disques (`--remove` puis `--add`) et laisser la reconstruction se terminer.

---

### Compléments utiles (hors énoncé)

**Rendre le RAID et le montage persistants au reboot :**

```bash
sudo mdadm --detail --scan | sudo tee -a /etc/mdadm.conf
sudo dracut -f                       # régénère l'initramfs (facultatif)
echo '/dev/vg_raid6/lv_critical /mnt/raid6 xfs defaults 0 0' | sudo tee -a /etc/fstab
```

**Arrêter et supprimer complètement un RAID :**

```bash
sudo mdadm --stop /dev/mdX
sudo mdadm --zero-superblock /dev/sdX
```

---

## 4. Dépannage

| Symptôme | Cause probable | Solution |
|----------|----------------|----------|
| `mdadm: device busy` | volume monté/utilisé (LVM actif) | `umount`, `lvremove`/`vgchange -an`, puis `mdadm --stop` |
| `mdadm: Cannot open ... busy` à la création | disque encore dans un ancien RAID | `mdadm --stop`, `--zero-superblock` |
| `pvcreate` : *device excluded by filter* | signature existante | `wipefs -a /dev/md0` |
| RAID disparaît après reboot | `mdadm.conf` non renseigné | `mdadm --detail --scan >> /etc/mdadm.conf` |
| RAID `degraded` | disque en panne | `--remove` puis `--add` un disque de remplacement |
| Reconstruction trop lente | limites de vitesse du noyau | `/proc/sys/dev/raid/speed_limit_min` |
| `mount` : mauvais type de FS | mauvais `mkfs`/type | vérifier avec `blkid` |
| Observer l'état | | `cat /proc/mdstat`, `mdadm --detail`, `lsblk`, `journalctl -k` |

---

## 5. Questions de révision

1. Qu'est-ce que le RAID ? Pourquoi dit-on que **RAID ≠ sauvegarde** ?
2. Quelle est la différence entre RAID 0, 1, 5, 6 et 10 (principe, capacité, tolérance) ?
3. Pourquoi une panne sur un RAID 0 détruit-elle tout le volume ?
4. Quelle est la capacité utile d'un RAID 1 de 1 Go + 1,2 Go ? d'un RAID 6 de 4 disques ?
5. Que fait la commande `mdadm --create /dev/md0 --level=0 --raid-devices=2 ...` ?
6. Comment lire `/proc/mdstat` (`[UU]`, `[U_]`, `(F)`, `recovery`) ?
7. À quoi servent `--fail`, `--remove` et `--add` ?
8. À quoi sert `mdadm --zero-superblock` ?
9. Que sont PV, VG et LV ? Dans quel ordre les crée-t-on ?
10. Différence entre `lvcreate -l 80%VG` et `-l 100%FREE` ?
11. Pourquoi supprimer dans l'ordre inverse (umount → lv → vg → pv → md) ?
12. Pourquoi le RAID 6 tolère-t-il 2 pannes ? Quel est son inconvénient ?
13. Que mesure `hdparm -tT` ? Que fait `dd if=/dev/zero of=... bs=1M count=1024` ?
14. Pourquoi combiner **RAID + LVM** ?
