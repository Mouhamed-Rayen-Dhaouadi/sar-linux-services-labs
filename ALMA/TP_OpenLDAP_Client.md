# TP – Lab OpenLDAP : configurer le client (AlmaLinux 8, SSSD)

**Distribution :** AlmaLinux 8 (`dnf`) · **Client :** `ldapclient.arctic.com` · **Serveur LDAP :** `ldapmaster.arctic.com` (192.168.205.2) · **Base :** `dc=arctic,dc=com`

> ⚠️ Cette fiche suit l'énoncé du lab. Les remarques marquées **à vérifier** sont mon interprétation : à confronter avec ton cours et à tester sur ta VM. Prérequis : le **serveur** du lab précédent (`TP_OpenLDAP_Serveur.md`) est opérationnel.

---

## 1. Théorie

### 1.1 Objectif

Permettre à une machine Linux d'**authentifier ses utilisateurs auprès de l'annuaire LDAP** : les comptes n'existent plus dans `/etc/passwd` du client mais sur le serveur.

### 1.2 Chaîne d'authentification côté client

```
Application (login, ssh, gdm)
        │
       PAM  ─────────►  vérifie le mot de passe        (auth)
        │
       NSS  ─────────►  résout uid/gid, home, shell    (passwd, group)
        │
      SSSD  ─────────►  démon intermédiaire + cache
        │
  Serveur OpenLDAP (ldap:// + StartTLS)
```

| Brique | Rôle |
|--------|------|
| **SSSD** (*System Security Services Daemon*) | accès unifié aux annuaires (LDAP, AD, IPA), **cache** des identifiants (travail hors ligne) |
| **NSS** (`/etc/nsswitch.conf`) | où chercher utilisateurs/groupes (`files sss`) |
| **PAM** | méthode d'authentification des services |
| **authselect** | outil qui gère les profils PAM/NSS (remplace `authconfig`) |
| **oddjob-mkhomedir** | **crée le répertoire personnel** à la première connexion |

### 1.3 Pourquoi `oddjobd` ?

Les comptes LDAP n'ont pas de `/home/<user>` sur le client. L'option `with-mkhomedir` + le service `oddjobd` le **créent automatiquement** au premier login.

---

## 2. Plan du TP

1. Nom d'hôte, `/etc/hosts`, test de connectivité
2. Installer client OpenLDAP + SSSD
3. Choisir le profil `authselect sssd with-mkhomedir`
4. Configurer `ldap.conf` + copier le certificat
5. Configurer `sssd.conf`
6. Tester (ldapsearch, `id`, console/GUI, SSH)

---

## 3. Pratique

### Étape 0 : Préparation

```bash
sudo hostnamectl set-hostname ldapclient.arctic.com
sudo vim /etc/hosts
```

```
##OpenLDAP server
192.168.205.2   ldapmaster.arctic.com

##OpenLDAP Client
192.168.205.16  ldapclient.arctic.com
```

```bash
ping -c3 ldapmaster.arctic.com
```

- ⚠️ L'IP du client est `.16` ici et `.13` dans le lab serveur : **rester cohérent** avec ta VM.

### Étape 1 : Installer les paquets

```bash
sudo dnf install openldap-clients sssd sssd-ldap oddjob-mkhomedir libsss_sudo
```

| Paquet | Rôle |
|--------|------|
| `openldap-clients` | outils `ldapsearch`, etc. |
| `sssd`, `sssd-ldap` | démon SSSD + fournisseur LDAP |
| `oddjob-mkhomedir` | création automatique du home |
| `libsss_sudo` | règles `sudo` depuis LDAP |

**Profil d'authentification :**

```bash
authselect list
sudo authselect select sssd with-mkhomedir --force
sudo systemctl enable --now oddjobd.service
systemctl status oddjobd.service
```

- `authselect select sssd` : active SSSD pour PAM et NSS (`passwd`, `group`, `netgroup`, `automount`, `services`).
- `with-mkhomedir` : active `pam_oddjob_mkhomedir`.
- `--force` : écrase la configuration existante ; une **sauvegarde** est créée dans `/var/lib/authselect/backups/`.
- Vérifier : `authselect current`.

### Étape 2 : Configurer le client OpenLDAP et SSSD

#### 2.1 `/etc/openldap/ldap.conf`

```
URI ldap://ldapmaster.arctic.com
BASE dc=arctic,dc=com
SUDOERS_BASE ou=sudo,dc=arctic,dc=com
TLS_CACERT /etc/pki/tls/ldapserver.crt
```

- Ces valeurs servent de **défauts** aux commandes `ldapsearch`, etc.

**Copier le certificat du serveur vers le client :**

```bash
# sur le serveur
scp /etc/pki/tls/ldapserver.crt ldapclient.arctic.com:/etc/pki/tls/
```

- Nécessaire car le certificat est **auto-signé** : le client doit le connaître pour valider TLS.

#### 2.2 `/etc/sssd/sssd.conf`

```ini
[domain/default]
id_provider = ldap
autofs_provider = ldap
auth_provider = ldap
chpass_provider = ldap
ldap_uri = ldap://ldapmaster.arctic.com
ldap_search_base = dc=arctic,dc=com
sudoers_base = ou=sudo,dc=arctic,dc=com
sudo_provider = ldap
ldap_id_use_start_tls = True
ldap_tls_cacertdir = /etc/pki/tls/
cache_credentials = True
ldap_tls_reqcert = allow

[sssd]
services = nss, pam, autofs, sudo
domains = default

[nss]
homedir_substring = /home

[sudo]
```

| Paramètre | Rôle |
|-----------|------|
| `id_provider` | source des **identités** (utilisateurs/groupes) |
| `auth_provider` | source de l'**authentification** |
| `chpass_provider` | changement de mot de passe |
| `autofs_provider` / `sudo_provider` | montages auto / règles sudo depuis LDAP |
| `ldap_uri` | adresse du serveur |
| `ldap_search_base` | base de recherche |
| `ldap_id_use_start_tls` | **StartTLS** sur le port 389 |
| `ldap_tls_cacertdir` | dossier du certificat d'autorité |
| `ldap_tls_reqcert = allow` | tolérant sur la validation du certificat serveur (⚠️ pas recommandé en production : préférer `demand`) |
| `cache_credentials` | mémorise les mots de passe pour fonctionner **hors ligne** |
| `[sssd] services / domains` | services SSSD actifs et domaines utilisés |
| `homedir_substring` | préfixe des homes |

> 💡 Dans l'énoncé la ligne `sudoers_base ou=sudo,...` n'a pas de `=`. *À vérifier* : la directive SSSD documentée pour la base sudo est plutôt `ldap_sudo_search_base`. Et `ou=sudo` **n'est pas créé** sur le serveur dans le lab serveur : sans impact pour l'authentification simple.

**Droits obligatoires et redémarrage :**

```bash
sudo chmod 0600 /etc/sssd/sssd.conf
sudo systemctl restart sssd
systemctl status sssd
```

- **SSSD refuse de démarrer** si `sssd.conf` n'est pas en `0600` appartenant à root (le fichier peut contenir des secrets).
- Le service `sssd` doit être **activé** au démarrage (`systemctl enable sssd`).

### Étape 3 : Tester l'authentification

**1. Interroger l'annuaire :**

```bash
ldapsearch -x -b "ou=people,dc=arctic,dc=com"
```

**2. Vérifier que le client voit le compte LDAP :**

```bash
id testuser
getent passwd testuser
```

- Attendu : `uid=2000(testuser) gid=2000(testuser) groups=2000(testuser)`.

**3. Se connecter** :

- **Interface graphique** (GDM) : saisir `testuser` puis le mot de passe.
- **SSH** :

```bash
ssh testuser@192.168.205.16
whoami
```

- Au premier login, le **home est créé automatiquement** (`/home/testuser`).

---

## 4. Dépannage

| Symptôme | Cause probable | Solution |
|----------|----------------|----------|
| `sssd` ne démarre pas | droits de `sssd.conf` | `chmod 0600`, `chown root:root` |
| `id testuser` : *no such user* | SSSD/NSS mal configuré, cache | `systemctl restart sssd`, `sss_cache -E`, vérifier `nsswitch.conf` |
| Login refusé | mot de passe incorrect, PAM non configuré | `authselect current`, journaux |
| Pas de répertoire personnel | `oddjobd` inactif / profil sans `with-mkhomedir` | `systemctl enable --now oddjobd`, `authselect select sssd with-mkhomedir` |
| Erreur TLS | certificat absent/incorrect, `TLS_CACERT` | copier `ldapserver.crt`, vérifier CN = FQDN |
| `Can't contact LDAP server` | réseau, pare-feu serveur, DNS | `ping`, ports 389/636, `/etc/hosts` |
| Modif LDAP non visible | cache SSSD | `sudo sss_cache -E` |
| Logs | | `/var/log/sssd/`, `journalctl -u sssd` |

---

## 5. Questions de révision

1. À quoi sert SSSD ? Quels avantages (cache, hors ligne) ?
2. Quel est le rôle de **NSS** et de **PAM** dans l'authentification LDAP ?
3. Que fait `authselect select sssd with-mkhomedir --force` ?
4. Pourquoi installe-t-on `oddjob-mkhomedir` et active-t-on `oddjobd` ?
5. Pourquoi copier `ldapserver.crt` sur le client ?
6. Que signifient `id_provider`, `auth_provider`, `ldap_search_base`, `ldap_id_use_start_tls` ?
7. Pourquoi `sssd.conf` doit-il être en `0600` ?
8. Que fait `cache_credentials = True` ?
9. Comment vérifier qu'un utilisateur LDAP est reconnu par le client ?
10. Comment tester la connexion en console, en GUI et en SSH ?
