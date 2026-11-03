# TP 08 – Atelier Samba

**Environnement :** 2 VM AlmaLinux 8 (serveur et client) + machine Windows physique · **Serveur :** `192.168.18.6` · **Réseau :** `192.168.18.0/24`

---

## 1. Théorie

### 1.1 SMB/CIFS et Samba

- **SMB** (*Server Message Block*, ancien nom **CIFS**) : protocole de **partage de fichiers et d'imprimantes** utilisé par Windows.
- **Samba** : implémentation libre de SMB pour Linux. Elle permet à Linux de **partager** des dossiers avec des clients Windows (et Linux) et de **se connecter** à des partages Windows.

### 1.2 Composants

| Élément | Rôle |
|---------|------|
| **`smbd`** (service `smb`) | partage de fichiers/imprimantes, authentification (ports **139** et **445** TCP) |
| **`nmbd`** (service `nmb`) | résolution de noms NetBIOS, navigation réseau (ports **137**, **138** UDP) |
| `/etc/samba/smb.conf` | fichier de configuration |
| `smbpasswd` | gère la **base de mots de passe Samba** (distincte de `/etc/shadow`) |
| `testparm` | vérifie la syntaxe de `smb.conf` |
| `smbclient` | client en ligne de commande |

### 1.3 Points clés

- Un utilisateur Samba doit **exister comme utilisateur Linux** *et* être ajouté à la base Samba (`smbpasswd -a`) ; son mot de passe Samba peut différer du mot de passe Linux.
- **`security = user`** : authentification par identifiant/mot de passe.
- **`workgroup`** : nom du groupe de travail Windows (visible avec `net config workstation`).
- Sections spéciales de `smb.conf` : `[global]` (réglages généraux), `[homes]` (répertoires personnels), `[printers]` (imprimantes) ; toute autre section `[nom]` = un partage.
- **SELinux** et **pare-feu** peuvent bloquer Samba.

---

## 2. Plan du TP

1. Installer Samba (serveur) ; le client existe sur la VM cliente
2. Désactiver SELinux (de façon permanente) et vérifier
3. Gérer le pare-feu
4. Créer le répertoire partagé
5. Créer les utilisateurs Linux et Samba
6. Configurer `smb.conf` (partage, `[homes]`, `[global]`)
7. Redémarrer les services
8. Vérifier la configuration
9. Tester depuis Windows et depuis Linux

---

## 3. Pratique

### Étape 0 – Installation

```bash
# Serveur
sudo dnf install -y samba samba-client
# Client
sudo dnf install -y samba-client cifs-utils
```

- `samba` : serveur ; `samba-client` : `smbclient`, `smbpasswd`… ; `cifs-utils` : permet de **monter** un partage avec `mount -t cifs`.

### Étape 1 – Désactiver SELinux (permanent)

```bash
sudo nano /etc/selinux/config
```

```
SELINUX=disabled
```

```bash
sudo reboot
sestatus
```

- Le fichier `/etc/selinux/config` fixe l'état au démarrage (`enforcing`, `permissive`, `disabled`) ; **un redémarrage est obligatoire**.
- `sestatus` : doit afficher `SELinux status: disabled`.

🔒 **Alternative recommandée** (garder SELinux actif) :

```bash
sudo setsebool -P samba_enable_home_dirs on
sudo chcon -R -t samba_share_t /samba/share
```

- `samba_enable_home_dirs` : autorise le partage des homes ; `samba_share_t` : étiquette SELinux pour un dossier partagé par Samba.

### Étape 2 – Pare-feu

```bash
sudo systemctl stop firewalld
```

Ou, mieux, laisser le pare-feu et ouvrir Samba :

```bash
sudo firewall-cmd --permanent --add-service=samba
sudo firewall-cmd --reload
```

- Ports concernés : **137, 138** (UDP), **139, 445** (TCP). Le service prédéfini `samba` les ouvre tous. (L'énoncé cite 137–139 ; **445** est le port SMB moderne.)

### Étape 3 – Répertoire à partager

```bash
sudo mkdir -p /samba/share
```

⚠️ Le dossier appartient à `root` (755) : les utilisateurs Samba ne pourront **pas y écrire**. Prévoir un groupe commun :

```bash
sudo groupadd sambashare
sudo chgrp sambashare /samba/share
sudo chmod 2770 /samba/share
```

- `chmod 2770` : `rwx` pour propriétaire et groupe, **rien** pour les autres ; le **setgid** (`2`) fait hériter aux nouveaux fichiers du groupe `sambashare`.

### Étape 4 – Utilisateurs

```bash
sudo useradd samba-user1
sudo useradd samba-user2
sudo usermod -aG sambashare samba-user1
sudo usermod -aG sambashare samba-user2
sudo smbpasswd -a samba-user1
sudo smbpasswd -a samba-user2
```

- `useradd` : crée les comptes Linux ; `usermod -aG` : les **ajoute** (*append*) au groupe `sambashare`.
- `smbpasswd -a <user>` : **ajoute** l'utilisateur à la base Samba et demande son **mot de passe Samba** (deux fois).
- Plus tard : `smbpasswd <user>` change le mot de passe ; `pdbedit -L` liste les utilisateurs Samba.

### Étape 5 – Configuration `/etc/samba/smb.conf`

```bash
sudo nano /etc/samba/smb.conf
```

**5.2 – Partage à ajouter en fin de fichier**

```ini
[samba-share]
   comment = Partage avec samba
   path = /samba/share
   valid users = samba-user1, samba-user2
   writable = yes
   browseable = yes
   create mask = 0660
```

- `[samba-share]` : nom du partage (ce que voit le client).
- `path` : dossier local partagé.
- `valid users` : seuls ces utilisateurs sont autorisés.
- `writable = yes` : écriture autorisée (équivaut à `read only = no`).
- `browseable = yes` : visible dans la liste des partages.
- `create mask = 0660` : droits des fichiers créés = `rw-rw----` (propriétaire + groupe).

**5.3 – Répertoires personnels `[homes]`**

```ini
[homes]
   comment = Home Directory Access
   browseable = no
   valid users = %S
   writable = yes
   create mask = 0660
```

- `[homes]` : partage **automatique** du répertoire personnel de chaque utilisateur.
- `valid users = %S` : `%S` est remplacé par le **nom du partage** (= le nom de l'utilisateur) → chacun n'accède qu'à son propre home.
- `browseable = no` : le partage générique « homes » n'est pas listé (l'utilisateur voit son nom).

**5.4 – Section `[global]`**

```ini
[global]
   security = user
   interfaces = eth0 192.168.18.0/24
   hosts allow = 127. 192.168.18.
   workgroup = WORKGROUP
```

- `security = user` : authentification par utilisateur/mot de passe.
- `interfaces` : interface réseau et réseau sur lesquels Samba **écoute** (adapter `eth0` au nom réel : `ip a`, ex. `ens33`).
- `hosts allow = 127. 192.168.18.` : machines **autorisées** (préfixes d'adresses : localhost et le réseau `192.168.18.x`).
- `workgroup` : nom du groupe de travail Windows (obtenu par `net config workstation` sous Windows).

### Étape 6 – Redémarrer les services

```bash
sudo systemctl enable --now smb nmb
sudo systemctl restart smb nmb
```

- `enable --now` : démarre et active au boot ; `restart` : recharge la nouvelle configuration. Les **deux** services (`smb` et `nmb`) sont à redémarrer.

### Étape 7 – Vérifier la configuration

```bash
testparm
sudo systemctl status smb
```

- `testparm` : analyse `smb.conf`, signale les erreurs et affiche les sections traitées (`homes`, `printers`, `samba-share`) ; terminer par `Loaded services file OK.` Appuyer sur Entrée pour voir toutes les définitions.
- `systemctl status smb` : le service doit être *active (running)*. (L'énoncé écrit `# /systemctl status smb` : le `/` est une coquille.)

### Étape 8 – Tester

**Depuis Windows** : dans l'explorateur ou « Exécuter » (Win+R) :

```
\\192.168.18.6
```

- On doit voir `samba-share` et le dossier personnel de l'utilisateur ; Windows demande l'identifiant (`samba-user1`) et le mot de passe **Samba**.
- Créer un fichier dans `samba-share`, puis côté serveur :

```bash
ls -l /samba/share/
```

- Les fichiers apparaissent, avec leur propriétaire et leurs droits (`create mask 0660`).

**Depuis Linux (VM cliente)**

```bash
smbclient //192.168.18.6/samba-share -U samba-user1
smbclient //192.168.18.6/homes -U samba-user1
smbclient //192.168.18.6/samba-share -U samba-user2
smbclient //192.168.18.6/homes -U samba-user2
```

- `smbclient //<serveur>/<partage> -U <utilisateur>` : ouvre une session interactive de type FTP.

Commandes dans `smb: \>` :

| Commande | Action |
|----------|--------|
| `ls` | lister le dossier distant |
| `get fichier` / `put fichier` | télécharger / envoyer |
| `mkdir`, `rm`, `cd`, `pwd` | gestion de dossiers/fichiers |
| `exit` | quitter |

Lister les partages d'un serveur : `smbclient -L //192.168.18.6 -U samba-user1`.

**Monter le partage de façon permanente (bonus)**

```bash
sudo mkdir -p /mnt/samba
sudo mount -t cifs //192.168.18.6/samba-share /mnt/samba -o username=samba-user1
```

- `-t cifs` : type de système de fichiers SMB ; `-o username=` : compte à utiliser (le mot de passe est demandé).

---

## 4. Dépannage

| Symptôme | Cause | Solution |
|----------|-------|----------|
| `NT_STATUS_LOGON_FAILURE` | mauvais mot de passe / utilisateur absent de la base Samba | `smbpasswd -a`, `pdbedit -L` |
| `NT_STATUS_ACCESS_DENIED` / écriture impossible | droits Unix du dossier ou `valid users` | `chmod 2770`, groupe `sambashare` |
| Partage invisible depuis Windows | pare-feu / `hosts allow` / `interfaces` | ouvrir `samba`, corriger `smb.conf` |
| `Connection refused` | `smb` arrêté | `systemctl status smb` |
| Homes inaccessibles | SELinux | `setsebool -P samba_enable_home_dirs on` |
| Configuration non prise en compte | pas de redémarrage | `systemctl restart smb nmb` |
| Logs | | `/var/log/samba/log.<machine>`, `journalctl -u smb` |

---

## 5. Questions de révision

1. Quels sont les rôles de `smbd` et `nmbd` ? Quels ports utilisent-ils ?
2. Pourquoi faut-il créer l'utilisateur **Linux** puis `smbpasswd -a` ?
3. Que signifient `valid users`, `writable`, `browseable`, `create mask` ?
4. Quel est le rôle de la section `[homes]` et de `%S` ?
5. Que contrôlent `interfaces` et `hosts allow` ?
6. À quoi sert `testparm` ?
7. Comment tester un partage depuis Windows et depuis Linux ?
8. Pourquoi SELinux peut-il bloquer Samba et comment l'éviter sans le désactiver ?
