# TP 09 – Installer et configurer OpenLDAP sur Ubuntu

**Domaine :** `[nom].com` → base DN `dc=[nom],dc=com` · **Administrateur :** `cn=admin,dc=[nom],dc=com`

> Remplace `[nom]` par **ton nom de famille** et `[prenom]` par **ton prénom** (minuscules, sans accent). Les captures de l'énoncé utilisent `hana` comme exemple.

---

## 1. Théorie

### 1.1 Annuaire et LDAP

- Un **annuaire** est une base de données **hiérarchique**, optimisée pour la **lecture** (beaucoup de recherches, peu d'écritures) : utilisateurs, groupes, machines, contacts…
- **LDAP** (*Lightweight Directory Access Protocol*) est le protocole standard pour interroger et modifier un annuaire. Il est une version « allégée » de **X.500** (DAP).
- **OpenLDAP** est l'implémentation libre ; son serveur s'appelle **`slapd`** (*Stand-alone LDAP Daemon*).
- Usage typique : **authentification centralisée** (comptes Linux, Apache, Postfix, Samba, VPN…). Active Directory de Microsoft repose aussi sur LDAP.

### 1.2 Le modèle de données : le DIT

L'annuaire est un **arbre** appelé **DIT** (*Directory Information Tree*). Chaque nœud est une **entrée**, identifiée par un **DN** (*Distinguished Name*), qui se lit **de la feuille vers la racine** :

```
uid=[prenom],ou=People,dc=[nom],dc=com
└─ RDN ─────┘└──── parent ─────────────┘
```

- **RDN** (*Relative DN*) : la partie propre à l'entrée (`uid=[prenom]`).
- **DN** : RDN + DN du parent = chemin complet, **unique**.
- Une entrée est composée d'**attributs** (`nom: valeur`) ; ses **`objectClass`** définissent quels attributs sont **obligatoires** (MUST) ou **facultatifs** (MAY). L'ensemble des classes/attributs forme le **schéma**.

| Notion | Signification | Exemple |
|--------|---------------|---------|
| `dc` | *Domain Component* : composant du domaine (racine) | `dc=[nom],dc=com` |
| `ou` | *Organizational Unit* : conteneur logique | `ou=People`, `ou=Groups` |
| `cn` | *Common Name* : nom commun d'un objet/groupe | `cn=LDAP` |
| `uid` | *User ID* : identifiant de l'utilisateur | `uid=[prenom]` |
| `sn` / `givenName` | nom / prénom | |
| `uidNumber` / `gidNumber` | UID / GID numériques Unix | `11` |
| `homeDirectory`, `loginShell` | répertoire personnel, shell | `/home/[prenom]`, `/bin/bash` |
| `userPassword` | mot de passe | `{SSHA}…` |
| `memberUid` | membre d'un `posixGroup` | |

**Classes utilisées dans ce TP**

| objectClass | Rôle |
|-------------|------|
| `organizationalUnit` | conteneur (`People`, `Groups`, `Files`) |
| `posixGroup` | groupe Unix (nécessite `cn` + `gidNumber`) |
| `inetOrgPerson` | personne (nécessite `cn` et `sn`) |
| `posixAccount` | compte Unix (`uid`, `uidNumber`, `gidNumber`, `homeDirectory`) |
| `shadowAccount` | informations de mot de passe type `/etc/shadow` |
| `top` | classe racine de toutes les classes |

### 1.3 LDIF

**LDIF** (*LDAP Data Interchange Format*) est un format **texte** pour décrire des entrées à importer/modifier.

- Une entrée = un bloc de lignes `attribut: valeur` commençant par `dn:`.
- Les entrées sont séparées par **une ligne vide** (obligatoire).
- **Aucun espace parasite** en début/fin de ligne ; le **parent** d'une entrée doit exister **avant** elle (ordre du fichier important).

### 1.4 Opérations, ports et authentification

- Opérations : **bind** (s'authentifier), **search**, **add**, **modify**, **delete**, **unbind**.
- Ports : **389** (LDAP, extensible en TLS via StartTLS), **636** (LDAPS). Socket Unix locale : **`ldapi:///`**.
- Deux modes d'authentification utilisés :
  - **Simple** (`-x`) : DN + mot de passe (`cn=admin,dc=…`). 🔒 Mot de passe en clair sur le réseau si pas de TLS.
  - **SASL EXTERNAL** (`-Y EXTERNAL`) sur `ldapi:///` : on s'authentifie avec l'**identité Unix locale** (root) ; sert à administrer la configuration (`cn=config`).

### 1.5 Outils clients

| Outil | Rôle |
|-------|------|
| `ldapsearch` | rechercher |
| `ldapadd` | ajouter des entrées (LDIF) |
| `ldapmodify` | modifier |
| `ldapdelete` | supprimer |
| `ldapwhoami` | vérifier une identité/authentification |
| `slappasswd` | générer un mot de passe haché (`{SSHA}`) |

---

## 2. Plan du TP

1. Installer `slapd` et `ldap-utils`
2. Configuration initiale : arrêt du service, `dpkg-reconfigure slapd`, `ldap.conf`, redémarrage, vérification
3. Créer le DIT : OU principales → sous-groupes → utilisateurs
4. Tester : `ldapsearch` en local, puis client graphique JXplorer

**Arborescence cible**

```
dc=[nom],dc=com
├── ou=Files
├── ou=Groups
│   ├── ou=developers
│   └── ou=networkadmins
│       ├── cn=Appache
│       └── cn=LDAP
│           └── cn=[prenom]
└── ou=People
    ├── uid=[prenom]
    └── uid=ahmed
```

---

## 3. Pratique

### Étape 1 – Installation

```bash
sudo apt-get install slapd ldap-utils
```

- `sudo` : exécute en root ; `apt-get install` : installe depuis les dépôts.
- `slapd` : le serveur ; `ldap-utils` : les clients (`ldapsearch`, `ldapadd`…).
- Pendant l'installation, Ubuntu demande le **mot de passe admin** : saisis-en un (on le redéfinit à l'étape suivante).

### Étape 2 – Configuration initiale

#### 2.1 Arrêt du service

```bash
sudo systemctl stop slapd
```

- Arrête `slapd` avant de reconfigurer.

#### 2.2 Reconfiguration

```bash
sudo dpkg-reconfigure slapd
```

- `dpkg-reconfigure` : relance l'assistant de configuration d'un paquet Debian/Ubuntu.

| Question | Réponse |
|----------|---------|
| Omettre la configuration d'OpenLDAP ? | **Non** (on veut créer la base) |
| Nom de domaine DNS | `[nom].com` → génère le DN racine `dc=[nom],dc=com` |
| Nom de l'organisation | `[nom].com` |
| Mot de passe administrateur | ton mot de passe (compte `cn=admin`) ; à confirmer |
| Backend de base de données | `MDB` (valeur par défaut) |
| Supprimer la base quand `slapd` est purgé ? | au choix (Non conseillé) |
| Déplacer l'ancienne base ? | **Oui** (pour repartir d'une base propre) |

#### 2.3 Configuration du client local

```bash
sudo nano /etc/ldap/ldap.conf
```

```
BASE   dc=[nom],dc=com
URI    ldap://localhost:389
```

- `BASE` : **base DN par défaut** des recherches côté client (évite de taper `-b`).
- `URI` : adresse du serveur (`ldap://hôte:port`) ; évite de taper `-H`.
- Décommenter ces deux lignes (supprimer le `#`) et les adapter.

```bash
sudo systemctl restart slapd
sudo systemctl status slapd
```

- `restart` : relance avec la nouvelle configuration ; `status` : doit indiquer *active (running)*.

#### 2.4 Vérification de la configuration de base

```bash
sudo ldapsearch -Q -LLL -Y EXTERNAL -H ldapi:///
```

- `ldapsearch` : recherche dans l'annuaire.
- `-Q` : mode silencieux (*quiet*), masque les messages SASL.
- `-LLL` : sortie LDIF épurée (sans version ni commentaires).
- `-Y EXTERNAL` : SASL **EXTERNAL** = identité Unix locale (root grâce à `sudo`).
- `-H ldapi:///` : se connecte par la **socket Unix locale** (pas de réseau).

Résultat attendu : l'entrée racine

```
dn: dc=[nom],dc=com
objectClass: top
objectClass: dcObject
objectClass: organization
o: [nom].com
dc: [nom]
```

### Étape 3 – Création du DIT

#### 3.1 Unités organisationnelles : `base-groups.ldif`

```
dn: ou=People,dc=[nom],dc=com
objectClass: organizationalUnit
ou: People

dn: ou=Groups,dc=[nom],dc=com
objectClass: organizationalUnit
ou: Groups

dn: ou=Files,dc=[nom],dc=com
objectClass: organizationalUnit
ou: Files
```

```bash
sudo ldapadd -x -D "cn=admin,dc=[nom],dc=com" -W -f base-groups.ldif
```

- `ldapadd` : ajoute les entrées décrites dans un fichier LDIF.
- `-x` : authentification **simple** (au lieu de SASL).
- `-D "cn=admin,…"` : **Bind DN**, l'identité utilisée pour se connecter (l'admin de l'annuaire).
- `-W` : demande le mot de passe **de façon interactive** (saisie masquée).
- `-f base-groups.ldif` : fichier source.
- Réponse attendue : `adding new entry "ou=People,dc=[nom],dc=com"` (×3).

#### 3.2 Sous-groupes : `add-groups.ldif`

```
dn: ou=networkadmins,ou=Groups,dc=[nom],dc=com
objectClass: organizationalUnit
ou: networkadmins

dn: ou=developers,ou=Groups,dc=[nom],dc=com
objectClass: organizationalUnit
ou: developers

dn: cn=LDAP,ou=networkadmins,ou=Groups,dc=[nom],dc=com
objectClass: posixGroup
cn: LDAP
gidNumber: 1

dn: cn=Appache,ou=networkadmins,ou=Groups,dc=[nom],dc=com
objectClass: posixGroup
cn: Appache
gidNumber: 2
```

```bash
sudo ldapadd -x -D "cn=admin,dc=[nom],dc=com" -W -f add-groups.ldif
```

- `ou=networkadmins` et `ou=developers` sont des **conteneurs** sous `ou=Groups`.
- `cn=LDAP` et `cn=Appache` sont de vrais **groupes Unix** (`posixGroup` + `gidNumber`) sous `networkadmins`.
- L'ordre compte : `ou=networkadmins` est créé **avant** les groupes qu'il contient.
- (« Appache » avec deux *p* est l'orthographe de l'énoncé : garde-la partout.)

#### 3.3 Utilisateurs : `add-users.ldif`

```
dn: uid=[prenom],ou=People,dc=[nom],dc=com
objectClass: top
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
uid: [prenom]
sn: [nom]
givenName: [prenom]
cn: [prenom] [nom]
displayName: [prenom] [nom]
uidNumber: 11
gidNumber: 11
userPassword: 123
gecos: [prenom] [nom]
loginShell: /bin/bash
homeDirectory: /home/[prenom]

dn: cn=[prenom],cn=LDAP,ou=networkadmins,ou=Groups,dc=[nom],dc=com
objectClass: posixGroup
cn: [prenom]
gidNumber: 1
memberUid: 11

dn: uid=ahmed,ou=People,dc=[nom],dc=com
objectClass: top
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
uid: ahmed
sn: ahmed
givenName: ahmed
cn: ahmed
displayName: ahmed ahmed
uidNumber: 13
gidNumber: 2
userPassword: 147
gecos: Ahmed ahmed
loginShell: /bin/bash
homeDirectory: /home/ahmed
```

```bash
sudo ldapadd -x -D "cn=admin,dc=[nom],dc=com" -W -f add-users.ldif
```

- Entrée **utilisateur** : combine `inetOrgPerson` (identité : `sn`, `cn` obligatoires) + `posixAccount` (compte Unix : UID/GID/home/shell) + `shadowAccount` (infos de mot de passe).
- `uidNumber` / `gidNumber` : identifiants numériques Unix de l'utilisateur et de son groupe principal.
- La 2ᵉ entrée **rattache** l'utilisateur au groupe `LDAP` dans l'arbre (comme sur la figure).
- 💡 Dans un annuaire Unix standard, `memberUid` contient normalement le **nom de login** (`[prenom]`) plutôt que le numéro ; ici on suit l'énoncé.
- 🔒 `userPassword: 123` est stocké **en clair**. Pour un mot de passe haché :

```bash
slappasswd
# → {SSHA}xxxxxxxxxxxxxxxx   (à coller dans userPassword:)
```

- `slappasswd` : demande un mot de passe et affiche son haché (`{SSHA}` = SHA salé).

### Étape 4 – Tests et validation

#### 4.1 Test local avec `ldapsearch`

```bash
ldapsearch -x -LLL -b dc=[nom],dc=com
```

- `-x` : simple (ici **anonyme**, aucun `-D`) ; `-LLL` : sortie épurée ; `-b` : **base DN** de la recherche (racine de ton domaine).
- Affiche tout l'arbre : base, `People`, `Groups`, `Files`, utilisateurs, groupes. (`userPassword` n'est pas visible en anonyme.)

Variantes utiles :

```bash
# un utilisateur précis
ldapsearch -x -LLL -b dc=[nom],dc=com "(uid=[prenom])"

# seulement certains attributs
ldapsearch -x -LLL -b ou=People,dc=[nom],dc=com "(objectClass=posixAccount)" uid uidNumber homeDirectory

# un seul niveau sous une OU (-s one) ; seulement l'entrée elle-même (-s base)
ldapsearch -x -LLL -b ou=Groups,dc=[nom],dc=com -s one dn

# vérifier qu'un utilisateur peut s'authentifier
ldapwhoami -x -D "uid=[prenom],ou=People,dc=[nom],dc=com" -w 123
```

- `"(uid=[prenom])"` : **filtre** de recherche (entre guillemets) ; les attributs listés à la fin limitent l'affichage.
- `-s` : portée (*scope*) : `base`, `one` (un niveau), `sub` (tout le sous-arbre, défaut).
- `ldapwhoami … -w 123` : teste un *bind* ; la réponse `dn:uid=[prenom],ou=People,…` confirme que l'authentification fonctionne.

#### 4.2 Client graphique distant : JXplorer

```bash
sudo apt install jxplorer
```

Connexion : menu **File → Connect**, puis :

| Champ | Valeur |
|-------|--------|
| Host | IP du serveur (`ip a` sur le serveur) |
| Port | `389` |
| Base DN | `dc=[nom],dc=com` |
| Level | `User + Password` |
| User DN | `cn=admin,dc=[nom],dc=com` |
| Password | mot de passe admin |

- Si la connexion réussit, l'arbre `com → [nom] → Files / Groups / People` s'affiche à gauche.
- Si le client est sur **une autre machine**, ouvrir le port : `sudo ufw allow 389/tcp`, et tester d'abord `ldapsearch -x -H ldap://IP_SERVEUR -b dc=[nom],dc=com`.

---

## 4. Dépannage

| Erreur | Cause | Solution |
|--------|-------|----------|
| `ldap_bind: Invalid credentials (49)` | mauvais mot de passe ou mauvais DN | vérifier `cn=admin,dc=[nom],dc=com` |
| `No such object (32)` | l'entrée **parente** n'existe pas | créer le parent avant (ordre du LDIF) |
| `Already exists (68)` | l'entrée existe déjà | la supprimer (`ldapdelete`) ou l'ignorer |
| `Object class violation (65)` / `requires attribute` | attribut obligatoire manquant (`sn`, `cn`, `gidNumber`…) | compléter l'entrée |
| `Can't contact LDAP server (-1)` | `slapd` arrêté / mauvaise URI / pare-feu | `systemctl status slapd`, `URI` dans `ldap.conf` |
| `invalid format` à l'import | ligne vide manquante, espace en trop | respecter le format LDIF |
| L'import s'arrête à la première erreur | comportement par défaut | ajouter `-c` (*continue*) pour poursuivre |
| Logs | | `sudo journalctl -u slapd` |

Bonus : supprimer une entrée : `ldapdelete -x -D "cn=admin,dc=[nom],dc=com" -W "uid=ahmed,ou=People,dc=[nom],dc=com"`.

---

## 5. Questions de révision

1. Qu'est-ce qu'un annuaire LDAP ? En quoi diffère-t-il d'une base de données relationnelle ?
2. Quelle est la différence entre un **DN** et un **RDN** ?
3. À quoi servent `dc`, `ou`, `cn` et `uid` ?
4. Qu'est-ce qu'un fichier **LDIF** ? Quelles règles faut-il respecter ?
5. Que signifient `-x`, `-D`, `-W`, `-f`, `-b`, `-LLL`, `-Y EXTERNAL`, `-H ldapi:///` ?
6. Pourquoi l'ordre des entrées dans le LDIF est-il important ?
7. Pourquoi ne faut-il pas stocker `userPassword` en clair ? Comment le hacher ?
8. Quelle différence entre `ldap://` (389) et `ldaps://` (636) ?
