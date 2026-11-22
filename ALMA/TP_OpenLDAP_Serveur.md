# TP – Lab OpenLDAP : installer et configurer le serveur (AlmaLinux 8)

**Distribution :** AlmaLinux 8 (`dnf`) · **Serveur :** `ldapmaster.arctic.com` (192.168.205.2) · **Client :** `ldapclient.arctic.com` · **Domaine LDAP :** `dc=arctic,dc=com` · **Paquets :** `symas-openldap-servers`, `symas-openldap-clients`

> ⚠️ Cette fiche suit l'énoncé du lab. Les remarques marquées **à vérifier** sont mon interprétation : à confronter avec ton cours et à tester sur ta VM.

---

## 1. Théorie

### 1.1 Qu'est-ce que LDAP ?

**LDAP** (*Lightweight Directory Access Protocol*) est un protocole d'accès à un **annuaire** : une base hiérarchique optimisée pour la **lecture** (utilisateurs, groupes, machines…). Usage typique ici : **authentification centralisée** des comptes Linux.

- Ports : **389** (LDAP, éventuellement avec StartTLS) et **636** (LDAPS, TLS dès la connexion).
- **OpenLDAP** : implémentation libre ; son démon est **`slapd`**.

### 1.2 Vocabulaire

| Terme | Signification |
|-------|---------------|
| **Entrée** | un objet de l'annuaire (utilisateur, groupe, unité…) |
| **DN** | *Distinguished Name* : chemin unique d'une entrée (`uid=testuser,ou=People,dc=arctic,dc=com`) |
| **RDN** | partie la plus à gauche du DN (`uid=testuser`) |
| **DIT** | *Directory Information Tree* : l'arborescence |
| **Suffixe / Base** | racine de l'arbre (`dc=arctic,dc=com`) |
| **Attribut** | propriété d'une entrée (`cn`, `sn`, `uidNumber`…) |
| **objectClass** | « type » de l'entrée : définit les attributs obligatoires/facultatifs |
| **Schéma** | ensemble des attributs et objectClass autorisés |
| **LDIF** | format texte pour décrire/modifier des entrées |
| **RootDN / RootPW** | compte administrateur de la base (`cn=Manager,...`) et son mot de passe |

**Abréviations de DN :** `dc` = domain component · `ou` = organizational unit · `cn` = common name · `uid` = identifiant · `o` = organisation.

### 1.3 Arbre construit dans ce lab

```
dc=arctic,dc=com
├── cn=Manager
├── ou=People
│   └── uid=testuser
└── ou=Group
    └── cn=testuser
```

### 1.4 Deux bases dans slapd (configuration dynamique `cn=config`)

| Base | Contenu | Accès |
|------|---------|-------|
| `cn=config` | **configuration** de slapd (`olcDatabase={0}config`, `{1}monitor`, `{2}mdb`) | via `ldapadd/ldapmodify -Y EXTERNAL -H ldapi:///` (root local) |
| `dc=arctic,dc=com` | **données** (utilisateurs, groupes) | via `-x -D cn=Manager,... -W` |

- `-Y EXTERNAL -H ldapi:///` : authentification **SASL EXTERNAL** par socket Unix (identité du root local), sans mot de passe.
- `-x` : authentification **simple** (bind avec DN + mot de passe).

### 1.5 Outils

| Commande | Rôle |
|----------|------|
| `ldapadd` | ajoute des entrées (depuis un LDIF) |
| `ldapmodify` | modifie des entrées (`changetype: modify`) |
| `ldapdelete` | supprime des entrées |
| `ldapsearch` | recherche |
| `slappasswd` | génère un hash de mot de passe (`{SSHA}`) |

---

## 2. Plan du TP

1. Préparer le serveur (mise à jour, nom d'hôte, `/etc/hosts`)
2. Installer OpenLDAP (dépôt Symas)
3. Configurer slapd : DB_CONFIG, service, pare-feu
4. Créer le mot de passe administrateur
5. Importer les schémas (cosine, nis, inetorgperson, sudo)
6. Définir le domaine (suffixe, RootDN)
7. Créer la structure (OU People/Group)
8. Gérer les utilisateurs
9. Activer SSL/TLS

---

## 3. Pratique

### Étape 0 : Préparation du serveur

```bash
sudo dnf update -y
sudo reboot -f
sudo hostnamectl set-hostname ldapmaster.arctic.com
sudo vim /etc/hosts
```

```
192.168.205.2   ldapmaster.arctic.com
192.168.205.13  ldapclient.arctic.com
```

- Le **FQDN correct** et la résolution par `/etc/hosts` sont indispensables (certificats TLS, client).
- ⚠️ L'IP du client est `.13` dans le lab serveur et `.16` dans le lab client : **utiliser la même** sur les deux machines.

### Étape 1 : Installer les paquets OpenLDAP

```bash
sudo wget -q https://repo.symas.com/configs/SOFL/rhel8/sofl.repo -O /etc/yum.repos.d/sofl.repo
sudo dnf install symas-openldap-clients symas-openldap-servers
rpm -qa | grep ldap
```

- AlmaLinux 8 ne fournit plus le **serveur** OpenLDAP : on passe par le dépôt **Symas** (`sofl.repo`).
- Dépendance installée automatiquement : `symas-openldap`.

### Étape 2 : Configurer le serveur

#### 2.1 Base de données et service

```bash
sudo cp /usr/share/openldap-servers/DB_CONFIG.example /var/lib/ldap/DB_CONFIG
sudo chown ldap. /var/lib/ldap/DB_CONFIG
sudo systemctl enable --now slapd
systemctl status slapd
```

- `DB_CONFIG` : paramètres de la base (cache, etc.) ; propriétaire = utilisateur `ldap`.
- `slapd` écoute sur `ldap:///`, `ldaps:///` et `ldapi:///` (voir `ExecStart`).

Pare-feu :

```bash
sudo firewall-cmd --add-service={ldap,ldaps} --permanent
sudo firewall-cmd --reload
```

#### 2.2 Mot de passe administrateur

```bash
slappasswd
# → {SSHA}Z6wjWlDYAgHkMpezhpCRSw3fGCLiNdMM   (hash à copier)
```

`changerootpw.ldif` :

```ldif
dn: olcDatabase={0}config,cn=config
changetype: modify
add: olcRootPW
olcRootPW: {SSHA}Z6wjWlDYAgHkMpezhpCRSw3fGCLiNdMM
```

```bash
sudo ldapadd -Y EXTERNAL -H ldapi:/// -f changerootpw.ldif
```

- `{SSHA}` : mot de passe **haché salé** (SHA-1 + sel), jamais en clair dans le LDIF.
- Cette étape définit le mot de passe d'administration de la **base de configuration** `cn=config`.

#### 2.3 Importer les schémas de base

Les schémas sont dans `/etc/openldap/schema/`. Il faut **cosine**, **nis** et **inetorgperson** :

```bash
sudo ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/cosine.ldif
sudo ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/nis.ldif
sudo ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/inetorgperson.ldif
```

| Schéma | Apporte |
|--------|---------|
| `cosine` | attributs de base (dépendance des autres) |
| `nis` | `posixAccount`, `posixGroup`, `shadowAccount` (comptes **Unix**) |
| `inetorgperson` | `inetOrgPerson` (personne dans une organisation) |

**Schéma sudo** (pour gérer les règles `sudo` dans LDAP) :

```bash
sudo cp /usr/share/doc/sudo/schema.OpenLDAP /etc/openldap/schema/sudo.schema
sudo tee /etc/openldap/schema/sudo.ldif <<EOF
dn: cn=sudo,cn=schema,cn=config
objectClass: olcSchemaConfig
cn: sudo
olcAttributeTypes: ( 1.3.6.1.4.1.15953.9.1.1 NAME 'sudoUser' ... )
olcObjectClasses: ( 1.3.6.1.4.1.15953.9.2.1 NAME 'sudoRole' SUP top STRUCTURAL ... )
EOF
sudo ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/sudo.ldif
```

- (Le contenu complet du LDIF — attributs `sudoUser`, `sudoHost`, `sudoCommand`, `sudoRunAs`, `sudoOption`, `sudoRunAsUser`, `sudoRunAsGroup` et classe `sudoRole` — est dans l'énoncé.)

#### 2.4 Définir le nom de domaine

`setdomainname.ldif` :

```ldif
dn: olcDatabase={2}mdb,cn=config
changetype: modify
replace: olcSuffix
olcSuffix: dc=arctic,dc=com

dn: olcDatabase={2}mdb,cn=config
changetype: modify
replace: olcRootDN
olcRootDN: cn=Manager,dc=arctic,dc=com

dn: olcDatabase={2}mdb,cn=config
changetype: modify
replace: olcRootPW
olcRootPW: {SSHA}Z6wjWlDYAgHkMpezhpCRSw3fGCLiNdMM

dn: olcDatabase={1}monitor,cn=config
changetype: modify
replace: olcAccess
olcAccess: {0}to * by dn.base="gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth" read
  by dn.base="cn=Manager,dc=arctic,dc=com" read by * none
```

```bash
sudo ldapmodify -Y EXTERNAL -H ldapi:/// -f setdomainname.ldif
```

| Attribut | Rôle |
|----------|------|
| `olcSuffix` | racine de l'annuaire |
| `olcRootDN` | administrateur de la base de **données** |
| `olcRootPW` | son mot de passe (haché) |
| `olcAccess` (monitor) | règles d'accès (ACL) à la base `monitor` |

### Étape 3 : Créer la structure (unités organisationnelles)

`adddomain.ldif` :

```ldif
dn: dc=arctic,dc=com
objectClass: top
objectClass: dcObject
objectClass: organization
o: My example Organisation
dc: arctic

dn: cn=Manager,dc=arctic,dc=com
objectClass: organizationalRole
cn: Manager
description: OpenLDAP Manager

dn: ou=People,dc=arctic,dc=com
objectClass: organizationalUnit
ou: People

dn: ou=Group,dc=arctic,dc=com
objectClass: organizationalUnit
ou: Group
```

```bash
sudo ldapadd -x -D cn=Manager,dc=arctic,dc=com -W -f adddomain.ldif
```

- `-D` : DN avec lequel on se connecte (*bind DN*) ; `-W` : demande le mot de passe.
- Les entrées parentes (racine) doivent être créées **avant** leurs enfants.

### Étape 4 : Gérer les utilisateurs

`addtestuser.ldif` :

```ldif
dn: uid=testuser,ou=People,dc=arctic,dc=com
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
cn: testuser
sn: temp
userPassword: {SSHA}XXXXXXXXXXXXXXXXXXXX
loginShell: /bin/bash
uidNumber: 2000
gidNumber: 2000
homeDirectory: /home/testuser
shadowLastChange: 0
shadowMax: 0
shadowWarning: 0

dn: cn=testuser,ou=Group,dc=arctic,dc=com
objectClass: posixGroup
cn: testuser
gidNumber: 2000
memberUid: testuser
```

```bash
sudo ldapadd -x -D cn=Manager,dc=arctic,dc=com -W -f addtestuser.ldif
ldapsearch -x cn=testuser -b dc=arctic,dc=com
```

- Le mot de passe de l'utilisateur est généré avec `slappasswd` et collé dans `userPassword`.
- Attributs **POSIX** : `uidNumber`, `gidNumber`, `homeDirectory`, `loginShell` → l'entrée est utilisable comme **compte Linux**.
- `inetOrgPerson` exige `cn` et `sn`.

**Suppression :**

```bash
sudo ldapdelete -x -W -D 'cn=Manager,dc=arctic,dc=com' "uid=testuser,ou=People,dc=arctic,dc=com"
sudo ldapdelete -x -W -D 'cn=Manager,dc=arctic,dc=com' "cn=testuser,ou=Group,dc=arctic,dc=com"
```

- ⚠️ L'énoncé écrit `testuser1` dans les exemples de suppression alors que l'utilisateur créé s'appelle `testuser` : **adapter le DN**.

### Étape 5 : SSL/TLS

**Certificat auto-signé :**

```bash
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/pki/tls/ldapserver.key \
  -out /etc/pki/tls/ldapserver.crt
sudo chown ldap:ldap /etc/pki/tls/{ldapserver.crt,ldapserver.key}
```

| Option | Rôle |
|--------|------|
| `-x509` | produit directement un certificat auto-signé |
| `-nodes` | clé privée **non chiffrée** (slapd doit pouvoir la lire au démarrage) |
| `-days 365` | validité |
| `-newkey rsa:2048` | nouvelle clé RSA 2048 bits |

> 💡 Le *Common Name* du certificat doit être le **FQDN** du serveur : `ldapmaster.arctic.com`.

`add-tls.ldif` :

```ldif
dn: cn=config
changetype: modify
add: olcTLSCACertificateFile
olcTLSCACertificateFile: /etc/pki/tls/ldapserver.crt
-
add: olcTLSCertificateKeyFile
olcTLSCertificateKeyFile: /etc/pki/tls/ldapserver.key
-
add: olcTLSCertificateFile
olcTLSCertificateFile: /etc/pki/tls/ldapserver.crt
```

```bash
sudo ldapadd -Y EXTERNAL -H ldapi:/// -f add-tls.ldif
sudo vim /etc/openldap/ldap.conf      # TLS_CACERT /etc/pki/tls/ldapserver.crt
sudo systemctl restart slapd
```

- Le `-` sépare plusieurs opérations dans une même entrée LDIF.
- Certificat auto-signé : le **client** doit avoir une copie du `.crt` (voir lab client).

---

## 4. Dépannage

| Symptôme | Cause probable | Solution |
|----------|----------------|----------|
| `slapd` ne démarre pas | erreur de config, droits | `journalctl -u slapd`, `slaptest -u` |
| `ldap_bind: Invalid credentials (49)` | mauvais mot de passe / mauvais DN | vérifier `-D`, hash `olcRootPW` |
| `ldap_add: Invalid syntax / Object class violation` | attribut obligatoire manquant | vérifier `objectClass` et attributs requis (`sn`, `cn`…) |
| `No such object (32)` | entrée parente absente | créer la racine / OU avant |
| `Already exists (68)` | entrée déjà créée | `ldapmodify` ou `ldapdelete` |
| `Naming violation` / schéma inconnu | schéma non importé | `ldapadd ... cosine/nis/inetorgperson.ldif` |
| Client ne se connecte pas | pare-feu / DNS | `firewall-cmd`, `/etc/hosts`, `ping` |
| TLS refusé | certificat / droits / CN | `chown ldap:ldap`, CN = FQDN, `TLS_CACERT` |
| Voir la configuration | | `sudo ldapsearch -Y EXTERNAL -H ldapi:/// -b cn=config` |

---

## 5. Questions de révision

1. Qu'est-ce que LDAP ? Quels sont ses ports (clair / TLS) ?
2. Différence entre DN, RDN, DIT, suffixe ?
3. À quoi servent un `objectClass` et un **schéma** ? Quels schémas importe-t-on et pourquoi ?
4. Que fait `slappasswd` ? Que signifie `{SSHA}` ?
5. Différence entre `-Y EXTERNAL -H ldapi:///` et `-x -D ... -W` ?
6. Quelle différence entre la base `cn=config` et la base `dc=arctic,dc=com` ?
7. Quel est le rôle de `olcSuffix`, `olcRootDN`, `olcRootPW` ?
8. Pourquoi créer `ou=People` et `ou=Group` ?
9. Quels objectClass/attributs rendent une entrée utilisable comme compte Linux ?
10. Comment ajouter, rechercher et supprimer un utilisateur ?
11. Comment activer TLS sur OpenLDAP (certificat, LDIF, `ldap.conf`) ?
12. Pourquoi le dépôt Symas est-il nécessaire sur AlmaLinux 8 ?
