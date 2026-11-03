# TP 06 – Atelier FTP : vsftpd et Pure-FTPd

**Distribution :** AlmaLinux 8 (`yum`/`dnf`) · **Client distant :** machine Windows ou second Linux

> ⚠️ L'énoncé de cet atelier ne contient que des **consignes** (pas de solution). Les réponses ci-dessous sont **mon interprétation** de ces consignes : à confronter avec ton cours et à tester sur ta VM.

---

## 1. Théorie

### 1.1 Le protocole FTP

**FTP** (*File Transfer Protocol*) permet de transférer des fichiers entre un client et un serveur.

- **Deux connexions TCP** :
  - **Contrôle** : port **21** (commandes `USER`, `PASS`, `LIST`, `RETR`…).
  - **Données** : transfert des fichiers/listes.
- **Mode actif** : le serveur ouvre la connexion de données depuis son port **20** vers le client (souvent bloqué par les pare-feu côté client).
- **Mode passif** : le client ouvre aussi la connexion de données vers un **port haut** annoncé par le serveur (`pasv_min_port`–`pasv_max_port`) → à ouvrir dans le pare-feu.
- 🔒 FTP envoie **identifiants et données en clair**. Alternatives : **FTPS** (FTP + TLS) ou **SFTP** (via SSH).

### 1.2 Types d'accès

- **Utilisateurs locaux** : comptes Linux (`/etc/passwd`), authentification PAM, arrivée dans `~`.
- **Anonyme** : login `anonymous` (ou `ftp`), mot de passe = e-mail quelconque ; accès à un dossier public, en général **lecture seule**.

### 1.3 Chroot (« emprisonner »)

Par défaut, un utilisateur FTP local peut parcourir **tout le système de fichiers** (`cd /`, `ls /etc`). Le **chroot** enferme l'utilisateur dans son répertoire personnel, qui devient sa racine `/`.

### 1.4 Logiciels

| | **vsftpd** | **Pure-FTPd** |
|---|-----------|--------------|
| Nom | *Very Secure FTP Daemon* | serveur FTP sécurisé et simple |
| Configuration | `/etc/vsftpd/vsftpd.conf` (un seul fichier) | `/etc/pure-ftpd/pure-ftpd.conf` ou options en ligne de commande |
| Utilisé par | Red Hat et de nombreuses entreprises | hébergeurs, installations légères |

### 1.5 Permissions de répertoire (rappel)

Sur un **dossier** : `r` = lister le contenu, `w` = créer/supprimer des entrées, `x` = traverser (`cd`). Donc `300` (`-wx------`) = on peut **écrire et créer** mais **pas lister** (« écrire sans voir »).

---

## 2. Plan du TP

**Partie A – vsftpd**
1. Installer vsftpd
2. Tester depuis une machine distante (config par défaut)
3. Installer un client FTP en ligne de commande
4. Naviguer avec `ftp` (comptes utilisateurs)
5. Emprisonner (chroot) les utilisateurs
6. Étudier/tester les directives
7. Partager `/ftp/share`

**Partie B – Pure-FTPd**
1. Créer le compte `ftp`
2. Verrouiller le compte, limiter les droits (écrire sans voir)
3. Lancer en mode anonyme uniquement
4. Lister/envoyer des fichiers avec `ftp`

---

## 3. Pratique

### Partie A : vsftpd

#### A.1 Installation

```bash
sudo yum install -y vsftpd
sudo systemctl enable --now vsftpd
sudo systemctl status vsftpd
```

- `yum install -y vsftpd` : installe le serveur.
- `enable --now` : démarre immédiatement **et** au boot.
- Ouvrir le pare-feu (sinon l'accès distant échoue) :

```bash
sudo firewall-cmd --permanent --add-service=ftp
sudo firewall-cmd --reload
```

- `--permanent` : règle conservée après redémarrage ; `--add-service=ftp` : ouvre le port 21 ; `--reload` : applique.

#### A.2 Test depuis Windows (sans modifier la configuration)

Dans la barre d'adresse de l'explorateur/navigateur : `ftp://<IP_du_serveur>`

**Que remarque-t-on ?** *(à vérifier sur ta VM)*
- La config par défaut a `anonymous_enable=NO` : le serveur **demande un identifiant**, l'accès anonyme est refusé.
- Avec un compte Linux valide (`local_enable=YES`), on arrive dans son **répertoire personnel**.
- Si rien ne répond : pare-feu (port 21) ou SELinux.

#### A.3 Client FTP en ligne de commande

```bash
sudo yum install -y ftp
```

#### A.4 Accéder et naviguer

```bash
ftp <IP_du_serveur>
```

Session type :

```
Name: user1
Password: ********
ftp> pwd
ftp> ls
ftp> cd /
ftp> ls
ftp> get fichier.txt
ftp> put local.txt
ftp> bye
```

- `pwd` : dossier courant distant ; `ls` : liste ; `cd` : change de dossier ; `get` : télécharge ; `put` : envoie ; `bye` : quitte.

**Que remarque-t-on ?**
- L'utilisateur peut faire `cd /` puis lister **tout le système** (`/etc`, `/var`…) dans la limite des droits Unix → **faille de confidentialité** : c'est la raison du chroot.

#### A.5 Emprisonner les utilisateurs (chroot)

```bash
sudo vi /etc/vsftpd/vsftpd.conf
```

Ajouter/décommenter :

```
chroot_local_user=YES
allow_writeable_chroot=YES
```

```bash
sudo systemctl restart vsftpd
```

- `chroot_local_user=YES` : **tous** les utilisateurs locaux sont enfermés dans leur home.
- `allow_writeable_chroot=YES` : depuis vsftpd 2.3.5, un chroot **inscriptible** est refusé (erreur `500 OOPS: vsftpd: refusing to run with writable root inside chroot()`) ; cette option l'autorise.
- Variante avec **liste d'exceptions** : `chroot_local_user=NO`, `chroot_list_enable=YES`, `chroot_list_file=/etc/vsftpd/chroot_list` → **seuls** les utilisateurs listés sont enfermés.

Test : reconnecter un utilisateur, faire `cd /` + `pwd` → il reste dans sa racine (`/`).

Si l'accès au home est refusé (SELinux) :

```bash
sudo setsebool -P ftp_home_dir on
```

- `-P` : réglage persistant ; `ftp_home_dir on` : autorise vsftpd à accéder aux répertoires personnels.

#### A.6 Principales directives de `vsftpd.conf` à tester

| Directive | Effet |
|-----------|-------|
| `anonymous_enable=YES/NO` | autorise l'accès anonyme |
| `local_enable=YES` | autorise les comptes locaux |
| `write_enable=YES` | autorise l'écriture (STOR, MKD, DELE…) |
| `local_umask=022` | masque des fichiers créés (644) |
| `anon_upload_enable=YES` | l'anonyme peut déposer (nécessite `write_enable`) |
| `anon_mkdir_write_enable=YES` | l'anonyme peut créer des dossiers |
| `anon_root=/ftp/share` | racine de l'anonyme |
| `local_root=/ftp/share` | racine des utilisateurs locaux |
| `dirmessage_enable=YES` | affiche `.message` en entrant dans un dossier |
| `xferlog_enable=YES` | journalise les transferts (`/var/log/xferlog`) |
| `ftpd_banner=Bienvenue` | bannière de connexion |
| `userlist_enable=YES`, `userlist_file`, `userlist_deny` | liste d'utilisateurs interdits (`deny=YES`) ou seuls autorisés (`deny=NO`) |
| `max_clients`, `max_per_ip` | limites de connexions |
| `local_max_rate`, `anon_max_rate` | débit maximal (octets/s) |
| `pasv_min_port`, `pasv_max_port` | plage de ports du mode passif |
| `listen=YES` | mode autonome (standalone) IPv4 |
| `ssl_enable=YES` | active FTPS |

Après chaque modification : `sudo systemctl restart vsftpd`, puis tester côté client.

#### A.7 Partager le dossier `/ftp/share`

```bash
sudo mkdir -p /ftp/share
sudo chmod 755 /ftp/share
echo "Bienvenue" | sudo tee /ftp/share/lisezmoi.txt
```

Dans `vsftpd.conf` (accès anonyme en **lecture seule** sur ce dossier) :

```
anonymous_enable=YES
anon_root=/ftp/share
anon_upload_enable=NO
```

SELinux (si activé) :

```bash
sudo chcon -R -t public_content_t /ftp/share
# ou, pour autoriser l'écriture :
sudo setsebool -P allow_ftpd_anon_write on
sudo chcon -R -t public_content_rw_t /ftp/share
```

- `chcon -t public_content_t` : étiquette SELinux « contenu public » lisible par vsftpd.
- `/ftp/share` doit être **lisible** par l'utilisateur `ftp` ; ne pas lui donner l'écriture sauf besoin.

```bash
sudo systemctl restart vsftpd
ftp localhost       # login : anonymous, mot de passe : n'importe lequel
```

- Pour que des **utilisateurs locaux** voient ce dossier, utiliser `local_root=/ftp/share`.

---

### Partie B : Pure-FTPd

```bash
sudo dnf install -y epel-release
sudo dnf install -y pure-ftpd
```

- Pure-FTPd est fourni par le dépôt **EPEL** (*Extra Packages for Enterprise Linux*). Son service s'appelle `pure-ftpd`.

> 💡 Si vsftpd tourne déjà, l'arrêter (`sudo systemctl stop vsftpd`) : les deux se disputent le **port 21**.

#### B.1 Compte utilisateur `ftp`

Pure-FTPd utilise le compte système **`ftp`** pour l'accès anonyme ; sa **home** est la racine de l'anonyme.

```bash
sudo useradd -d /home/ftp -s /sbin/nologin ftp       # (ignorer l'erreur s'il existe déjà)
sudo mkdir -p /home/ftp
sudo chown ftp:ftp /home/ftp
```

- `-d /home/ftp` : répertoire personnel = racine anonyme.
- `-s /sbin/nologin` : **aucun shell** → le compte ne peut pas ouvrir de session de connexion.

#### B.2 Verrouiller le compte et limiter les droits

```bash
sudo passwd -l ftp
sudo chmod 300 /home/ftp
ls -ld /home/ftp
```

- `passwd -l ftp` : **verrouille** le mot de passe (aucune connexion par mot de passe possible).
- `chmod 300` = `-wx------` : le propriétaire peut **écrire/créer** (`w`) et traverser (`x`) mais **pas lister** (pas de `r`) ; groupe et autres : **aucun droit**.
- Résultat attendu : `d-wx------ ftp ftp /home/ftp`.

#### B.3 Lancer en mode anonyme uniquement

Test rapide en ligne de commande :

```bash
sudo pure-ftpd -e &
```

- `-e` (`--anonymousonly`) : **n'accepte que** les connexions anonymes.
- Option complémentaire : `-M` (`--anonymouscancreatedirs`) : l'anonyme peut créer des dossiers ; `-i` : l'anonyme **ne peut pas** déposer.

Version permanente via le fichier de configuration :

```bash
sudo vi /etc/pure-ftpd/pure-ftpd.conf
```

```
AnonymousOnly      yes
NoAnonymous        no
```

```bash
sudo systemctl enable --now pure-ftpd
sudo firewall-cmd --permanent --add-service=ftp && sudo firewall-cmd --reload
```

#### B.4 Lister et envoyer des fichiers

```bash
echo "test" > test.txt
ftp localhost
```

```
Name: anonymous
Password: (n'importe quoi)
ftp> ls
ftp> put test.txt
ftp> ls
ftp> bye
```

**Résultat attendu :**
- `put test.txt` → **réussi** (écriture autorisée).
- `ls` → **vide ou refusé** (pas de droit `r` sur le dossier) : on peut déposer sans voir le contenu.
- Vérification côté serveur : `sudo ls -l /home/ftp` doit montrer `test.txt`.

---

## 4. Dépannage

| Symptôme | Cause | Solution |
|----------|-------|----------|
| Connexion impossible depuis un autre poste | pare-feu | `firewall-cmd --add-service=ftp` |
| `500 OOPS: refusing to run with writable root inside chroot()` | chroot inscriptible | `allow_writeable_chroot=YES` |
| `530 Login incorrect` | utilisateur dans `ftpusers` / mauvais mot de passe / `local_enable=NO` | vérifier `/etc/vsftpd/ftpusers`, `local_enable=YES` |
| `550 Permission denied` | droits Unix ou SELinux | `ls -ld`, `chcon`, `setsebool` |
| Listage bloqué après login (mode passif) | ports passifs fermés | `pasv_min_port/max_port` + pare-feu |
| Port 21 déjà utilisé | deux serveurs FTP | arrêter l'autre |
| Logs | | `/var/log/vsftpd.log`, `/var/log/xferlog`, `journalctl -u vsftpd` |

---

## 5. Questions de révision

1. Pourquoi FTP utilise-t-il deux connexions ? Différence actif/passif ?
2. Pourquoi FTP est-il considéré comme peu sûr ? Quelles alternatives ?
3. Qu'est-ce que le chroot et quel problème résout-il ?
4. Quelle permission Unix permet d'écrire dans un dossier sans pouvoir le lister ?
5. Que fait l'option `-e` de `pure-ftpd` ?
6. Comment limiter un utilisateur FTP à son répertoire personnel dans vsftpd ?
7. Pourquoi vsftpd refuse-t-il un chroot inscriptible par défaut ?
