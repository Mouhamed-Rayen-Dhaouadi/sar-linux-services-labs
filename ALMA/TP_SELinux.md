# TP – Lab SELinux : modes et contextes (AlmaLinux)

**Distribution :** AlmaLinux 8 (`yum`/`dnf`) · **Machine :** `serverX` (connexion en `root`) · **Outils :** `getenforce`, `setenforce`, `semanage`, `restorecon` · **Service de test :** Apache (`httpd`)

> ⚠️ Cette fiche suit l'énoncé du lab. Les remarques marquées **à vérifier** sont mon interprétation : à confronter avec ton cours et à tester sur ta VM.

---

## 1. Théorie

### 1.1 Qu'est-ce que SELinux ?

**SELinux** (*Security-Enhanced Linux*) est un système de **contrôle d'accès obligatoire** (**MAC**) intégré au noyau. Il s'ajoute aux droits Unix classiques (**DAC** : `rwx`, propriétaire, groupe).

- Même **root** ou un fichier en `777` peuvent être bloqués si la **politique SELinux** l'interdit.
- Principe du **moindre privilège** : un processus (ex. Apache) ne peut accéder qu'à ce que la politique lui autorise explicitement.
- Un accès n'est autorisé que si **DAC ET SELinux** l'autorisent.

### 1.2 Les trois modes

| Mode | Comportement |
|------|--------------|
| **Enforcing** | la politique est **appliquée** : les accès interdits sont **bloqués** et journalisés |
| **Permissive** | la politique n'est **pas appliquée** : les violations sont **seulement journalisées** (utile pour diagnostiquer) |
| **Disabled** | SELinux est **désactivé** |

- Passer de/vers `disabled` nécessite un **redémarrage** (et un re-étiquetage au retour).
- `enforcing` ↔ `permissive` se fait **à chaud** avec `setenforce`.

### 1.3 Mode courant vs mode par défaut

| | Mode **courant** (runtime) | Mode **par défaut** (au boot) |
|---|---------------------------|--------------------------------|
| Voir | `getenforce` / `sestatus` | `/etc/selinux/config` |
| Changer | `setenforce 0|1` (temporaire) | éditer `SELINUX=` (permanent, au prochain reboot) |

### 1.4 Le contexte SELinux

Chaque fichier, processus, port… possède un **contexte** (vu avec `ls -Z`, `ps -Z`) :

```
unconfined_u : object_r : httpd_sys_content_t : s0
   utilisateur    rôle         type            niveau
```

| Champ | Rôle |
|-------|------|
| **Utilisateur** (`_u`) | identité SELinux |
| **Rôle** (`_r`) | rôle (`object_r` pour les fichiers) |
| **Type** (`_t`) | **le plus important** (politique *targeted*) : c'est lui qui décide des accès |
| **Niveau** (`s0`) | MLS/MCS (sécurité multi-niveaux) |

- Règle : le **type du processus** (`httpd_t`) doit avoir le droit d'accéder au **type du fichier** (`httpd_sys_content_t`).
- Les types usuels : `httpd_sys_content_t` (contenu web lisible), `httpd_sys_rw_content_t` (contenu web inscriptible), `default_t` (étiquette par défaut d'un dossier créé hors arborescence connue, **non** autorisée à Apache).

### 1.5 Commandes essentielles

| Commande | Rôle |
|----------|------|
| `getenforce` | affiche le mode courant |
| `sestatus` | état détaillé (mode courant, mode config, politique) |
| `setenforce 0` / `1` | permissive / enforcing (temporaire) |
| `ls -Z`, `ps -Z`, `id -Z` | afficher les contextes |
| `semanage fcontext -a -t TYPE 'chemin(/.*)?'` | définir une **règle** de contexte **persistante** |
| `restorecon -Rv chemin` | **appliquer** les contextes de la politique (`-R` récursif, `-v` verbeux) |
| `chcon -t TYPE chemin` | changer le contexte **temporairement** (perdu au re-étiquetage) |
| `semanage port -l` | lister les ports autorisés par type |
| `getsebool -a` / `setsebool -P` | lister / modifier les **booléens** (`-P` = persistant) |
| `ausearch -m avc` / `sealert` | analyser les refus |

### 1.6 `chcon` vs `semanage fcontext` + `restorecon`

| | `chcon` | `semanage fcontext` + `restorecon` |
|---|---------|------------------------------------|
| Persistant ? | **non** (écrasé par un `restorecon` ou un re-étiquetage) | **oui** (règle enregistrée dans la politique) |
| Usage | test rapide | solution **correcte** |

---

## 2. Plan du TP

**Exercice 1 – Modes SELinux**
1. Afficher le mode courant
2. Mettre le mode par défaut à `permissive` + reboot
3. Vérifier après reboot
4. Remettre `enforcing` par défaut
5. Passer le mode courant en `enforcing`

**Exercice 2 – Contexte d'un répertoire web non standard**
1. Installer Apache
2. Créer `/custom` et `index.html`
3. Modifier `httpd.conf`
4. Constater l'erreur 403
5. Corriger avec `semanage fcontext` + `restorecon`
6. Vérifier

---

## 3. Pratique

### Exercice 1 : Modification des modes SELinux

**1. Mode courant**

```bash
getenforce
```

**2. Passer le mode par défaut à `permissive`, puis redémarrer**

```bash
vi /etc/selinux/config
grep '^SELINUX' /etc/selinux/config
# SELINUX=permissive
# SELINUXTYPE=targeted
reboot
```

- `SELINUX=` : le **mode** au démarrage ; `SELINUXTYPE=targeted` : seuls certains services sont confinés (politique par défaut).

**3. Après redémarrage**

```bash
getenforce
# Permissive
```

**4. Remettre `enforcing` par défaut**

```bash
vi /etc/selinux/config
grep '^SELINUX' /etc/selinux/config
# SELINUX=enforcing
# SELINUXTYPE=targeted
```

**5. Passer immédiatement en `enforcing`** (sans reboot)

```bash
setenforce 1
getenforce
# Enforcing
```

- Le fichier de config ne change **pas** le mode courant ; `setenforce` ne change **pas** le fichier. Il faut les deux pour un résultat cohérent et persistant.

---

### Exercice 2 : Modifier de façon persistante le contexte d'un répertoire

**Scénario :** un serveur web publie du contenu depuis une racine **non standard** (`/custom`).

**1. Installer Apache**

```bash
yum install -y httpd
```

**2. Créer la racine et la page**

```bash
mkdir /custom
echo 'This is serverX. ' > /custom/index.html
```

**3. Configurer Apache** : remplacer les **deux** occurrences de `/var/www/html` par `/custom` dans `/etc/httpd/conf/httpd.conf` :

```bash
vi /etc/httpd/conf/httpd.conf
grep custom /etc/httpd/conf/httpd.conf
```

```
DocumentRoot "/custom"
<Directory "/custom">
```

- `DocumentRoot` : racine du site ; `<Directory>` : règles d'accès (`Require all granted`) sur ce dossier.

**Démarrer le service :**

```bash
systemctl start httpd
```

**4. Tester → erreur 403**

```bash
curl http://localhost/index.html
# ou navigateur sur http://localhost/index.html
```

- **Résultat : `403 Forbidden`.**
- **Cause :** les droits Unix sont corrects, mais **SELinux interdit** à Apache (`httpd_t`) de lire `/custom` : ce dossier a le type **`default_t`** (étiquette par défaut), non autorisé pour Apache.

**Vérifier dans les logs :**

```bash
sudo tail -n 20 /var/log/audit/audit.log
# ou
sudo tail -f /var/log/httpd/error_log
```

- Dans `audit.log`, chercher les lignes **`avc: denied`** (AVC = *Access Vector Cache*).
- Commandes complémentaires : `ausearch -m avc -ts recent`, `ls -Zd /custom /custom/index.html`.

**5. Définir la règle de contexte et l'appliquer**

```bash
semanage fcontext -a -t httpd_sys_content_t '/custom(/.*)?'
restorecon -Rv /custom
```

| Élément | Rôle |
|---------|------|
| `semanage fcontext -a` | **ajoute** une règle de contexte dans la politique |
| `-t httpd_sys_content_t` | type à appliquer (contenu web lisible par Apache) |
| `'/custom(/.*)?'` | expression régulière : `/custom` **et** tout son contenu |
| `restorecon -Rv /custom` | **applique** la règle aux fichiers existants (récursif, verbeux) |

Sortie attendue :

```
Relabeled /custom from unconfined_u:object_r:default_t:s0 to unconfined_u:object_r:httpd_sys_content_t:s0
Relabeled /custom/index.html from unconfined_u:object_r:default_t:s0 to unconfined_u:object_r:httpd_sys_content_t:s0
```

- *(Le format exact du message peut varier selon la version.)*
- Si `semanage` est introuvable : `dnf install -y policycoreutils-python-utils`.

**6. Retester**

```bash
curl http://localhost/index.html
# This is serverX.
ls -Zd /custom
```

- La page s'affiche : le dossier a maintenant le type `httpd_sys_content_t`.
- Les règles `semanage fcontext` sont **persistantes** : elles survivent aux reboots et aux re-étiquetages (`restorecon`, `fixfiles`).

> 💡 Pour un accès depuis une autre machine : `firewall-cmd --permanent --add-service=http && firewall-cmd --reload`, et `systemctl enable --now httpd`.

---

## 4. Dépannage

| Symptôme | Cause probable | Solution |
|----------|----------------|----------|
| `403 Forbidden` alors que les droits Unix sont bons | mauvais type SELinux | `ls -Z`, `semanage fcontext` + `restorecon` |
| Le service marche en `permissive` mais pas en `enforcing` | politique bloque (AVC) | `ausearch -m avc`, corriger le contexte/booléen/port |
| Le contexte revient à l'ancien après un re-étiquetage | `chcon` utilisé (non persistant) | `semanage fcontext -a` + `restorecon` |
| `semanage: command not found` | paquet absent | `dnf install policycoreutils-python-utils` |
| Fichier **déplacé** (`mv`) garde son ancien contexte | `mv` conserve le contexte | `restorecon -Rv` (ou `cp` qui prend le contexte du dossier cible) |
| Service sur un **port non standard** refusé | port non autorisé pour le type | `semanage port -a -t http_port_t -p tcp 8080` |
| Besoin d'une autorisation précise (ex. FTP, Samba) | booléen désactivé | `getsebool -a \| grep ftp`, `setsebool -P ...` |
| Logs | | `/var/log/audit/audit.log`, `journalctl -t setroubleshoot`, `sealert -a` |

---

## 5. Questions de révision

1. Qu'est-ce que SELinux ? Différence entre **DAC** et **MAC** ?
2. Quels sont les trois modes SELinux ? Différence entre *permissive* et *disabled* ?
3. Différence entre le mode **courant** et le mode **par défaut** ? Comment changer chacun ?
4. À quoi sert `/etc/selinux/config` ? Que signifie `SELINUXTYPE=targeted` ?
5. Que contient un **contexte** SELinux ? Quel champ est le plus important ?
6. Pourquoi Apache renvoie-t-il **403** sur `/custom` alors que les droits Unix sont corrects ?
7. Comment confirmer qu'**SELinux** est responsable du blocage ?
8. Que font `semanage fcontext -a -t httpd_sys_content_t '/custom(/.*)?'` et `restorecon -Rv /custom` ?
9. Pourquoi ne suffit-il pas d'utiliser `chcon` ?
10. Que signifie l'expression `(/.*)?` dans la règle ?
11. Quel est le type par défaut d'un dossier créé dans `/` ? Pourquoi Apache ne peut-il pas le lire ?
12. Comment rendre un réglage de booléen SELinux **persistant** ?
