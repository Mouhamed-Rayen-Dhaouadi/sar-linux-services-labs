# TP 07 – Atelier NFS (Network File System)

**Environnement :** 2 VM AlmaLinux 8 (serveur et client), VMware Workstation ou VirtualBox
**Réseau :** serveur `192.168.18.6` · client `192.168.18.4` · réseau `192.168.18.0/24`

---

## 1. Théorie

### 1.1 Qu'est-ce que NFS ?

**NFS** permet à une machine **cliente** de monter un répertoire distant (exporté par un **serveur**) et de l'utiliser comme un répertoire local. Typique des environnements Unix/Linux.

- **Serveur** : *exporte* des répertoires (`/etc/exports`).
- **Client** : *monte* l'export dans un point de montage local (`mount -t nfs`).
- Composants : `nfs-server` (service), `rpcbind` (annuaire RPC : indique sur quels ports tournent les services), `mountd`, `nfsd`.
- Port principal : **2049** (NFSv4) ; NFSv3 utilise aussi `rpcbind` (111) et `mountd`.

### 1.2 Options d'export importantes

| Option | Effet |
|--------|-------|
| `ro` / `rw` | lecture seule / lecture-écriture |
| `sync` / `async` | écriture confirmée après écriture disque / avant |
| `root_squash` (**défaut**) | **root du client est transformé en utilisateur anonyme** (`nobody`, UID 65534) |
| `no_root_squash` | root du client **garde ses droits root** sur l'export (dangereux) |
| `all_squash` | **tous** les utilisateurs sont transformés en anonyme |
| `anonuid` / `anongid` | UID/GID de l'utilisateur anonyme |

### 1.3 Sécurité NFS classique : UID/GID

NFS (authentification `AUTH_SYS`) **fait confiance au client** et identifie les utilisateurs par leur **UID/GID numériques**, **pas par leur nom**. Conséquences :
- deux comptes de **même UID** sur deux machines sont considérés comme le **même utilisateur** pour NFS ;
- d'où `root_squash`, pour éviter que root d'une machine cliente soit root sur le serveur.

### 1.4 Permissions `1777` (sticky bit)

`chmod 1777` = `rwxrwxrwt` : tout le monde peut lire/écrire, et le **sticky bit** (`t`) empêche un utilisateur de **supprimer/renommer les fichiers des autres** (comme `/tmp`).

---

## 2. Plan du TP

**Partie 1**
- Serveur : installer, créer le dossier, éditer `/etc/exports`, démarrer, exporter
- Client : créer les points de montage, monter, (optionnel) `fstab`, vérifier les droits

**Partie 2** (UID, `root_squash`)
1. Créer `user1` (UID 600) des deux côtés
2. Créer un fichier côté client
3. Le supprimer côté serveur
4. Fichier créé par root client : quel propriétaire ?
5. `no_root_squash` : effet et risques

---

## 3. Pratique

### Préparation (serveur **et** client)

```bash
sudo dnf install -y nfs-utils
sudo systemctl enable --now rpcbind
sudo systemctl status rpcbind
```

- `nfs-utils` : outils et services NFS (serveur et client).
- `rpcbind` : service d'annuaire RPC, requis par NFS.

### Partie 1 – Côté serveur

#### Étape 1 – Dossier à partager

```bash
sudo mkdir -p /nfs/share
sudo chmod 1777 /nfs/share
```

- `mkdir -p` : crée `/nfs` puis `/nfs/share` ; `chmod 1777` : voir théorie (accès à tous + sticky bit).

#### Étape 2 – `/etc/exports`

```bash
sudo nano /etc/exports
```

```
/nfs/share 192.168.18.0/24(ro)
/tmp       192.168.18.4(rw)
```

Syntaxe : `<chemin>  <client>(<options>)`

- `/nfs/share` : chemin **complet** du dossier exporté.
- `192.168.18.0/24` : **tout le réseau** peut y accéder (adapter à ton réseau, voir `ip a`) ; on peut aussi mettre une IP, une liste séparée par des espaces ou un nom d'hôte.
- `ro` : lecture seule ; `rw` : lecture/écriture (ici `/tmp` n'est accessible qu'au client `192.168.18.4`).
- ⚠️ **Pas d'espace** entre le client et la parenthèse : `192.168.18.4 (rw)` signifierait « tout le monde en `rw` ».

#### Étape 3 – Démarrer les services

```bash
sudo systemctl restart rpcbind
sudo systemctl enable --now nfs-server
```

- L'énoncé ne redémarre que `rpcbind` ; il faut aussi **démarrer `nfs-server`** pour que les exports soient servis.

Pare-feu :

```bash
sudo firewall-cmd --permanent --add-service={nfs,rpc-bind,mountd}
sudo firewall-cmd --reload
```

- Ouvre les services NFS, RPC et mountd ; (ou désactiver temporairement `firewalld` en labo : `sudo systemctl stop firewalld`).

#### Étape 4 – Activer et vérifier les exports

```bash
sudo exportfs -a
sudo exportfs
sudo exportfs -v
```

- `exportfs -a` : **exporte** toutes les entrées de `/etc/exports`.
- `exportfs` : affiche la liste des exports actifs ; `-v` ajoute les options effectives (`sync`, `root_squash`…).
- Après toute modification d'`/etc/exports` : `sudo exportfs -ra` (*re-export all*).

Attendu :

```
/tmp          192.168.18.4
/nfs/share    192.168.18.0/24
```

### Partie 1 – Côté client

#### Étape 1 – Points de montage

```bash
sudo mkdir -p /nfs/mount-pt1
sudo mount -t nfs 192.168.18.6:/nfs/share /nfs/mount-pt1

sudo mkdir -p /nfs/mount-pt2
sudo mount -t nfs 192.168.18.6:/tmp /nfs/mount-pt2
```

- `mkdir -p` : crée les points de montage (`-p` car `/nfs` n'existe pas sur le client).
- `mount -t nfs <serveur>:<export> <point_de_montage>` : monte l'export distant.
- ⚠️ L'énoncé écrit `192.168.18.6 :/nfs/share` avec un espace avant `:` ; il ne faut **pas** d'espace.

Vérifier :

```bash
df -h | grep nfs
mount | grep nfs
showmount -e 192.168.18.6
```

- `df -h` : montages avec tailles ; `mount | grep nfs` : montages NFS actifs ; `showmount -e <serveur>` : liste les exports du serveur.

**Montage permanent (optionnel) : `/etc/fstab`**

```
192.168.18.6:/nfs/share  /nfs/mount-pt1  nfs  defaults  0 0
192.168.18.6:/tmp        /nfs/mount-pt2  nfs  defaults  0 0
```

Champs : *source* · *point de montage* · *type* · *options* · *dump* · *fsck*.
- L'énoncé met `nfs 0 0` : il manque le champ **options** ; mettre `defaults`.
- Tester sans redémarrer : `sudo mount -a`.

#### Étape 2 – Vérifier lecture / écriture

```bash
touch /nfs/mount-pt1/test       # → échec : « Read-only file system » (export ro)
touch /nfs/mount-pt2/test       # → réussi (export rw)
ls -l /nfs/mount-pt2
```

- `/nfs/share` est exporté en `ro` → **lecture seule** ; `/tmp` est exporté en `rw` → **lecture/écriture**.

### Partie 2 – UID et `root_squash`

#### 1. Créer `user1` (UID 600) sur le serveur **et** le client

```bash
sudo useradd -u 600 user1
id user1
```

- `-u 600` : force l'UID (identique des deux côtés, sinon NFS les considère comme deux utilisateurs différents).

#### 2. Fichier créé côté client par `user1`

```bash
su - user1
touch /nfs/mount-pt2/file-user1-client
ls -l /nfs/mount-pt2/file-user1-client
```

#### 3. Côté serveur : `user1` essaie de supprimer le fichier

```bash
su - user1
rm /tmp/file-user1-client
```

**Résultat :** la suppression **réussit**. Explication : NFS compare les **UID** : `user1` a l'UID 600 sur les deux machines, donc il est **propriétaire** du fichier côté serveur (le sticky bit de `/tmp` autorise le propriétaire à supprimer). Si l'UID différait, la suppression serait refusée (*Operation not permitted*).

#### 4. Fichier créé par **root** du client

```bash
# sur le client, en root
touch /nfs/mount-pt2/file-root-client
ls -l /nfs/mount-pt2/file-root-client
```

**Propriétaire :** `nobody` (UID 65534, parfois `nfsnobody`), pas `root`.

**Explication :** par défaut, l'export a **`root_squash`** : le serveur transforme les requêtes de **root du client** en utilisateur anonyme, afin que root d'une machine cliente ne devienne pas root sur le serveur.

#### 5. Reconnaître root comme root : `no_root_squash`

Sur le serveur, `/etc/exports` :

```
/tmp  192.168.18.4(rw,no_root_squash)
```

```bash
sudo exportfs -ra
sudo exportfs -v
```

Sur le client, remonter et refaire le test :

```bash
sudo umount /nfs/mount-pt2
sudo mount -t nfs 192.168.18.6:/tmp /nfs/mount-pt2
sudo touch /nfs/mount-pt2/file-root-client2
ls -l /nfs/mount-pt2/file-root-client2      # → propriétaire root
```

**Répercussions (sécurité) 🔒 :** root du **client** a désormais les pleins pouvoirs **sur les fichiers de l'export** : il peut lire/modifier/supprimer n'importe quel fichier, créer des binaires **setuid root**, modifier des fichiers sensibles… Si le client est compromis, l'export l'est aussi. À réserver à des clients **totalement de confiance** (ex. sauvegarde), et à éviter sur des répertoires système.

---

## 4. Dépannage

| Symptôme | Cause | Solution |
|----------|-------|----------|
| `mount.nfs: access denied by server` | client non autorisé dans `/etc/exports` | corriger l'IP, `exportfs -ra` |
| `mount.nfs: Connection timed out` | pare-feu / service arrêté | `firewall-cmd`, `systemctl status nfs-server` |
| `No such file or directory` au montage | point de montage inexistant | `mkdir -p` |
| `Read-only file system` | export en `ro` | passer en `rw` |
| `Permission denied` en écriture | droits Unix du dossier / squash | `chmod`, vérifier UID |
| Propriétaire `nobody` | `root_squash` | normal ; `no_root_squash` seulement si nécessaire |
| Diagnostic | | `showmount -e`, `rpcinfo -p <serveur>`, `journalctl -u nfs-server` |

---

## 5. Questions de révision

1. Quel est le rôle de `/etc/exports` ? Quelle commande applique ses modifications ?
2. Différence entre `ro` et `rw`, `sync` et `async` ?
3. Comment NFS identifie-t-il les utilisateurs ? Pourquoi les UID doivent-ils concorder ?
4. Que fait `root_squash` ? Pourquoi est-il activé par défaut ?
5. Quel est le risque de `no_root_squash` ?
6. À quoi sert le sticky bit (`1777`) ?
7. Comment monter automatiquement un partage NFS au démarrage ?
