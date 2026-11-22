# TP – Atelier iSCSI : target AlmaLinux, initiators Ubuntu et Windows

**Distribution serveur :** AlmaLinux 8 (`yum`/`dnf`) · **Client Linux :** Ubuntu (`apt`) · **Client Windows :** physique ou virtuel · **Paquets :** `targetcli` (serveur) / `open-iscsi` (Ubuntu) ou `iscsi-initiator-utils` (AlmaLinux)

> ⚠️ Les réponses ci-dessous sont **mon interprétation** de l'énoncé : à confronter avec ton cours et à tester sur ta VM. Quelques captures de l'énoncé n'étaient pas lisibles (ligne à modifier dans `iscsid.conf`, etc.) : points marqués *à vérifier*.

---

## 1. Théorie

### 1.1 Le protocole iSCSI

**iSCSI** (*Internet Small Computer System Interface*) transporte des commandes SCSI **sur TCP/IP**. Il permet à un client d'utiliser un **disque distant comme un disque local** (accès **bloc**, pas fichier).

- Port : **TCP 3260**.
- Diffère de Samba/NFS : ici le client reçoit un **périphérique bloc brut** (`/dev/sdX`) qu'il **partitionne, formate et monte** lui-même.

### 1.2 Vocabulaire

| Terme | Signification |
|-------|---------------|
| **Target** | le **serveur** qui expose le stockage |
| **Initiator** | le **client** qui se connecte au stockage |
| **IQN** | *iSCSI Qualified Name*, identifiant unique d'un target ou initiator |
| **Backstore** | objet de stockage réel (disque, fichier, bloc) qui sert de support |
| **LUN** | *Logical Unit Number* : unité logique présentée à l'initiator (le « disque » vu par le client) |
| **TPG** | *Target Portal Group* : groupe de portails (IP:port) d'un target |
| **ACL** | liste des initiators **autorisés** à accéder au target |
| **Portal** | adresse `IP:port` du target (3260) |
| **Discovery** | découverte des targets disponibles sur un portal (`sendtargets`) |

### 1.3 Format d'un IQN

```
iqn.2022-06.com.example:server1.target01
 │     │        │            └── nom unique choisi
 │     │        └── domaine inversé
 │     └── année-mois
 └── préfixe
```

### 1.4 Chaîne de configuration côté target

```
Disque réel (/dev/sdf) → Backstore (block) → Target (IQN) → LUN → ACL (IQN de l'initiator)
```

### 1.5 Étapes côté initiator

```
Installation → InitiatorName → Découverte → Connexion (login) → Nouveau disque → Partition → ext4 → Montage → fstab
```

---

## 2. Plan du TP

**Côté serveur (target)**
1. Installer `targetcli`
2. Ajouter un disque virtuel de 1 Go
3. Créer le backstore, la cible, le LUN, l'ACL
4. Ouvrir le port 3260
5. Démarrer le service `target`

**Côté client Linux (Ubuntu)**
1. Installer `open-iscsi`
2. Configurer `iscsid.conf` et `initiatorname.iscsi`
3. Découvrir et se connecter
4. Partitionner, formater, monter, `fstab`

**Côté client Windows**
1. Initiateur iSCSI → connexion → formater le disque

---

## 3. Pratique

### Partie A : Serveur (target AlmaLinux)

#### A.1 Installation

```bash
sudo yum -y install targetcli
```

#### A.2 Disque virtuel supplémentaire

Ajouter dans l'hyperviseur un **disque de 1 Go** à la VM serveur, puis l'identifier :

```bash
lsblk
```

- Dans l'énoncé il apparaît comme `/dev/sdf` : **adapter** au nom réel dans ta VM (`sdb`, `sdc`…).
- Ce disque n'est **ni partitionné ni monté** : il est exporté tel quel.

#### A.3 Configurer avec `targetcli`

```bash
sudo targetcli
```

**1. Backstore de type bloc**

```
/> cd /backstores/block/
/backstores/block> create name=disk1 dev=/dev/sdf
```

- Associe le nom `disk1` au disque physique `/dev/sdf`.

**2. Cible iSCSI (target)**

```
/> cd /iscsi/
/iscsi> create iqn.2022-06.com.example:server1.target01
```

- Crée aussi automatiquement le **TPG 1** et un portail par défaut `0.0.0.0:3260`.

**3. LUN**

```
/> cd /iscsi/iqn.2022-06.com.example:server1.target01/tpg1/luns
.../luns> create /backstores/block/disk1
```

- Relie le backstore `disk1` à la cible : c'est le « disque » vu par le client.

**4. ACL**

```
.../tpg1/luns> cd ../acls
.../tpg1/acls> create iqn.2022-06.com.example:node01.initiator01
```

- Seul l'initiator portant **cet IQN** sera autorisé.
- ⚠️ Cet IQN doit être **identique** à celui du fichier `initiatorname.iscsi` du client.

**5. Vérifier et sauvegarder**

```
/> ls
/> saveconfig
/> exit
```

- `ls` : affiche l'arborescence complète (backstores, iSCSI, TPG, ACL, LUN).
- `saveconfig` : écrit la configuration (`/etc/target/saveconfig.json`) ; `exit` l'enregistre aussi généralement automatiquement.

#### A.4 Pare-feu

```bash
sudo firewall-cmd --permanent --add-service=iscsi-target
sudo firewall-cmd --permanent --add-port=3260/tcp
sudo firewall-cmd --reload
```

- Ouvre le port **TCP 3260** (l'un ou l'autre suffit ; les deux sont donnés dans l'énoncé).

#### A.5 Démarrer le service

```bash
sudo systemctl restart target
sudo systemctl enable target
```

- Le service s'appelle **`target`** (pas `targetcli`).
- `enable` : relance la configuration iSCSI **au démarrage** (sinon elle est perdue après reboot).

---

### Partie B : Client Linux (Ubuntu, initiator)

#### B.1 Installation

```bash
sudo apt install open-iscsi
```

(Sur AlmaLinux : `sudo yum install -y iscsi-initiator-utils`.)

#### B.2 Configuration de `/etc/iscsi/iscsid.conf`

```bash
sudo nano /etc/iscsi/iscsid.conf
```

- Modifier la ligne indiquée dans l'énoncé (*à vérifier*) : en général le démarrage automatique des sessions :

```
node.startup = automatic
```

**Option : protéger par login/mot de passe (CHAP)**, décommenter :

```
node.session.auth.authmethod = CHAP
node.session.auth.username = username
node.session.auth.password = password
```

- `username` et `password` doivent être **définis côté target** (dans l'ACL).

#### B.3 Nom de l'initiator

```bash
sudo nano /etc/iscsi/initiatorname.iscsi
```

```
InitiatorName=iqn.2022-06.com.example:node01.initiator01
```

- Doit **correspondre à l'ACL** créée sur le target.

```bash
sudo systemctl restart iscsid.service
```

#### B.4 Interface et découverte

```bash
sudo iscsiadm -m iface -I iscsi01 --op=new
sudo iscsiadm -m discovery -I iscsi01 --op=new --op=del --type sendtargets --portal <IP_serveur>:3260
```

| Option | Rôle |
|--------|------|
| `-m iface -I iscsi01 --op=new` | crée l'interface iSCSI `iscsi01` |
| `-m discovery` | mode découverte |
| `--type sendtargets` | demande au portal la liste de ses targets |
| `--portal IP:3260` | adresse du target |

- Résultat attendu : `IP:3260,1 iqn.2022-06.com.example:server1.target01`.

#### B.5 Connexion automatique

```bash
sudo iscsiadm -m node --op=update -n node.conn[0].startup -v automatic
sudo iscsiadm -m node --op=update -n node.startup -v automatic
```

```bash
sudo systemctl enable open-iscsi
sudo systemctl restart iscsid.service
```

#### B.6 Ouvrir la session

```bash
sudo iscsiadm -m node --loginall=automatic
```

Vérifications :

```bash
sudo iscsiadm -m session      # sessions ouvertes
lsblk                         # un nouveau disque apparaît (ex. sdc)
lsblk --scsi                  # idem, vue SCSI
```

#### B.7 Partition, système de fichiers, montage

```bash
sudo gdisk /dev/sdc           # n (nouvelle partition) → w (écrire)
lsblk                         # vérifier sdc1
sudo mkfs.ext4 /dev/sdc1
sudo mkdir /mnt/iscsi
sudo mount /dev/sdc1 /mnt/iscsi
```

- `gdisk` : partitionneur GPT ; dans l'énoncé le nouveau disque est `/dev/sdc` (*adapter selon `lsblk`*).
- `mkfs.ext4` : crée le système de fichiers.

#### B.8 Montage permanent : `/etc/fstab`

```
/dev/sdc1   /mnt/iscsi   ext4   _netdev   0   0
```

- **`_netdev`** : indique que le disque est **sur le réseau** → monté **après** le démarrage du réseau (sinon blocage/erreur au boot).
- 💡 Plus fiable : utiliser l'**UUID** (`blkid`) au lieu de `/dev/sdc1`, car le nom du disque peut changer.
- Tester : `sudo mount -a`.

---

### Partie C : Client Windows

1. Ouvrir **Initiateur iSCSI** (outil inclus dans Windows).
2. Onglet **Cible** : saisir l'IP du serveur → **Connexion rapide**.
3. Sélectionner la cible `iqn.2022-06.com.example:server1.target01` → **Se connecter**.
4. Ouvrir la **Gestion des disques** : un **nouveau disque** apparaît → l'initialiser, créer un volume, le **formater** (NTFS) et l'utiliser.

> ⚠️ Pour que Windows soit autorisé, il faut **ajouter son IQN** (visible dans l'onglet *Configuration* de l'initiateur) comme **ACL** dans targetcli.

---

## 4. Dépannage

| Symptôme | Cause probable | Solution |
|----------|----------------|----------|
| Découverte impossible | pare-feu / service `target` arrêté | ouvrir 3260, `systemctl status target` |
| Découverte OK mais login refusé | IQN client ≠ ACL | aligner `initiatorname.iscsi` et l'ACL |
| Login refusé avec CHAP | mauvais login/mot de passe | vérifier côté ACL et `iscsid.conf` |
| Pas de nouveau disque après login | session non ouverte | `iscsiadm -m session`, `--loginall=automatic` |
| Configuration perdue après reboot | `saveconfig` / `enable target` oubliés | `saveconfig`, `systemctl enable target` |
| Blocage au boot (client) | `fstab` sans `_netdev` | ajouter `_netdev` |
| Nom du disque change (`sdc` → `sdd`) | ordre de détection | utiliser l'**UUID** dans `fstab` |
| Logs | | `journalctl -u target`, `journalctl -u iscsid` |

---

## 5. Questions de révision

1. Qu'est-ce qu'iSCSI ? Quelle différence avec un partage de fichiers (NFS/Samba) ?
2. Que sont un **target** et un **initiator** ? Quel est le port utilisé ?
3. Qu'est-ce qu'un **IQN** ? Donner un exemple.
4. Rôle d'un **backstore**, d'un **LUN**, d'une **ACL** ?
5. Dans quel ordre configure-t-on backstore, target, LUN, ACL ?
6. Pourquoi l'IQN de `initiatorname.iscsi` doit-il correspondre à l'ACL ?
7. Que fait la commande `iscsiadm -m discovery --type sendtargets` ?
8. Comment activer la connexion automatique au démarrage ?
9. Pourquoi l'option **`_netdev`** dans `/etc/fstab` ?
10. Comment protéger l'accès par login/mot de passe (CHAP) ?
11. Comment voir le nouveau disque apparu côté client ?
12. Comment se connecter depuis Windows ?
