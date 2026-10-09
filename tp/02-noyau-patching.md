# TP 02 – Patching du noyau Linux (6.1 → 6.1.1)

**Objectif :** appliquer un patch officiel sur les sources du noyau, puis (optionnel) recompiler et démarrer dessus.

---

## 1. Théorie

### 1.1 Qu'est-ce qu'un patch ?

- Un **patch** (ou *diff*) est un fichier texte qui décrit les **différences** entre deux versions de fichiers : lignes ajoutées (`+`), supprimées (`-`), avec leur contexte.
- L'outil **`patch`** lit ce fichier et modifie les sources pour passer de la version A à la version B.
- Avantage : télécharger quelques Mo de patch plutôt que l'archive complète (~130 Mo).

### 1.2 Numérotation du noyau

Format `VERSION.PATCHLEVEL.SUBLEVEL` (+ `EXTRAVERSION`) dans le `Makefile` racine :

```make
VERSION = 6
PATCHLEVEL = 1
SUBLEVEL = 0          # ← devient 1 après le patch 6.1.1
EXTRAVERSION =
```

- **6.1** = branche **LTS** (*Long Term Support*).
- **6.1.1** = première mise à jour de maintenance (correctifs de bugs/sécurité) de la branche 6.1.
- Le fichier `patch-6.1.1.xz` est un patch **incrémental depuis 6.1** (il s'applique sur un arbre 6.1.0 propre). Pour aller en 6.1.2 depuis 6.1.1, il faut le patch 6.1.2 *incrémental* correspondant, ou repartir de 6.1 avec le patch cumulatif.

### 1.3 L'option `-p1`

Dans le patch, les chemins ressemblent à `a/kernel/sched/core.c`. `-p1` retire **1 niveau** de répertoire (`a/`) pour retrouver `kernel/sched/core.c` quand on est à la racine des sources.

---

## 2. Plan du TP

1. Préparation : télécharger/extraire 6.1, vérifier le `Makefile`, télécharger le patch
2. Appliquer le patch, vérifier le `Makefile` (SUBLEVEL = 1)
3. *(Optionnel)* Compiler le noyau patché
4. Mettre à jour GRUB et redémarrer
5. Vérifier qu'on utilise bien le noyau patché

---

## 3. Pratique

### 3.1 Préparation

```bash
cd ~
wget https://cdn.kernel.org/pub/linux/kernel/v6.x/linux-6.1.tar.xz
wget https://cdn.kernel.org/pub/linux/kernel/v6.x/patch-6.1.1.xz
tar -xvf linux-6.1.tar.xz
unxz patch-6.1.1.xz
cd linux-6.1
head -5 Makefile
```

- `wget` : télécharge les sources 6.1 et le patch.
- `tar -xvf` : extrait l'archive (`x` extraire, `v` verbeux, `f` fichier).
- `unxz patch-6.1.1.xz` : décompresse → fichier `patch-6.1.1`.
- `head -5 Makefile` : affiche les 5 premières lignes → `VERSION = 6`, `PATCHLEVEL = 1`, `SUBLEVEL = 0`.

⚠️ Dans l'énoncé, `cd linux-6.1` est fait **avant** le `wget` du patch, mais la commande de patch utilise `../patch-6.1.1`. Télécharge donc le patch **dans le dossier parent** de `linux-6.1` (comme ci-dessus) pour que `../patch-6.1.1` soit correct.

### 3.2 Application du patch

```bash
patch -p1 --dry-run < ../patch-6.1.1     # simulation (recommandé)
patch -p1 < ../patch-6.1.1
head -5 Makefile
```

- `--dry-run` : **teste** l'application sans rien modifier ; vérifie qu'il n'y aura pas de rejet.
- `patch -p1 < ../patch-6.1.1` : applique le patch ; `<` envoie le contenu du fichier sur l'entrée standard de `patch`.
- Résultat attendu : `SUBLEVEL = 1` dans le `Makefile` → la version devient **6.1.1**.

Fichiers `.rej` / `.orig` : si le patch échoue sur un bloc, il crée un `.rej` (rejeté). Cela signifie que l'arbre source n'est pas la bonne version de départ.

Annuler un patch : `patch -R -p1 < ../patch-6.1.1` (`-R` = *reverse*).

### 3.3 Compilation (optionnelle)

```bash
sudo apt update
sudo apt install -y build-essential libncurses-dev bison flex libssl-dev libelf-dev bc dwarves xz-utils

cp /boot/config-$(uname -r) .config
make oldconfig
make -j$(nproc)
sudo make modules_install
sudo make install
```

- Premier bloc : outils et bibliothèques nécessaires (`build-essential` = gcc, make…, `libssl-dev`, `libelf-dev`, `bison`/`flex` = analyseurs).
- `cp /boot/config-$(uname -r) .config` : reprend la configuration du noyau actuel.
- `make oldconfig` : met à jour `.config` ; pose une question pour chaque **nouvelle** option absente de l'ancienne config (Entrée = valeur par défaut). `make olddefconfig` répond automatiquement par défaut.
- `make -j$(nproc)` : compile noyau + modules en parallèle.
- `make modules_install` : installe les modules dans `/lib/modules/6.1.1/`.
- `make install` : installe `vmlinuz-6.1.1`, `System.map`, `config`, génère l'initramfs.

⚠️ **Piège Ubuntu** : la config d'Ubuntu référence des certificats Canonical absents des sources officielles → erreur `No rule to make target 'debian/canonical-certs.pem'`. Solution :

```bash
scripts/config --disable SYSTEM_TRUSTED_KEYS
scripts/config --disable SYSTEM_REVOCATION_KEYS
```

### 3.4 Bootloader

```bash
sudo update-grub
sudo reboot
```

- `update-grub` : régénère `/boot/grub/grub.cfg` en détectant les noyaux dans `/boot` (ajoute l'entrée 6.1.1).
- Au redémarrage : menu GRUB → *Advanced options for Ubuntu* → choisir le noyau 6.1.1.

### 3.5 Vérifier que le noyau patché est utilisé

```bash
uname -r
cat /proc/version
dmesg | grep "Linux version"
```

- `uname -r` : version du noyau **en cours** → doit afficher `6.1.1`.
- `/proc/version` : chaîne complète (version, compilateur, date de compilation).
- `dmesg | grep "Linux version"` : première ligne du journal noyau au démarrage.
- *(Bonus)* `ls /boot` : doit contenir `vmlinuz-6.1.1`.

---

## 4. Dépannage

| Symptôme | Cause | Solution |
|----------|-------|----------|
| `Hunk #1 FAILED` | sources pas en 6.1.0 | ré-extraire l'archive propre puis re-patcher |
| `can't find file to patch` | mauvais `-p` ou mauvais dossier | se placer à la racine de `linux-6.1`, utiliser `-p1` |
| `uname -r` affiche toujours l'ancien noyau | GRUB a démarré l'ancien | choisir le bon noyau dans GRUB |
| Échec de compilation (certificats) | config Ubuntu | voir le piège ci-dessus |

---

## 5. Questions de révision

1. Qu'est-ce qu'un patch et à quoi sert `-p1` ?
2. Pourquoi faire un `--dry-run` ?
3. Quelle ligne du `Makefile` change après le patch ?
4. Différence entre `make oldconfig` et `make olddefconfig` ?
5. Comment vérifier, après redémarrage, que le noyau patché est utilisé ? (trois méthodes)
6. Comment annuler un patch ?
