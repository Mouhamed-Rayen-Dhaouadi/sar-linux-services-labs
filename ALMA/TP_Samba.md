# TP – Atelier Samba : partage de fichiers Linux ↔ Windows

**Distribution :** AlmaLinux 8 (`yum`/`dnf`) · **Machines :** 1 serveur AlmaLinux, 1 client AlmaLinux, 1 Windows physique · **Réseau :** 1 carte réseau

> ⚠️ Les réponses ci-dessous sont **mon interprétation** de l'énoncé : à confronter avec ton cours et à tester sur ta VM.

---

## 1. Théorie

### 1.1 Qu'est-ce que Samba ?

**Samba** est l'implémentation libre du protocole **SMB/CIFS** (celui des partages Windows). Il permet à un serveur Linux de partager des dossiers et imprimantes avec des clients **Windows** ou **Linux**.

### 1.2 Les services

| Service | Démon | Rôle | Ports |
|---------|-------|------|-------|
| `smb` | `smbd` | partage de fichiers/imprimantes, authentification | **TCP 445** (et 139) |
| `nmb` | `nmbd` | résolution de noms NetBIOS, navigation réseau | **UDP 137, 138** |

- Ports de l'énoncé : **137, 138, 139** (TCP et UDP) ; en pratique 445 est le port SMB moderne.

### 1.3 Utilisateurs Samba

- Un utilisateur Samba doit **d'abord exister comme utilisateur Linux** (`useradd`).
- Il a ensuite un **mot de passe Samba distinct**, stocké dans la base Samba (`smbpasswd -a`).
- Les droits réels sur les fichiers restent ceux du **système de fichiers Linux** (propriétaire, `rwx`) **en plus** des règles du partage.

### 1.4 Structure de `smb.conf`

| Section | Rôle |
|---------|------|
| `[global]` | paramètres généraux du serveur |
| `[homes]` | partage automatique du répertoire personnel de chaque utilisateur |
| `[printers]` | partage des imprimantes |
| `[nom-partage]` | un partage personnalisé |

### 1.5 SELinux et pare-feu

- Enjeu : SELinux et `firewalld` peuvent bloquer Samba. L'énoncé **désactive SELinux** et **arrête/ouvre** le pare-feu pour simplifier le TP.
- ⚠️ En production on ne désactive pas SELinux : on utilise les booléens/contextes (`samba_enable_home_dirs`, `samba_share_t`).

---

## 2. Plan du TP

1. Désactiver SELinux (permanent) et vérifier
2. Pare-feu : arrêter `firewalld` ou ouvrir Samba
3. Créer le répertoire à partager
4. Créer 2 utilisateurs + mots de passe Samba
5. Configurer `smb.conf` (partage, `[homes]`, `[global]`)
6. Redémarrer `smb` et `nmb`
7. Vérifier la configuration
8. Tester depuis Windows et depuis Linux

---

## 3. Pratique

### Étape 0 : Installation

Sur le **serveur** :

```bash
sudo yum install -y samba samba-client
```

- `samba` : le serveur de fichiers.
- Sur le **client** Linux : `smbclient` (le client Samba), à installer si absent : `sudo yum install -y samba-client`.

### Étape 1 : Désactiver SELinux de façon permanente

```bash
sudo nano /etc/sysconfig/selinux
```

Modifier la ligne :

```
SELINUX=disabled
```

```bash
sudo reboot
# après redémarrage :
sestatus
```

Résultat attendu : `SELinux status: disabled`.

- Le fichier de config ne prend effet qu'**au redémarrage** (d'où le `reboot`).
- Test temporaire sans reboot : `sudo setenforce 0` (passe en *permissive*, non permanent).

### Étape 2 : Pare-feu

**Option 1 : arrêter le pare-feu** (simple, pour le TP) :

```bash
sudo systemctl stop firewalld
```

**Option 2 : autoriser Samba** (recommandé) :

```bash
sudo firewall-cmd --permanent --add-service=samba
sudo firewall-cmd --reload
```

- `--add-service=samba` ouvre d'un coup les bons ports (137, 138, 139, 445).
- `--permanent` : conservé après redémarrage ; `--reload` : applique.

### Étape 3 : Créer le répertoire à partager

```bash
sudo mkdir -p /samba/share
```

- `-p` : crée aussi les dossiers parents (`/samba` puis `/samba/share`).
- Pour que les 2 utilisateurs puissent y **écrire**, prévoir les droits Unix, par exemple :

```bash
sudo chmod 777 /samba/share
# ou, plus propre : un groupe commun
sudo groupadd sambagrp
sudo usermod -aG sambagrp samba-user1
sudo usermod -aG sambagrp samba-user2
sudo chgrp sambagrp /samba/share
sudo chmod 2770 /samba/share
```

### Étape 4 : Créer les utilisateurs

**4.1 : Utilisateurs Linux**

```bash
sudo useradd samba-user1
sudo useradd samba-user2
```

**4.2 : Mots de passe Samba**

```bash
sudo smbpasswd -a samba-user1
sudo smbpasswd -a samba-user2
```

- `-a` : **ajoute** l'utilisateur dans la base Samba et demande son mot de passe.
- Ce mot de passe est **indépendant** du mot de passe Linux.
- Pour vérifier : `sudo pdbedit -L`.

### Étape 5 : Configurer `/etc/samba/smb.conf`

```bash
sudo nano /etc/samba/smb.conf
```

**5.2 : Ajouter à la fin du fichier le partage :**

```ini
[samba-share]
   comment = Partage avec samba
   path = /samba/share
   valid users = samba-user1, samba-user2
   writable = yes
   browseable = yes
   create mask = 0660
```

| Paramètre | Effet |
|-----------|-------|
| `comment` | description affichée aux clients |
| `path` | dossier réel partagé |
| `valid users` | **seuls** ces utilisateurs peuvent se connecter au partage |
| `writable = yes` | écriture autorisée (équivaut à `read only = no`) |
| `browseable = yes` | le partage est **visible** dans la liste du réseau |
| `create mask = 0660` | permissions des fichiers créés : `rw-rw----` |

**5.3 : Accès aux répertoires personnels : section `[homes]`**

```ini
[homes]
   comment = Home Directory Access
   browseable = no
   valid users = %S
   writable = yes
   create mask = 0660
```

- `%S` : nom du partage courant = nom de l'utilisateur connecté → chacun n'accède qu'à **son** home.
- `browseable = no` : le partage `homes` n'apparaît pas lui-même dans la liste (mais le dossier de l'utilisateur, si).

**5.4 : Section `[global]`**

```ini
[global]
   security = user
   interfaces = eth0 192.168.18.0/24
   hosts allow = 127. 192.168.18.
   workgroup = WORKGROUP
```

| Ligne | Rôle |
|-------|------|
| `security = user` | authentification **par utilisateur** (login + mot de passe) |
| `interfaces` | interface et réseau sur lesquels le serveur **écoute** |
| `hosts allow` | machines/réseaux **autorisés** à utiliser Samba (`127.` = localhost, `192.168.18.` = le réseau) |
| `workgroup` | nom du groupe de travail Windows (visible avec `net config workstation` sous Windows) |

> 💡 Adapter `eth0` (nom réel de l'interface : `ip a`) et `192.168.18.` (ton réseau).

### Étape 6 : Redémarrer les services

```bash
sudo systemctl restart smb
sudo systemctl restart nmb
sudo systemctl enable smb nmb     # démarrage au boot (conseillé)
```

### Étape 7 : Vérifier la configuration

```bash
testparm
sudo systemctl status smb
```

- `testparm` : **vérifie la syntaxe** de `smb.conf` et affiche la configuration effective (sections `[homes]`, `[printers]`, `[samba-share]` chargées → `Loaded services file OK`).
- Rôle attendu : `Server role: ROLE_STANDALONE`.
- `systemctl status smb` : le service doit être `active (running)`.

### Étape 8 : Tester

#### Depuis Windows

Dans l'explorateur (barre d'adresse) ou *Exécuter* (`Win+R`) :

```
\\192.168.18.6
```

- Saisir `samba-user1` / son mot de passe **Samba**.
- On voit les partages `samba-share` et le dossier personnel `samba-user1`.
- Créer un fichier depuis Windows, puis vérifier côté serveur :

```bash
ls -l /samba/share/
```

- Le propriétaire des fichiers créés est l'utilisateur Samba connecté, avec les droits `0660` (`create mask`).

#### Depuis Linux (client AlmaLinux)

```bash
smbclient //192.168.18.6/samba-share -U samba-user1
smbclient //192.168.18.6/homes       -U samba-user1
smbclient //192.168.18.6/samba-share -U samba-user2
smbclient //192.168.18.6/homes       -U samba-user2
```

Commandes dans l'invite `smb: \>` :

| Commande | Rôle |
|----------|------|
| `ls` | lister |
| `cd` | changer de dossier |
| `get fichier` | télécharger |
| `put fichier` | envoyer |
| `mkdir dossier` | créer un dossier |
| `exit` | quitter |

Lister les partages d'un serveur : `smbclient -L //192.168.18.6 -U samba-user1`.

Montage permanent possible (client) :

```bash
sudo mount -t cifs //192.168.18.6/samba-share /mnt -o username=samba-user1
```

(nécessite `cifs-utils`).

---

## 4. Dépannage

| Symptôme | Cause probable | Solution |
|----------|----------------|----------|
| Connexion impossible depuis un autre poste | pare-feu | `firewall-cmd --add-service=samba` ou stopper `firewalld` |
| `NT_STATUS_LOGON_FAILURE` | mauvais mot de passe / utilisateur absent de la base Samba | `smbpasswd -a user`, `pdbedit -L` |
| `NT_STATUS_ACCESS_DENIED` | utilisateur absent de `valid users` / droits Unix insuffisants | corriger `valid users`, `chmod`/`chown` |
| `NT_STATUS_HOST_UNREACHABLE` / blocage | `hosts allow` ou `interfaces` incorrect | corriger `smb.conf` |
| Modification non prise en compte | service non redémarré | `systemctl restart smb nmb` |
| Erreur de syntaxe | `smb.conf` invalide | `testparm` |
| Fichiers non modifiables | droits Unix du dossier ou `writable = no` | `ls -ld /samba/share`, `writable = yes` |
| Accès refusé alors que la config est bonne | SELinux | `sestatus`, `setenforce 0` ou `setsebool -P samba_enable_home_dirs on` |
| Logs | | `/var/log/samba/log.<machine>`, `journalctl -u smb` |

---

## 5. Questions de révision

1. Qu'est-ce que Samba et quel protocole implémente-t-il ?
2. Quelle est la différence entre les services `smb` et `nmb` ?
3. Pourquoi faut-il créer l'utilisateur avec `useradd` **avant** `smbpasswd -a` ?
4. Que fait `smbpasswd -a` ? Le mot de passe Samba est-il le même que celui de Linux ?
5. À quoi servent `valid users`, `writable`, `browseable` et `create mask = 0660` ?
6. Que signifie `%S` dans la section `[homes]` ?
7. Rôle des paramètres `interfaces`, `hosts allow` et `workgroup` de la section `[global]` ?
8. Comment vérifier la syntaxe de `smb.conf` ?
9. Quels ports faut-il ouvrir dans le pare-feu pour Samba ?
10. Comment tester le partage depuis Windows ? depuis Linux ?
11. Comment désactiver SELinux de façon permanente et le vérifier ?
