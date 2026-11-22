# TP – Authentification avec PAM (AlmaLinux)

**Distribution :** AlmaLinux 8 · **Fichiers :** `/etc/pam.d/*` · **Compte à bloquer :** `joshua` · **Comptes de test :** `joshua`, `user`, `root`

> ⚠️ L'énoncé contient surtout des **consignes** (avec quelques explications). Les points marqués **à vérifier** sont mon interprétation : à tester sur ta VM.
>
> 🛑 **Avant toute modification de PAM** : garde une **session root ouverte** dans un autre terminal. Une erreur peut bloquer **tout le monde**, root compris.

---

## 1. Théorie

### 1.1 Qu'est-ce que PAM ?

**PAM** (*Pluggable Authentication Modules*) est une couche qui sépare les **applications** (login, sshd, gdm, sudo…) de la **méthode d'authentification** (mot de passe Unix, LDAP, liste noire, quotas…). Chaque service a **son propre fichier** dans `/etc/pam.d/`.

### 1.2 Syntaxe d'une ligne

```
type   contrôle   module   arguments
auth   required   pam_listfile.so   onerr=succeed item=user sense=deny file=/etc/nologinusers
```

### 1.3 Les 4 types de gestion

| Type | Rôle |
|------|------|
| `auth` | **authentifier** l'utilisateur (mot de passe…) |
| `account` | vérifier que le compte est **autorisé** (expiré ? horaires ? `nologin` ?) |
| `password` | **changement** de mot de passe |
| `session` | actions à l'ouverture/fermeture de session (montages, limites, home) |

### 1.3 bis Les contrôles (flags)

| Contrôle | Si le module **réussit** | Si le module **échoue** |
|----------|--------------------------|--------------------------|
| `required` | continue ; | l'échec est **mémorisé**, la pile continue, **résultat final = échec** |
| `requisite` | continue | **arrêt immédiat** : échec |
| `sufficient` | **arrêt immédiat : succès** (si aucun `required` précédent n'a échoué) | ignoré, continue |
| `optional` | sans effet sur le résultat | sans effet sur le résultat |
| `include` | inclut les lignes d'un autre fichier du même type | |

- ⚠️ **L'ordre des lignes est capital** : la pile est lue **de haut en bas**.
- PAM est **très strict** sur la syntaxe (une faute = module introuvable ou erreur).

### 1.4 Modules à connaître

| Module | Rôle |
|--------|------|
| `pam_unix.so` | authentification par `/etc/passwd` / `/etc/shadow` |
| `pam_listfile.so` | autorise/refuse selon une **liste** (utilisateurs, shells, tty…) |
| `pam_nologin.so` | si `/etc/nologin` existe : **refuse les non-root** et affiche son contenu |
| `pam_deny.so` | refuse toujours |
| `pam_permit.so` | accepte toujours |
| `pam_securetty.so` | restreint les connexions root à certains terminaux |
| `pam_sss.so` | authentification via SSSD (LDAP…) |

### 1.5 Arguments de `pam_listfile`

| Argument | Sens |
|----------|------|
| `item=user` | on teste le **nom d'utilisateur** |
| `sense=deny` | si l'item est **dans** la liste → **refus** (`allow` = n'autorise que ceux de la liste) |
| `file=/etc/nologinusers` | fichier de la liste (un nom par ligne) |
| `onerr=succeed` | en cas d'**erreur interne** du module (ex. fichier illisible) → succès (`fail` = échec) |

Doc : `/usr/share/doc/pam/txts/README.pam_listfile`.

### 1.6 `system-auth` et `authselect`

- `/etc/pam.d/system-auth` est **inclus** par la plupart des services (login, sshd, gdm…). Une règle placée ici s'applique **partout**.
- Sous AlmaLinux 8, ce fichier est **géré par `authselect`** (lien symbolique vers `/etc/authselect/`) : une régénération du profil peut **écraser** une modification manuelle. Pour une modification durable : créer un profil personnalisé (`authselect create-profile ...`).

---

## 2. Plan du TP

0. Sauvegardes
1. Règle `pam_listfile` dans `/etc/pam.d/login`
2. Créer `joshua` et `user`, tester console / X
3. Faute volontaire dans la règle
4. Déplacer la règle dans `system-auth` (en bas, puis en haut)
5. Fichier `/etc/nologin`
6. Autres services concernés

---

## 3. Pratique

### Étape 0 : Sauvegardes

```bash
sudo cp -a /etc/pam.d/login        /etc/pam.d/login.bak
sudo cp -a /etc/pam.d/system-auth  /etc/pam.d/system-auth.bak
```

- `-a` conserve droits, propriétaire et liens.
- Restauration en cas d'erreur : `sudo cp -a /etc/pam.d/login.bak /etc/pam.d/login`.

### Étape 1 : Règle dans `/etc/pam.d/login`

Ajouter **juste sous les autres lignes `auth`** :

```
auth   required   pam_listfile.so onerr=succeed item=user sense=deny file=/etc/nologinusers
```

Lire : `/usr/share/doc/pam/txts/README.pam_listfile`.

### Étape 2 : Utilisateurs et liste

```bash
sudo useradd joshua
sudo useradd user
sudo passwd joshua
sudo passwd user
echo joshua | sudo tee /etc/nologinusers
```

- Les comptes de test **doivent avoir un mot de passe**, sinon on ne peut pas distinguer une panne PAM d'un compte sans mot de passe.
- Le fichier `/etc/nologinusers` contient un nom par ligne.

**Tests :**

1. **Console virtuelle** (`Ctrl+Alt+F3`) : `joshua` → **refusé** ; `user` → **accepté**.
   - Si **personne** ne peut se connecter : la ligne est mal écrite.
2. **Gestionnaire d'affichage X (GDM)** : `joshua` **peut encore se connecter**.
   - La règle est dans `/etc/pam.d/login`, qui ne concerne **que le service `login`** (consoles texte).
   - GDM utilise un autre fichier (`/etc/pam.d/gdm-password`). **Chaque service a sa propre configuration PAM.**

### Étape 3 : Erreur d'orthographe volontaire

Exemple : `pam_listfiel.so` à la place de `pam_listfile.so`.

**Conséquence :** **plus personne ne peut se connecter à ce service, même root.**

- PAM **ne trouve pas le module** : c'est une erreur de **chargement**.
- La ligne étant `required`, l'authentification échoue pour **tout le monde**.
- `onerr=succeed` **ne protège pas** ici : il ne s'applique qu'aux erreurs **internes** d'un module **correctement chargé**.
- **Correction** : rétablir l'orthographe (depuis la session root restée ouverte, ou depuis la sauvegarde `login.bak`).

### Étape 4 : Déplacer la règle dans `system-auth`

**4a.** Supprimer la ligne de `/etc/pam.d/login` et l'ajouter dans `/etc/pam.d/system-auth` **juste sous les autres entrées `auth`**.

**Test (console + X) : à vérifier sur ta VM.** Raisonnement attendu :

- Dans `system-auth`, `pam_unix.so` est `sufficient`. Quand il **réussit**, PAM s'arrête (succès) et **la ligne placée en dessous n'est jamais évaluée** → `joshua` **peut encore se connecter**.
- *Interprétation à confirmer par le test* : placée **après** `pam_unix`, la règle est **inefficace** pour un utilisateur dont le mot de passe est correct.

**4b.** Couper la ligne et la coller **au-dessus de toutes les autres lignes `auth`**.

**Test (console et X) : `joshua` est maintenant bloqué dans les deux cas.**

**Pourquoi ?**

- Notre ligne est évaluée **avant** `pam_unix`. Elle échoue pour `joshua` et, comme elle est `required`, l'échec est **mémorisé**.
- Quand `pam_unix` (`sufficient`) réussit ensuite, PAM ne peut plus conclure au succès, car un `required` a déjà échoué.
- `login` **et** GDM **incluent** `system-auth` : la règle s'applique donc **partout**, bien au-delà de `login` seul.

> 💡 **À retenir : l'ordre des lignes change le comportement.**

| Position de la règle `required` | Effet pour `joshua` |
|---------------------------------|---------------------|
| Avant `pam_unix` (`sufficient`) | **bloqué** (échec mémorisé) |
| Après `pam_unix` (`sufficient`) | **non évaluée** si `pam_unix` réussit → non bloqué |

### Étape 5 : Fichier `/etc/nologin`

```bash
echo "Système couramment indisponible --- PARTEZ" | sudo tee /etc/nologin
```

Ce fichier est lu par le module **`pam_nologin.so`** : s'il existe, les utilisateurs **non-root** sont refusés et **son contenu leur est affiché**.

| Cas | Résultat attendu |
|-----|------------------|
| Utilisateur normal sur **console** | **refusé**, message affiché |
| Utilisateur normal dans **X** | à tester (dépend du fichier PAM de GDM) |
| **root** (console et X) | normalement **autorisé** : `pam_nologin` laisse passer root pour qu'il puisse **réparer le système** |

> Supprimer ensuite `/etc/nologin` pour rétablir l'accès : `sudo rm /etc/nologin`.

### Étape 6 : Autres services

```bash
grep -l pam_listfile /etc/pam.d/*     # services utilisant pam_listfile
grep -l pam_nologin  /etc/pam.d/*     # services affectés par /etc/nologin
```

- **`pam_listfile`** : plusieurs services l'utilisent ; typiquement **vsftpd** (liste `ftpusers`, cf. TP FTP). Résultat exact : à vérifier avec la commande ci-dessus.
- **`/etc/nologin`** affecte **tous les services dont le fichier PAM charge `pam_nologin.so`** (typiquement `login`, `sshd`, etc.). Résultat exact : à vérifier avec `grep`.

---

## 4. Dépannage

| Symptôme | Cause probable | Solution |
|----------|----------------|----------|
| Plus personne ne se connecte (même root) | faute de syntaxe / module introuvable en `required` | corriger depuis la session root ouverte, ou restaurer le `.bak` (live CD / mode rescue en dernier recours) |
| La règle n'a aucun effet sur GDM / SSH | règle dans `login` uniquement | la mettre dans `system-auth` (ou le fichier du service) |
| La règle n'a aucun effet même dans `system-auth` | placée **après** `pam_unix` (`sufficient`) | la placer **au-dessus** |
| Modification de `system-auth` disparue | écrasée par `authselect` | profil authselect personnalisé |
| Tout le monde bloqué par `sense=deny` | fichier listant trop de noms / `sense` inversé | vérifier `/etc/nologinusers`, `sense=deny` |
| Un seul utilisateur « normal » bloqué par `/etc/nologin` | normal : `pam_nologin` | `rm /etc/nologin` |
| Logs | | `/var/log/secure`, `journalctl` |

---

## 5. Questions de révision

1. Qu'est-ce que PAM ? Quel problème résout-il ?
2. Quels sont les 4 types de gestion (`auth`, `account`, `password`, `session`) ?
3. Différence entre `required`, `requisite`, `sufficient`, `optional` ?
4. Pourquoi l'ordre des lignes est-il important ?
5. Que signifient `item=user`, `sense=deny`, `file=`, `onerr=succeed` dans `pam_listfile` ?
6. Pourquoi une règle dans `/etc/pam.d/login` n'affecte-t-elle pas GDM ?
7. Quelle est la conséquence d'une faute de frappe dans un module `required` ? Pourquoi `onerr=succeed` ne protège-t-il pas ?
8. Pourquoi placer la règle **avant** `pam_unix` dans `system-auth` ?
9. Pourquoi une règle dans `system-auth` s'applique-t-elle à plusieurs services ?
10. Que fait `pam_nologin.so` ? Pourquoi root est-il autorisé ?
11. Quelle précaution prendre avant de modifier PAM ?
12. Quel est le rôle d'`authselect` vis-à-vis de `system-auth` ?
