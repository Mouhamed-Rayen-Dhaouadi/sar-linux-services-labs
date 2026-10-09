# TP 01 – Configuration du noyau Linux : modules et compilation

**Distribution :** AlmaLinux 8 · **Outils :** `gcc`, `make`, `kmod`

---

## 1. Théorie

### 1.1 Le noyau Linux et les modules

- Le **noyau** (kernel) est le cœur du système : il gère CPU, mémoire, périphériques, réseau, systèmes de fichiers.
- Linux est un noyau **monolithique modulaire** : les fonctions de base sont dans l'image du noyau, mais on peut **ajouter ou retirer du code à chaud** sous forme de **modules** (fichiers `.ko`, *kernel object*).
- Un module s'exécute en **espace noyau** (*kernel space*) avec tous les privilèges : une erreur peut faire planter la machine (*kernel panic*).
- Un module n'a pas de `main()` : il fournit deux fonctions de rappel
  - **init** : appelée au chargement (`insmod`/`modprobe`)
  - **exit / cleanup** : appelée au déchargement (`rmmod`/`modprobe -r`)
- Un module ne peut pas utiliser `printf` (pas de libc dans le noyau) ; il utilise **`printk`** avec un niveau (`KERN_INFO`, `KERN_ERR`…). Les messages sont lus avec **`dmesg`** ou dans `/var/log/messages`.

### 1.2 Compiler un module : Kbuild

- Un module externe se compile **contre l'arbre du noyau en cours d'exécution** : `/lib/modules/$(uname -r)/build` (fourni par le paquet `kernel-devel`).
- Le `Makefile` du module contient `obj-m += nom.o` : il dit à **Kbuild** de produire `nom.ko`.
- Les `MODULE_LICENSE`, `MODULE_AUTHOR`… alimentent `modinfo`. Sans licence `GPL`, le noyau est marqué **« tainted »** (message `module license 'unspecified' taints kernel`).

### 1.3 `insmod` vs `modprobe`

| | `insmod` | `modprobe` |
|---|---------|-----------|
| Chemin du module | n'importe lequel (`./x.ko`) | doit être dans `/lib/modules/$(uname -r)/` |
| Dépendances | **non** résolues | **résolues** via `modules.dep` |
| Déchargement | `rmmod` | `modprobe -r` |

`depmod -a` (re)génère `modules.dep`, la table des dépendances utilisée par `modprobe`.

### 1.4 Compiler un noyau complet

| Élément | Rôle |
|---------|------|
| `.config` | fichier de configuration du noyau (options `=y` intégré, `=m` module, non défini = absent) |
| `make menuconfig` | interface semi-graphique (ncurses) pour éditer `.config` |
| `bzImage` | image compressée du noyau (`vmlinuz`) |
| `make modules_install` | copie les modules dans `/lib/modules/<version>/` |
| `make install` | copie `vmlinuz`, `System.map`, crée l'`initramfs` et met à jour **GRUB2** |

---

## 2. Plan du TP

**Partie A : module**
1. Écrire `my_module.c`
2. Écrire le `Makefile` et compiler
3. Consulter le module avec `modinfo`
4. Charger / décharger (`insmod`, `rmmod`) et lire les logs
5. Charger avec `modprobe` (copie dans `extra/` + `depmod`)
6. Charger automatiquement au démarrage

**Partie B : noyau 5.15**
1. Télécharger et extraire les sources
2. Préparer `.config`
3. Compiler
4. Installer, redémarrer, vérifier avec `uname -r`

---

## 3. Pratique

### 3.0 Prérequis

```bash
sudo yum groupinstall "Development tools" -y
sudo yum install -y kernel-devel-$(uname -r) kernel-headers elfutils-libelf-devel
```

- `groupinstall "Development tools"` : installe `gcc`, `make`, `patch`, etc.
- `kernel-devel-$(uname -r)` : en-têtes + `Makefile`s du noyau **exactement de la version en cours** (`$(uname -r)` est remplacé par la version, ex. `4.18.0-477…`). Sans lui, `/lib/modules/.../build` n'existe pas.
- `elfutils-libelf-devel` : nécessaire à la compilation de modules récents.

### 3.1 Partie A : le module

#### Étape 1 : `my_module.c`

```c
#include <linux/module.h>
#include <linux/kernel.h>

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Arctic");
MODULE_DESCRIPTION("Module hello world");
MODULE_VERSION("Version 1.00");

int init_module(void)
{
    printk(KERN_INFO "Chargement de module .... \n");
    return 0;                       /* 0 = succès */
}

void cleanup_module(void)
{
    printk(KERN_INFO "Déchargement de module ... \n");
}
```

- `#include <linux/module.h>` : macros et types nécessaires à tout module.
- `MODULE_*` : métadonnées lisibles avec `modinfo`.
- `init_module()` : exécutée au chargement ; retourner une valeur ≠ 0 **refuse** le chargement.
- `cleanup_module()` : exécutée au déchargement.
- `printk(KERN_INFO …)` : écrit dans le tampon du noyau (visible via `dmesg`).

💡 Version moderne équivalente : `static int __init mon_init(void)` + `module_init(mon_init);` et `module_exit(mon_exit);`.

#### Étape 2 : `Makefile` (dans le même dossier)

```make
obj-m += my_module.o
all:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules
clean:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
```

- `obj-m += my_module.o` : « construire `my_module.ko` à partir de `my_module.c` ».
- `make -C <dir>` : se placer dans le dossier du noyau pour utiliser **son** système de build.
- `M=$(PWD)` : indique où se trouve le module externe à compiler.
- `modules` / `clean` : cibles de compilation / nettoyage.

⚠️ Les lignes de commande dans un Makefile commencent par une **tabulation**, pas des espaces.

```bash
make
```

→ génère `my_module.ko` (ainsi que `.o`, `.mod.c`, `Module.symvers`…).

#### Étape 3 : informations

```bash
modinfo my_module.ko
```

Affiche : chemin (`filename`), `version`, `description`, `author`, `license`, `depends`, `vermagic` (version du noyau pour laquelle il est compilé).

#### Étape 4 : chargement manuel

```bash
sudo insmod my_module.ko
lsmod | grep my_module
sudo rmmod my_module
dmesg | tail
sudo cat /var/log/messages
```

- `insmod my_module.ko` : charge le module (appelle `init_module`).
- `lsmod` : liste les modules chargés (nom, taille, nombre d'utilisations) ; `grep` filtre la ligne voulue.
- `rmmod my_module` : décharge (appelle `cleanup_module`). Le **nom** du module est sans `.ko`.
- `dmesg | tail` : les dernières lignes du tampon noyau → doit montrer « Chargement… » puis « Déchargement… ».
- `/var/log/messages` : journal système où ces messages sont aussi enregistrés.

#### Étape 5 : `modprobe`

```bash
sudo mkdir -p /lib/modules/$(uname -r)/extra
sudo cp my_module.ko /lib/modules/$(uname -r)/extra/
sudo depmod -a
sudo modprobe my_module
dmesg | tail
sudo modprobe -r my_module
dmesg | tail
```

- `extra/` : dossier conventionnel pour les modules hors arbre du noyau.
- `depmod -a` : analyse **tous** les modules et régénère `modules.dep` ; indispensable pour que `modprobe` connaisse le nouveau module.
- `modprobe my_module` : charge le module (et ses dépendances) par son nom.
- `modprobe -r my_module` : décharge (et les dépendances devenues inutiles).

#### Étape 6 : chargement au démarrage

```bash
echo "my_module" | sudo tee /etc/modules-load.d/my_module.conf
sudo reboot
# après redémarrage :
lsmod | grep my_module
```

- Le service `systemd-modules-load` lit chaque fichier `/etc/modules-load.d/*.conf` au boot et charge les modules qui y sont listés (un nom par ligne).
- `tee` écrit dans un fichier **avec les droits root** (un simple `>` serait exécuté avec les droits de l'utilisateur).

---

### 3.2 Partie B : compiler et installer le noyau 5.15

#### Étape 1 : sources

```bash
cd /tmp
wget https://cdn.kernel.org/pub/linux/kernel/v5.x/linux-5.15.32.tar.xz
sudo cp linux-5.15.32.tar.xz /usr/src/
cd /usr/src
sudo tar xvf linux-5.15.32.tar.xz
cd linux-5.15.32
```

- `wget URL` : télécharge l'archive depuis kernel.org.
- `tar xvf` : **x** extraire, **v** verbeux, **f** fichier ; `.tar.xz` est décompressé automatiquement.
- `/usr/src` : emplacement habituel des sources.

#### Étape 2 : configuration

```bash
sudo yum groupinstall "Development Tools" -y
sudo yum install -y ncurses-devel openssl-devel bison flex bc perl elfutils-libelf-devel
sudo cp -v /boot/config-$(uname -r) .config
sudo make menuconfig
```

- `ncurses-devel` : bibliothèque dont `menuconfig` a besoin ; `openssl-devel`, `bison`, `flex`, `bc`… : requis par la compilation du noyau.
- `cp /boot/config-$(uname -r) .config` : repart de la configuration **du noyau actuel** (qui fonctionne) au lieu de tout configurer à la main.
- `make menuconfig` : menu pour modifier les options (`[*]` intégré, `<M>` module, `[ ]` désactivé). Enregistrer puis quitter (Save / Exit).

⚠️ Si la compilation échoue sur des certificats (`SYSTEM_TRUSTED_KEYS`), désactiver ces options :

```bash
sudo scripts/config --disable SYSTEM_TRUSTED_KEYS --disable SYSTEM_REVOCATION_KEYS
sudo make olddefconfig
```

(`olddefconfig` : met à jour `.config` en acceptant les valeurs par défaut pour les nouvelles options.)

#### Étape 3 : compilation

```bash
sudo make bzImage && sudo make modules
# plus rapide, en parallèle :
sudo make -j$(nproc)
```

- `make bzImage` : compile l'image du noyau.
- `make modules` : compile tous les modules.
- `&&` : exécute la seconde commande **seulement si** la première réussit.
- `-j$(nproc)` : autant de tâches parallèles que de cœurs CPU (`nproc` = nombre de cœurs).

⏱️ Plusieurs dizaines de minutes ; prévoir **≥ 20 Go** d'espace disque.

#### Étape 4 : installation

```bash
sudo make modules_install && sudo make install
```

- `modules_install` : copie les modules dans `/lib/modules/5.15.32/`.
- `install` : place `vmlinuz-5.15.32` et `System.map` dans `/boot`, génère l'**initramfs** et **ajoute l'entrée dans GRUB2** (`/boot/grub2/grub.cfg`).

```bash
sudo reboot
# dans le menu GRUB : choisir « AlmaLinux (5.15.32) »
uname -r        # → 5.15.32
```

- `uname -r` : affiche la version du noyau **en cours d'exécution**.

---

## 4. Vérification / dépannage

| Symptôme | Cause probable | Solution |
|----------|----------------|----------|
| `make: *** /lib/modules/…/build: No such file` | `kernel-devel` absent ou d'une autre version | installer `kernel-devel-$(uname -r)` |
| `Makefile:… *** missing separator` | espaces au lieu de tabulation | remplacer par une tabulation |
| `insmod: ERROR: Invalid module format` | module compilé pour un autre noyau | recompiler avec le noyau en cours |
| `modprobe: FATAL: Module … not found` | oubli de `depmod -a` ou mauvais dossier | copier dans `extra/` puis `depmod -a` |
| Rien dans `dmesg` | tampon saturé / droits | `sudo dmesg \| tail`, ou `journalctl -k` |
| Le noyau 5.15.32 n'apparaît pas dans GRUB | `make install` non exécuté | relancer, puis `grub2-mkconfig -o /boot/grub2/grub.cfg` |

---

## 5. Questions de révision

1. Quelle différence entre `insmod` et `modprobe` ?
2. À quoi sert `depmod -a` ?
3. Pourquoi `printk` et non `printf` ?
4. Que signifie `obj-m` ? Et `M=$(PWD)` ?
5. Que contient `/boot/config-$(uname -r)` et pourquoi le copier en `.config` ?
6. Quelles commandes installent un noyau compilé, et que modifient-elles ?
7. Comment charger un module automatiquement au démarrage ?
