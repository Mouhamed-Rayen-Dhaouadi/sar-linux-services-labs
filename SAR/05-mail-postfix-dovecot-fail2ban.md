# TP 05 – Serveur de messagerie : Postfix, Dovecot et Fail2ban (Ubuntu 22.04)

**Domaine :** `nomprenom.com` (exemple du TP : `bouabidiimen.com`) · **Utilisateurs :** `nom` et `prenom` (exemple : `bouabidi` et `imen`) · **Stockage :** Maildir

> Dans ce document, remplace `bouabidi`, `imen` et `bouabidiimen.com` par **ton nom, ton prénom et ton domaine**.

---

## 1. Théorie

### 1.1 Les trois rôles d'une messagerie

| Rôle | Sigle | Logiciel du TP | Protocole / port |
|------|-------|----------------|------------------|
| Serveur d'**envoi / transfert** | **MTA** (*Mail Transfer Agent*) | **Postfix** | **SMTP** – 25 |
| Serveur d'**accès** aux boîtes | MDA/serveur IMAP-POP | **Dovecot** | **IMAP** – 143, **POP3** – 110 |
| **Client** de messagerie | **MUA** (*Mail User Agent*) | **Thunderbird** | utilise SMTP pour envoyer, IMAP/POP3 pour lire |

Chemin d'un message : MUA de l'expéditeur → (SMTP) → MTA → dépôt dans la boîte du destinataire → (IMAP/POP3) → MUA du destinataire.

### 1.2 SMTP, IMAP, POP3

- **SMTP** : envoie/relaie les messages entre serveurs (port 25 ; 587 = *submission* authentifiée, non utilisé ici).
- **POP3** : le client **télécharge** les messages (en général il les retire du serveur).
- **IMAP** : le client **consulte** les messages **sur le serveur** (dossiers, synchronisation multi-appareils).

### 1.3 Formats de stockage

- **mbox** : un seul fichier par boîte (`/var/mail/user`).
- **Maildir** : un **dossier** `~/Maildir/` avec `new/` (non lus), `cur/` (lus), `tmp/` ; **un fichier par message** → plus robuste et performant.

### 1.4 Paramètres Postfix à connaître

| Paramètre | Rôle |
|-----------|------|
| `myhostname` | nom FQDN du serveur (ex. `mail.bouabidiimen.com`) |
| `mydomain` / `myorigin` | domaine ajouté aux adresses locales (lu dans `/etc/mailname`) |
| `mydestination` | domaines pour lesquels le serveur **accepte** le courrier en local |
| `mynetworks` | réseaux de confiance autorisés à **relayer** |
| `home_mailbox` | emplacement de la boîte (`Maildir/` = format Maildir) |
| `inet_interfaces` | interfaces d'écoute |
| `recipient_delimiter` | séparateur d'extension (`user+tag@…`) |

### 1.5 Fail2ban

**Fail2ban** lit les journaux, détecte des échecs d'authentification répétés (force brute) et **bannit temporairement l'IP** en ajoutant une règle de pare-feu.

- **jail** : une surveillance (ex. `dovecot`) = filtre + action.
- `maxretry` : nombre d'échecs tolérés ; `findtime` : fenêtre de temps pour les compter ; `bantime` : durée du bannissement (s).
- On ne modifie **jamais** `jail.conf` (écrasé par les mises à jour) : on crée **`jail.local`**.

---

## 2. Plan du TP

1. Créer les utilisateurs
2. Installer Postfix
3. Configurer Postfix (`main.cf`)
4. Alias système
5. Pare-feu (UFW)
6. Tester Postfix avec `telnet`
7. Installer Dovecot
8. Configurer Dovecot
9. Tester Dovecot avec `telnet` (POP3)
10. Configurer Thunderbird
11. Fail2ban : installation, jail, simulation d'attaque, déblocage

---

## 3. Pratique

### Étape 1 – Utilisateurs

```bash
sudo useradd -m bouabidi
sudo passwd bouabidi
sudo useradd -m imen
sudo passwd imen
ls -la /home/
```

- `useradd -m` : crée l'utilisateur **et son répertoire personnel** (`/home/<nom>`), où sera `Maildir`.
- `passwd <nom>` : définit le mot de passe (utilisé par Dovecot pour l'authentification).
- `ls -la /home/` : vérifie la création des dossiers.

### Étape 2 – Installer Postfix

```bash
sudo apt update
sudo apt install postfix
```

Assistant interactif :
1. **Type de configuration** : `Internet Site`
2. **System mail name** : `bouabidiimen.com`

- *Internet Site* : le serveur envoie et reçoit directement en SMTP.

### Étape 3 – Configurer `/etc/postfix/main.cf`

```bash
sudo nano /etc/postfix/main.cf
```

```
myhostname = mail.bouabidiimen.com
myorigin = /etc/mailname
mydestination = $myhostname, localhost.$mydomain, localhost, $mydomain
relayhost =
mynetworks = 127.0.0.0/8 [::ffff:127.0.0.0]/104 [::1]/128 192.168.1.0/24
mailbox_size_limit = 0
recipient_delimiter = +
inet_interfaces = all
inet_protocols = all
home_mailbox = Maildir/
```

- `myhostname` : nom annoncé par le serveur (bannière `220 mail.bouabidiimen.com`).
- `myorigin = /etc/mailname` : domaine ajouté aux expéditeurs locaux.
- `mydestination` : ajouter **`$mydomain`** pour que `imen@bouabidiimen.com` soit reconnu comme **boîte locale**.
- `relayhost =` (vide) : pas de relais externe, livraison directe.
- `mynetworks` : ajouter **l'adresse réseau du serveur** (`192.168.1.0/24`) à la liste de confiance.
- `mailbox_size_limit = 0` : taille de boîte illimitée.
- `recipient_delimiter = +` : autorise `imen+facture@…`.
- `inet_interfaces = all` : écoute sur toutes les interfaces ; `inet_protocols = all` : IPv4 et IPv6.
- `home_mailbox = Maildir/` : le courrier est stocké dans `~/Maildir/` (**le `/` final indique Maildir**).

```bash
sudo systemctl restart postfix
```

💡 Vérifier les valeurs actives : `postconf -n` ; modifier une valeur : `sudo postconf -e 'home_mailbox = Maildir/'`.

### Étape 4 – Alias système

```bash
sudo nano /etc/aliases
```

```
postmaster: root
user1: bouabidi@bouabidiimen.com
user2: imen@bouabidiimen.com
```

```bash
sudo newaliases
```

- `/etc/aliases` : redirige un nom local vers une autre adresse ; `newaliases` **recompile** la base d'alias (indispensable après modification).
- Ici `user1` correspond à ton nom de famille et `user2` à ton prénom (alias de démonstration).

### Étape 5 – Pare-feu (UFW)

```bash
sudo apt install ufw
sudo ufw allow 25/tcp
sudo ufw allow 110/tcp
sudo ufw allow 143/tcp
sudo ufw allow OpenSSH      # si SSH utilisé
sudo ufw reload
sudo ufw enable
sudo ufw status verbose
sudo systemctl enable --now postfix
```

- `ufw allow 25/tcp` : SMTP ; `110/tcp` : POP3 ; `143/tcp` : IMAP. (587/tcp *submission* non requis ici.)
- `ufw reload` : recharge les règles ; `ufw enable` : active le pare-feu.
- `ufw status verbose` : liste des règles.
- `systemctl enable --now postfix` : démarre **et** active au boot.

⚠️ Si tu as suivi le TP DNS (`ufw disable`), le pare-feu devient actif ici : ouvre aussi le port **53** si le DNS doit rester joignable (`sudo ufw allow 53`).

### Étape 6 – Test SMTP avec `telnet`

```bash
sudo apt install telnet
telnet localhost smtp
```

Dialogue (les lignes à saisir sont celles sans code numérique) :

```
220 mail.bouabidiimen.com ESMTP Postfix (Ubuntu)
ehlo localhost
250-mail.bouabidiimen.com
 ... (liste des extensions)
mail from: bouabidi@bouabidiimen.com
250 2.1.0 Ok
rcpt to: imen@bouabidiimen.com
250 2.1.5 Ok
data
354 End data with <CR><LF>.<CR><LF>
test
.
250 2.0.0 Ok: queued as 3642480116
quit
221 2.0.0 Bye
```

- `ehlo localhost` : salutation SMTP étendue ; le serveur liste ses capacités.
- `mail from:` : **expéditeur** de l'enveloppe ; `rcpt to:` : **destinataire**.
- `data` : début du contenu ; on écrit le message puis une ligne contenant **uniquement un point (`.`)** pour terminer (le point est important).
- `quit` : ferme la session.

Vérifier la réception :

```bash
sudo su
cd /home/imen/Maildir/new/
ls
cat <nom-du-fichier>
exit
```

- Le message est un fichier dans `new/` (non lu) ; `cat` montre les en-têtes (`Return-Path`, `Delivered-To`, `Received`, `Message-Id`…) et le corps `test`.

### Étape 7 – Installer Dovecot

```bash
sudo apt-get install postfix dovecot-imapd dovecot-pop3d
```

- `dovecot-imapd` : service IMAP ; `dovecot-pop3d` : service POP3.

### Étape 8 – Configurer Dovecot

**1. `/etc/dovecot/dovecot.conf`**

```bash
sudo gedit /etc/dovecot/dovecot.conf
```

```
listen = *, ::
!include_try /usr/share/dovecot/protocols.d/*.protocol
```

- `listen = *, ::` : écoute sur toutes les adresses IPv4 (`*`) et IPv6 (`::`) — décommenter la ligne.
- `!include_try …protocols.d/*.protocol` : charge les protocoles installés (IMAP/POP3) ; `include_try` n'échoue pas si absent.

**2. `/etc/dovecot/conf.d/10-mail.conf`**

```
mail_location = maildir:~/Maildir
```

- Indique à Dovecot de lire les mails au format **Maildir** dans `~/Maildir` (`~` = répertoire personnel de l'utilisateur).
- ⚠️ Une **seule** directive `mail_location` doit être active.

**3. `/etc/dovecot/conf.d/10-auth.conf`**

```
disable_plaintext_auth = no
auth_mechanisms = plain login
```

- `disable_plaintext_auth = no` : autorise les mots de passe **en clair** sans chiffrement TLS (nécessaire car le TP n'active pas TLS). 🔒 En production : `yes` + TLS.
- `auth_mechanisms = plain login` : mécanismes d'authentification acceptés (`plain` standard, `login` pour certains clients comme Outlook).

**4. `/etc/dovecot/conf.d/10-master.conf`** (authentification partagée avec Postfix)

```
# Postfix smtp-auth
unix_listener /var/spool/postfix/private/auth {
    mode = 0666
    user = postfix
    group = postfix
}
```

- Crée une **socket Unix** par laquelle Postfix peut demander à Dovecot de vérifier un mot de passe (SASL).
- `mode = 0666` : droits de la socket ; `user`/`group = postfix` : propriétaire = l'utilisateur de Postfix.

**5. Redémarrer**

```bash
sudo systemctl restart dovecot
sudo systemctl status dovecot
sudo ss -tlnp | grep -E ':110|:143'
```

- Dovecot doit écouter sur 110 et 143.

### Étape 9 – Test POP3 avec `telnet`

```bash
telnet localhost pop3
```

```
+OK Dovecot ready.
user imen
+OK
pass <mot_de_passe_de_imen>
+OK Logged in.
list
+OK 1 messages:
1 428
.
retr 1
+OK 428 octets
 ... (contenu du message)
.
quit
+OK Logging out.
```

- `user` / `pass` : identification ; `list` : liste des messages (numéro + taille) ; `retr 1` : affiche le message 1 ; `quit` : quitte.
- Le nom d'utilisateur est le **login Linux** (`imen`), pas l'adresse complète.

### Étape 10 – Thunderbird

```bash
sudo apt install thunderbird -y
thunderbird &
```

Configuration **manuelle** pour chaque compte :

| Champ | Utilisateur 1 | Utilisateur 2 |
|-------|---------------|---------------|
| Nom complet | `bouabidi` | `imen` |
| Adresse e-mail | `bouabidi@bouabidiimen.com` | `imen@bouabidiimen.com` |
| Serveur entrant | IMAP, hôte `localhost` ou `192.168.1.100`, port **143**, sécurité **None** | idem |
| Serveur sortant | SMTP, même hôte, port **25**, sécurité **None** | idem |
| **Username** (entrant et sortant) | `bouabidi` | `imen` |

- Cliquer sur **Re-test** puis **Done**.
- Ajouter le 2ᵉ compte : *Settings → Account Settings → Account Actions → Add Mail Account*.
- Test : envoyer un mail de `imen` vers `bouabidi` et vérifier la réception dans la boîte de réception de `bouabidi`.

### Étape 11 – Fail2ban

**11.1 Installation**

```bash
sudo apt install fail2ban -y
```

**11.2 `/etc/fail2ban/jail.local`**

```bash
sudo nano /etc/fail2ban/jail.local
```

```ini
[DEFAULT]
ignoreself = false

[postfix-sasl]
enabled  = false
port     = smtp,submission
logpath  = /var/log/mail.log
maxretry = 3
bantime  = 600
findtime = 300

[dovecot]
enabled  = true
port     = pop3,pop3s,imap,imaps,submission
logpath  = /var/log/mail.log
maxretry = 3
bantime  = 600
findtime = 300
```

| Paramètre | Rôle |
|-----------|------|
| `maxretry = 3` | 3 échecs maximum avant bannissement |
| `findtime = 300` | on compte les échecs sur 300 s (5 min) |
| `bantime = 600` | bannissement de 600 s (10 min) |
| `ignoreself = false` | **ne pas** ignorer les IP de la machine → permet de tester depuis `localhost` |
| `enabled` | active ou non la jail |
| `logpath` | journal surveillé |

- `postfix-sasl` désactivée : l'authentification SMTP n'est pas dans le périmètre du TP.
- 💡 N'utiliser qu'**une** des deux méthodes pour tester depuis localhost : `ignoreself = false` **ou** ne pas lister `127.0.0.1/8 ::1` dans `ignoreip`.

**11.3 Démarrage et état**

```bash
sudo systemctl restart fail2ban
sudo fail2ban-client status
sudo fail2ban-client status dovecot
```

- `fail2ban-client status` : liste des jails actives ; `status dovecot` : détails (échecs en cours, total, IP bannies).
- État initial attendu : `Currently failed: 0`, `Currently banned: 0`.

**11.5 Simulation d'attaque par force brute**

```bash
for i in 1 2 3 4 5 6; do
  (echo "user imen"; sleep 0.3; echo "pass wrong$i"; sleep 0.3; echo "quit") | timeout 5 telnet localhost pop3 2>/dev/null
  sleep 2
done
```

- Boucle de 6 tentatives avec un mauvais mot de passe (`wrong$i`).
- `(echo …; sleep …)` : envoie les commandes POP3 avec de courtes pauses ; `| timeout 5 telnet …` : les passe à `telnet` (arrêt après 5 s) ; `2>/dev/null` masque les erreurs.

```bash
sudo tail -20 /var/log/mail.log
```

Lignes attendues (une par tentative) :

```
dovecot: pop3-login: Disconnected: Aborted login by logging out (auth failed, 1 attempts in X secs): user=<imen>, method=PLAIN, rip=127.0.0.1, lip=127.0.0.1
```

- `rip` = *remote IP* (client) ; `lip` = *local IP* (serveur). Le **cumul de ces lignes** dans `findtime` déclenche le ban après `maxretry`.

**11.6 Constater le bannissement**

```bash
sudo fail2ban-client status dovecot
```

Attendu :

```
|- Currently failed: 1
|- Total failed:     4
`- Currently banned: 1  /  Banned IP list: 127.0.0.1
```

**Débannir** (après le test) :

```bash
sudo fail2ban-client set dovecot unbanip 127.0.0.1
```

**11.7 Retour à un réglage de production**

Remettre `ignoreself = true` dans `jail.local`, puis :

```bash
sudo systemctl restart fail2ban
sudo fail2ban-client status dovecot
```

---

## 4. Dépannage

| Symptôme | Cause | Solution |
|----------|-------|----------|
| `rcpt to` refusé (`Relay access denied`) | domaine absent de `mydestination` | ajouter `$mydomain` |
| Aucun fichier dans `Maildir/new` | `home_mailbox` absent | `home_mailbox = Maildir/` + restart |
| Thunderbird : échec d'authentification | mauvais *username* | utiliser `imen`, pas l'e-mail complet |
| `Plaintext authentication disallowed` | `disable_plaintext_auth = yes` | passer à `no` (labo) |
| Dovecot ne démarre pas | faute de config | `sudo doveconf -n`, `journalctl -u dovecot` |
| Fail2ban ne bannit pas | mauvais `logpath` / `ignoreself` | vérifier `/var/log/mail.log`, `fail2ban-client status dovecot` |
| Logs Postfix | | `sudo tail -f /var/log/mail.log` |

---

## 5. Questions de révision

1. Différence entre MTA, serveur IMAP/POP3 et MUA ?
2. Différence entre IMAP et POP3 ? Quels ports ?
3. Pourquoi Maildir plutôt que mbox ? Quelle directive Postfix et quelle directive Dovecot l'activent ?
4. Rôle de `mydestination` et `mynetworks` ?
5. Que fait `newaliases` ?
6. Comment fonctionne Fail2ban (jail, `maxretry`, `findtime`, `bantime`) ?
7. Pourquoi ne pas modifier `jail.conf` ?
8. Pourquoi `disable_plaintext_auth = no` est-il à éviter en production ?
