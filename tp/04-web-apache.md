# TP 04 – Serveur Web Apache sous Ubuntu 22.04 (virtual hosts et sécurisation)

**Prérequis :** Ubuntu 22.04, accès Internet, droits root. **Réseau :** `192.168.1.100` (principale), `.101` et `.102` (alias).

---

## 1. Théorie

### 1.1 Web et HTTP

- Le **Web** = documents hypertextes accessibles via Internet par un **navigateur** (client).
- **HTTP** (*HyperText Transfer Protocol*) : protocole applicatif **requête/réponse** du modèle client-serveur. Le client envoie une requête (`GET /index.html`), le serveur renvoie une réponse (code `200 OK`, `403 Forbidden`, `404 Not Found`…).
- **Ports** : HTTP = **80**, HTTPS = **443**. Un autre port s'indique dans l'URL : `http://www.esprit4.com:8080/`. Un port identifie une application sur une machine.

### 1.2 Apache et les hôtes virtuels

**Apache HTTP Server** (`apache2` sur Debian/Ubuntu) peut héberger **plusieurs sites sur un seul serveur** grâce aux **virtual hosts**. Trois méthodes :

| Type | Critère de choix du site | Exemple du TP |
|------|--------------------------|---------------|
| **Basé sur le nom** | en-tête HTTP `Host:` (nom demandé) | `esprit1.com`, `esprit2.com` → même IP `192.168.1.100:80` |
| **Basé sur l'IP** | adresse IP de destination | `esprit3.com` → `192.168.1.101:80` |
| **Basé sur le port** | numéro de port | `esprit4.com` → `192.168.1.100:8080` |

### 1.3 Organisation Apache sous Debian/Ubuntu

| Chemin | Rôle |
|--------|------|
| `/etc/apache2/apache2.conf` | configuration principale |
| `/etc/apache2/ports.conf` | ports d'écoute (`Listen`) |
| `/etc/apache2/sites-available/` | sites **disponibles** (fichiers `.conf`) |
| `/etc/apache2/sites-enabled/` | sites **activés** (liens symboliques) |
| `/etc/apache2/conf-available/security.conf` | réglages de sécurité |
| `/var/www/` | racines web (*DocumentRoot*) |
| `/var/log/apache2/` | journaux `access.log` et `error.log` |

Outils : `a2ensite` / `a2dissite` (activer/désactiver un site), `apache2ctl configtest` (vérifier la syntaxe).

### 1.4 Résolution de noms pour les tests

Sans vrai DNS, le fichier **`/etc/hosts`** associe un nom à une IP localement ; il est consulté **avant** le DNS.

---

## 2. Plan du TP

- **I.** Hébergement basé sur le **nom** (esprit1 / esprit2)
- **II.** Hébergement basé sur l'**IP** (esprit3)
- **III.** Hébergement basé sur le **port** (esprit4)
- **IV.** **Sécurisation** : ports ouverts, UFW, masquer la version, désactiver le listage, restreindre un répertoire

---

## 3. Pratique

### 0. Installation

```bash
sudo apt update
sudo apt install apache2
```

- `apt update` : actualise la liste des paquets ; `apt install apache2` : installe Apache (le service démarre automatiquement).

### I. Hébergement virtuel basé sur le nom

#### I.1 Arborescence

```bash
sudo mkdir -p /var/www/esprit1.com/public_html
sudo mkdir -p /var/www/esprit2.com/public_html
```

- `mkdir -p` : crée le dossier **et** les parents manquants. `public_html` contiendra les fichiers du site.

#### I.2 Permissions

```bash
sudo chown -R $USER:$USER /var/www/esprit1.com/public_html
sudo chown -R $USER:$USER /var/www/esprit2.com/public_html
sudo chmod -R 755 /var/www
```

- `chown -R $USER:$USER` : rend votre utilisateur propriétaire (récursif) pour pouvoir éditer les pages.
- `chmod -R 755` : propriétaire `rwx`, groupe et autres `r-x` → Apache (utilisateur `www-data`) peut **lire** et traverser les dossiers.

#### I.3 Pages de démonstration

```bash
sudo gedit /var/www/esprit1.com/public_html/index.html
```

```html
<html>
  <head>
    <title>Welcome to Esprit1.com!</title>
  </head>
  <body>
    <h1>Success! The esprit1.com virtual host is working!</h1>
  </body>
</html>
```

Même chose pour `esprit2.com` (adapter le texte).

#### I.4 Fichiers de virtual host

```bash
sudo cp /etc/apache2/sites-available/000-default.conf /etc/apache2/sites-available/esprit1.com.conf
sudo gedit /etc/apache2/sites-available/esprit1.com.conf
```

```apache
<VirtualHost 192.168.1.100:80>
    ServerName www.esprit1.com
    ServerAdmin admin@esprit1.com
    DocumentRoot /var/www/esprit1.com/public_html
    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```

- `<VirtualHost IP:port>` : adresse et port auxquels ce site répond.
- `ServerName` : nom du site (comparé à l'en-tête `Host:`).
- `ServerAdmin` : e-mail affiché dans certaines pages d'erreur.
- `DocumentRoot` : dossier racine du site.
- `ErrorLog` / `CustomLog` : fichiers de journaux (`${APACHE_LOG_DIR}` = `/var/log/apache2`) ; `combined` = format détaillé.

Faire de même pour `esprit2.com.conf` (`ServerName www.esprit2.com`, `DocumentRoot /var/www/esprit2.com/public_html`).

#### I.5 Activation

```bash
sudo a2ensite esprit1.com.conf
sudo a2ensite esprit2.com.conf
sudo a2dissite 000-default.conf
sudo systemctl reload apache2
```

- `a2ensite` : crée un lien dans `sites-enabled/` → active le site.
- `a2dissite 000-default.conf` : désactive le site par défaut pour éviter qu'il serve les requêtes inconnues.
- `systemctl reload apache2` : recharge la configuration **sans couper** les connexions (plus doux que `restart`).

#### I.6 Fichier hosts

```bash
sudo gedit /etc/hosts
```

```
192.168.1.100 www.esprit1.com
192.168.1.100 www.esprit2.com
```

Les deux noms pointent vers la **même IP** : c'est l'en-tête `Host:` qui permet à Apache de choisir le bon site.

#### I.7 Test

Navigateur : `http://www.esprit1.com` puis `http://www.esprit2.com` → « Success! … ». Test en ligne de commande : `curl http://www.esprit1.com`.

---

### II. Hébergement basé sur l'IP

```bash
ip a
sudo netplan apply        # si les alias n'apparaissent pas
```

- `ip a` : liste des adresses (`192.168.1.100`, `.101`, `.102` doivent figurer sur `ens33`) ; les alias sont définis dans Netplan (voir TP DNS).

Créer `/etc/apache2/sites-available/esprit3.com.conf` :

```apache
<VirtualHost 192.168.1.101:80>
    ServerName www.esprit3.com
    ServerAdmin admin@esprit3.com
    DocumentRoot /var/www/esprit3.com/public_html
    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```

Puis :

```bash
sudo mkdir -p /var/www/esprit3.com/public_html
sudo gedit /var/www/esprit3.com/public_html/index.html     # page de test
sudo a2ensite esprit3.com.conf
sudo systemctl reload apache2
echo "192.168.1.101 www.esprit3.com" | sudo tee -a /etc/hosts
```

- Ici le site est choisi selon l'**IP de destination** `192.168.1.101`.
- `tee -a` : **ajoute** (*append*) à la fin du fichier avec les droits root.

---

### III. Hébergement basé sur le port

```bash
sudo gedit /etc/apache2/ports.conf
```

Ajouter :

```apache
Listen 8080
```

- `Listen 8080` : Apache écoute **aussi** sur le port 8080 (sans cela, le VirtualHost `:8080` ne reçoit rien).

`/etc/apache2/sites-available/esprit4.com.conf` :

```apache
<VirtualHost 192.168.1.100:8080>
    ServerName www.esprit4.com
    ServerAdmin admin@esprit4.com
    DocumentRoot /var/www/esprit4.com/public_html
    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```

```bash
sudo mkdir -p /var/www/esprit4.com/public_html
sudo gedit /var/www/esprit4.com/public_html/index.html
sudo a2ensite esprit4.com.conf
sudo systemctl restart apache2
echo "192.168.1.100 www.esprit4.com" | sudo tee -a /etc/hosts
```

Test : `http://www.esprit4.com:8080`.

⚠️ `Listen` ne s'applique qu'après un **`restart`** (un `reload` suffit souvent, mais le redémarrage est le plus sûr). Si UFW est actif, autoriser aussi `8080/tcp`.

---

### IV. Sécurisation d'Apache

#### IV.1 Ports ouverts

```bash
sudo ss -tulpn
# ou
sudo netstat -tulpn
```

- `ss`/`netstat` : `-t` TCP, `-u` UDP, `-l` en écoute, `-p` processus, `-n` numérique.
- Réponses attendues : Apache écoute sur le port **80** (et 8080 si configuré) ; le processus est **`apache2`**.

#### IV.2 Pare-feu UFW

```bash
sudo ufw status
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw allow OpenSSH
sudo ufw enable
sudo ufw status verbose
```

- `ufw status` : état du pare-feu.
- `ufw allow 80/tcp` : autorise HTTP ; `443/tcp` : HTTPS ; `OpenSSH` : profil d'application pour SSH (port 22). Équivalent : `sudo ufw allow 'Apache Full'`.
- `ufw enable` : active le pare-feu (**autoriser SSH avant** si on travaille à distance !).
- `ufw status verbose` : règles détaillées (politique par défaut : `deny (incoming)`).

🔒 Seuls les services **strictement nécessaires** doivent être accessibles. Si `OpenSSH` n'apparaît pas dans `ufw app list`, `openssh-server` n'est pas installé : n'ouvrir `22/tcp` que si l'accès SSH est réellement nécessaire.

#### IV.3 Masquer la version

```bash
sudo nano /etc/apache2/conf-available/security.conf
```

Remplacer :

```apache
ServerTokens OS        →   ServerTokens Prod
ServerSignature On     →   ServerSignature Off
```

```bash
sudo systemctl restart apache2
curl -I http://localhost
```

- `ServerTokens Prod` : l'en-tête `Server:` n'indique plus que « Apache » (sans version ni OS).
- `ServerSignature Off` : supprime la ligne de signature en bas des pages d'erreur.
- `curl -I` : n'affiche que les **en-têtes** de réponse → vérifier `Server: Apache`.

Pourquoi : la version et l'OS aident un attaquant à cibler des vulnérabilités connues.

#### IV.4 Désactiver l'affichage des répertoires

```bash
mkdir /var/www/esprit1.com/public_html/images     # sans index.html !
sudo nano /etc/apache2/sites-available/esprit1.com.conf
```

```apache
<VirtualHost 192.168.1.100:80>
    ServerName www.esprit1.com
    ServerAdmin admin@esprit1.com
    DocumentRoot /var/www/esprit1.com/public_html

    <Directory /var/www/esprit1.com/public_html>
        Options -Indexes +FollowSymLinks
        AllowOverride None
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```

- `Options -Indexes` : **interdit** le listage automatique d'un dossier sans `index.html` ; `+FollowSymLinks` : suit les liens symboliques.
- `AllowOverride None` : ignore les fichiers `.htaccess` (plus sûr et plus rapide).
- `Require all granted` : autorise l'accès à tous (contrôle d'accès Apache 2.4).

```bash
sudo apache2ctl configtest
sudo systemctl restart apache2
```

- `configtest` : vérifie la syntaxe (« Syntax OK »).

Test : `http://www.esprit1.com/images` → **403 Forbidden** (au lieu de la liste des fichiers).

#### IV.5 Restreindre un répertoire privé

```bash
mkdir /var/www/esprit1.com/public_html/private
echo "Private Area" > /var/www/esprit1.com/public_html/private/index.html
```

Ajouter dans le VirtualHost :

```apache
<Directory /var/www/esprit1.com/public_html/private>
    Require all denied
</Directory>
```

```bash
sudo apache2ctl configtest
sudo systemctl restart apache2
```

Test : `http://www.esprit1.com/private` → **403 Forbidden**.

- `Require all denied` : refuse l'accès à **tous** (pour n'autoriser qu'un réseau : `Require ip 192.168.1.0/24`).

---

## 4. Dépannage

| Symptôme | Cause | Solution |
|----------|-------|----------|
| « Site introuvable » | `/etc/hosts` incomplet | ajouter la ligne IP + nom |
| Toujours la page par défaut Apache | site non activé / `000-default` actif | `a2ensite`, `a2dissite 000-default.conf`, reload |
| `403 Forbidden` sur un site normal | permissions ou `Require` | `chmod 755`, vérifier `<Directory>` |
| `apache2ctl configtest` : erreur | faute de syntaxe | lire la ligne signalée |
| Port 8080 inaccessible | `Listen 8080` absent ou UFW | ajouter `Listen`, `ufw allow 8080/tcp` |
| Logs | | `sudo tail -f /var/log/apache2/error.log` |

---

## 5. Questions de révision

1. Quelles sont les trois méthodes d'hébergement virtuel et quel critère utilise chacune ?
2. Pourquoi deux sites peuvent-ils partager `192.168.1.100:80` ?
3. Différence entre `reload` et `restart` ?
4. Rôle de `a2ensite` et `sites-enabled` ?
5. Que font `ServerTokens Prod` et `Options -Indexes` ? Pourquoi sont-ils utiles ?
6. Comment interdire l'accès à un dossier ? Comment l'autoriser uniquement à un réseau ?
7. Quelle commande vérifie la syntaxe de la configuration Apache ?
