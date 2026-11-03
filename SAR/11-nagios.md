# TP 11 – Installation et configuration de Nagios Core sur Ubuntu 22.04

**Objectif :** installer Nagios Core avec un script, ouvrir l'interface web, puis surveiller des **machines** (hôtes) et des **services**.
**Personnalisation :** remplace `prenom-nom` par tes prénom et nom (minuscules, sans accent ni espace, ex. `samibenali`).

---

## 1. Théorie

### 1.1 La supervision

La **supervision** (*monitoring*) surveille en continu l'état des machines, des services et du réseau, et **alerte** l'administrateur avant que les utilisateurs ne constatent la panne. **Nagios Core** est l'un des outils libres les plus répandus.

### 1.2 Architecture de Nagios

| Composant | Rôle |
|-----------|------|
| **Démon `nagios`** | planifie les vérifications, évalue les états, déclenche les notifications |
| **Plugins** (`/usr/local/nagios/libexec`) | petits programmes qui **testent** quelque chose (`check_ping`, `check_http`, `check_ssh`…) |
| **Interface web** (CGI via **Apache**) | affichage de l'état (`/nagios`), protégée par `htpasswd` |
| **Fichiers de configuration** (`/usr/local/nagios/etc/`) | décrivent les objets à surveiller |

Fonctionnement : Nagios exécute un **plugin**, lit son **code de retour** et affiche l'état.

### 1.3 États

| Objet | États possibles |
|-------|-----------------|
| **Hôte** (machine) | `UP`, `DOWN`, `UNREACHABLE` |
| **Service** | `OK` (code 0), `WARNING` (1), `CRITICAL` (2), `UNKNOWN` (3) |

- **Soft / Hard state** : après un premier échec, l'état est *soft* ; il devient *hard* (et déclenche la notification) après **`max_check_attempts`** échecs consécutifs.

### 1.4 Objets de configuration

| Objet | Rôle |
|-------|------|
| `host` | une machine à surveiller |
| `service` | un test sur une machine (PING, HTTP, SSH…) |
| `command` | définition d'une commande de vérification (utilise un plugin) |
| `contact` / `contactgroup` | qui est prévenu |
| `timeperiod` | plages horaires (`24x7`, `workhours`) |
| `hostgroup` / `servicegroup` | regroupements |
| **Template** (`use …`) | modèle dont un objet **hérite** (ex. `linux-server`, `generic-service`) |

### 1.5 Chemins importants

| Chemin | Contenu |
|--------|---------|
| `/usr/local/nagios/etc/nagios.cfg` | configuration principale |
| `/usr/local/nagios/etc/objects/` | objets d'exemple (`contacts.cfg`, `templates.cfg`, `commands.cfg`, `localhost.cfg`…) |
| `/usr/local/nagios/etc/servers/` | **nos** objets (`host.cfg`, `services.cfg`) ; dossier activé par `cfg_dir` |
| `/usr/local/nagios/etc/htpasswd.users` | comptes de l'interface web |
| `/usr/local/nagios/var/` | état, journaux |

---

## 2. Plan du TP

1. **Installation automatique** : préparer le système → créer le script → l'exécuter → vérifier
2. **Configuration** : personnaliser le contact → ajouter un hôte → surveiller un service (PING)

---

## 3. Pratique

### Étape 1 – Préparer le système

```bash
sudo apt update
sudo apt upgrade
sudo apt install build-essential apache2 apache2-utils php libgd-dev libapache2-mod-php libperl-dev libssl-dev daemon wget
ip a
```

- `apt update` : actualise la liste des paquets (aucune erreur de dépôt ne doit persister) ; `apt upgrade` : met à niveau.
- `build-essential` : compilateur et outils (`gcc`, `make`) car Nagios est **compilé** ; `apache2` + `apache2-utils` (`htpasswd`) : serveur web et outils ; `php` + `libapache2-mod-php` : PHP pour Apache ; `libgd-dev` : bibliothèque graphique (cartes) ; `libperl-dev`, `libssl-dev` : Perl et SSL ; `daemon`, `wget` : outils (démon, téléchargement).
- `ip a` : relever l'**adresse IP** de la machine.

### Étape 2 – Créer le script

```bash
nano install_nagios.sh
```

Coller le contenu (reconstitué ci-dessous), enregistrer (`Ctrl+O`, Entrée), quitter (`Ctrl+X`).

```bash
#!/bin/bash
# Installation automatique de Nagios Core sur Ubuntu 22.04
# Utilisation : sudo bash install_nagios.sh [mot_de_passe_nagiosadmin]
set -e
export DEBIAN_FRONTEND=noninteractive

NAGIOS_VERSION="4.5.9"
PLUGINS_VERSION="2.4.10"
ADMIN_PASS="${1:-nagios}"
CFG=/usr/local/nagios/etc

if [ "$(id -u)" -ne 0 ]; then
  echo "Erreur : lancez ce script avec sudo."
  exit 1
fi

echo "[1/7] Installation des prérequis..."
apt install -y build-essential apache2 apache2-utils php libgd-dev libapache2-mod-php libperl-dev libssl-dev daemon wget

echo "[2/7] Création de l'utilisateur et des groupes..."
id nagios >/dev/null 2>&1 || useradd nagios
getent group nagcmd >/dev/null || groupadd nagcmd
usermod -a -G nagcmd nagios
usermod -a -G nagios,nagcmd www-data

echo "[3/7] Téléchargement et compilation de Nagios Core $NAGIOS_VERSION..."
cd /tmp
wget -q "https://github.com/NagiosEnterprises/nagioscore/releases/download/nagios-$NAGIOS_VERSION/nagios-$NAGIOS_VERSION.tar.gz"
tar -xzf "nagios-$NAGIOS_VERSION.tar.gz"
cd "nagios-$NAGIOS_VERSION"
./configure --with-nagios-group=nagios --with-command-group=nagcmd
make all

echo "[4/7] Installation de Nagios et de l'interface web..."
make install
make install-commandmode
make install-init
make install-config
install -c -m 644 sample-config/httpd.conf /etc/apache2/sites-available/nagios.conf
cp -R contrib/eventhandlers/ /usr/local/nagios/libexec/
chown -R nagios:nagios /usr/local/nagios/libexec/eventhandlers

echo "[5/7] Installation des plugins Nagios $PLUGINS_VERSION..."
cd /tmp
wget -q "https://nagios-plugins.org/download/nagios-plugins-$PLUGINS_VERSION.tar.gz"
tar -xzf "nagios-plugins-$PLUGINS_VERSION.tar.gz"
cd "nagios-plugins-$PLUGINS_VERSION"
./configure --with-nagios-user=nagios --with-nagios-group=nagios --with-openssl
make
make install

echo "[6/7] Configuration de Nagios et d'Apache..."
mkdir -p $CFG/servers
sed -i 's|^#cfg_dir=/usr/local/nagios/etc/servers|cfg_dir=/usr/local/nagios/etc/servers|' $CFG/nagios.cfg
grep -q '^cfg_dir=/usr/local/nagios/etc/servers' $CFG/nagios.cfg || echo 'cfg_dir=/usr/local/nagios/etc/servers' >> $CFG/nagios.cfg
ln -sf /etc/apache2/sites-available/nagios.conf /etc/apache2/sites-enabled/nagios.conf
a2enmod rewrite cgi
htpasswd -bc $CFG/htpasswd.users nagiosadmin "$ADMIN_PASS"

echo "[7/7] Démarrage des services..."
systemctl enable nagios apache2
systemctl restart nagios
systemctl restart apache2

IP=$(hostname -I | awk '{print $1}')
echo ""
echo "Installation terminée."
echo "Interface web : http://$IP/nagios"
echo "Utilisateur : nagiosadmin / Mot de passe : $ADMIN_PASS"
```

**Explication du script**

| Partie | Rôle |
|--------|------|
| `set -e` | le script **s'arrête à la première erreur** |
| `export DEBIAN_FRONTEND=noninteractive` | `apt` ne pose aucune question |
| `ADMIN_PASS="${1:-nagios}"` | mot de passe = 1ᵉʳ argument, sinon `nagios` par défaut |
| `id -u -ne 0` | vérifie qu'on est **root** (sinon message d'erreur) |
| **[2/7]** `useradd nagios`, `groupadd nagcmd` | utilisateur système `nagios` et groupe `nagcmd` (commandes externes) ; `usermod -a -G` ajoute `nagios` et `www-data` (Apache) aux groupes pour qu'Apache puisse envoyer des commandes à Nagios |
| **[3/7]** `wget`, `tar -xzf` | télécharge et décompresse les sources |
| `./configure --with-nagios-group=… --with-command-group=…` | prépare la compilation (groupes utilisés) |
| `make all` | compile |
| **[4/7]** `make install` | installe le programme ; `install-commandmode` : fichier de commandes externe ; `install-init` : script/service **systemd** ; `install-config` : fichiers de configuration d'exemple |
| `install -c -m 644 … nagios.conf` | copie la configuration **Apache** de Nagios (CGI et accès web) |
| `cp -R contrib/eventhandlers` | gestionnaires d'événements d'exemple |
| **[5/7]** plugins | compile et installe les **plugins** (`check_ping`, `check_http`…) |
| **[6/7]** `mkdir -p …/servers` + `sed`/`grep … \|\| echo` | crée le dossier et **active `cfg_dir=…/servers`** dans `nagios.cfg` (décommente la ligne, ou l'ajoute si absente) |
| `ln -sf … sites-enabled/` | active le site Apache de Nagios |
| `a2enmod rewrite cgi` | active les modules Apache `rewrite` et `cgi` (nécessaires à l'interface) |
| `htpasswd -bc … nagiosadmin "$ADMIN_PASS"` | crée le fichier de comptes avec l'utilisateur `nagiosadmin` ; `-b` mot de passe en argument, `-c` crée le fichier |
| **[7/7]** `systemctl enable/restart` | démarre et active Nagios et Apache au boot |
| `hostname -I \| awk '{print $1}'` | récupère la 1ʳᵉ adresse IP pour afficher l'URL finale |

🔒 Le mot de passe passé en argument reste visible dans l'historique du shell et la liste des processus : acceptable en TP, pas en production.

### Étape 3 – Lancer le script

```bash
sudo bash install_nagios.sh MonMotDePasse
```

- Remplace `MonMotDePasse` par le mot de passe de `nagiosadmin` (sans argument : `nagios`).
- Durée : quelques minutes (compilation). La progression s'affiche de **[1/7]** à **[7/7]**.

### Étape 4 – Vérifier

```bash
sudo systemctl status nagios
```

- Doit afficher **active (running)** en vert (quitter avec `q`).
- Ouvrir `http://adresse_ip_du_serveur/nagios` → identifiant `nagiosadmin` et le mot de passe choisi. Au début, seule la machine **`localhost`** est visible.
- Si un pare-feu UFW est actif : `sudo ufw allow 80/tcp`.

### Étape 5 – Personnaliser le contact

```bash
sudo nano /usr/local/nagios/etc/objects/contacts.cfg
```

Dans le bloc `define contact` de `nagiosadmin`, modifier **seulement** :

```
alias    Prénom NOM
email    votre.email@exemple.com
```

- `contact_name` doit **rester `nagiosadmin`** : il est lié au compte de connexion à l'interface web.
- Le contact est la personne **notifiée** en cas de problème.

### Étape 6 – Ajouter une machine à surveiller

```bash
sudo nano /usr/local/nagios/etc/servers/host.cfg
```

```
define host {
    use                    linux-server
    host_name              pc-prenom-nom
    alias                  PC de Prénom NOM
    address                192.168.1.100
    max_check_attempts     3
    check_period           24x7
    notification_interval  30
    notification_period    24x7
}
```

| Champ | Signification |
|-------|---------------|
| `use` | **template** dont on hérite (`linux-server`, défini dans `templates.cfg`) |
| `host_name` | nom **unique** dans Nagios (`pc-` + prénom + nom, minuscules) |
| `alias` | description lisible |
| `address` | adresse IP (ou nom DNS) à tester — à adapter à ton réseau |
| `max_check_attempts` | nombre d'échecs avant de déclarer l'hôte **DOWN** (état *hard*) |
| `check_period` | plage horaire des vérifications (`24x7` = en permanence) |
| `notification_interval` | minutes entre deux alertes tant que le problème dure |
| `notification_period` | plage horaire où les alertes sont autorisées |

Pour une 2ᵉ machine : ajouter un **autre bloc `define host`** avec un autre `host_name` et une autre `address`.

**Vérifier la syntaxe puis recharger**

```bash
sudo /usr/local/nagios/bin/nagios -v /usr/local/nagios/etc/nagios.cfg
sudo systemctl reload nagios
```

- `nagios -v <fichier>` : **mode vérification** (*verify*) : contrôle toute la configuration ; doit finir par **`Total Warnings: 0`** et **`Total Errors: 0`**.
- `systemctl reload nagios` : recharge la configuration sans arrêter le démon.
- **Vérification** : menu **Hosts** à gauche → la machine apparaît avec l'état **UP**.

### Étape 7 – Surveiller un service (PING)

```bash
sudo nano /usr/local/nagios/etc/servers/services.cfg
```

```
define service {
    use                  generic-service
    host_name            pc-prenom-nom
    service_description  PING
    check_command        check_ping!100.0,20%!500.0,60%
}
```

- `use generic-service` : template de service (valeurs par défaut).
- `host_name` : doit être **le nom d'un hôte déjà défini** dans `host.cfg`.
- `service_description` : nom affiché dans l'interface.
- `check_command check_ping!100.0,20%!500.0,60%` : la commande `check_ping` (définie dans `commands.cfg`) avec des **arguments séparés par `!`** :
  - `$ARG1$ = 100.0,20%` → **WARNING** si délai ≥ 100 ms **ou** perte ≥ 20 % ;
  - `$ARG2$ = 500.0,60%` → **CRITICAL** si délai ≥ 500 ms **ou** perte ≥ 60 %.

```bash
sudo /usr/local/nagios/bin/nagios -v /usr/local/nagios/etc/nagios.cfg
sudo systemctl reload nagios
```

- **Vérification** : menu **Services** → le service **PING** apparaît en **vert (OK)**.

### Bonus – Tester un plugin à la main et simuler une panne

```bash
/usr/local/nagios/libexec/check_ping -H 192.168.1.100 -w 100.0,20% -c 500.0,60% -p 5
echo $?
```

- `-H` hôte, `-w` seuils *warning*, `-c` seuils *critical*, `-p 5` nombre de paquets ; `echo $?` affiche le **code de retour** (0 OK, 1 WARNING, 2 CRITICAL, 3 UNKNOWN). Très utile pour déboguer.

Ajouter d'autres services (exemples) :

```
define service {
    use                  generic-service
    host_name            pc-prenom-nom
    service_description  HTTP
    check_command        check_http
}

define service {
    use                  generic-service
    host_name            pc-prenom-nom
    service_description  SSH
    check_command        check_ssh
}
```

- Arrêter `apache2` sur la machine surveillée : le service HTTP passe en **CRITICAL** (d'abord *soft*, puis *hard* après `max_check_attempts` essais).

---

## 4. Dépannage

| Symptôme | Cause probable | Solution |
|----------|----------------|----------|
| `Total Errors` > 0 à la vérification | faute de syntaxe / objet inconnu | lire la ligne signalée, corriger |
| `Template 'linux-server' not found` | `templates.cfg` non chargé | vérifier `cfg_file=…/templates.cfg` dans `nagios.cfg` |
| `unable to find host` pour un service | `host_name` ne correspond à aucun hôte | même nom exact que dans `host.cfg` |
| Hôte jamais ajouté | `cfg_dir=…/servers` non activé | vérifier la ligne dans `nagios.cfg` |
| Page web : *Could not read host and service status* | démon Nagios arrêté | `systemctl status nagios`, `journalctl -u nagios` |
| Erreur 500 / page blanche | modules Apache `cgi`/`rewrite` ou PHP | `a2enmod rewrite cgi`, redémarrer Apache |
| Impossible de se connecter | mauvais mot de passe | `sudo htpasswd /usr/local/nagios/etc/htpasswd.users nagiosadmin` |
| Hôte `DOWN` alors que la machine répond | ICMP bloqué (pare-feu) | autoriser ICMP, ou vérifier l'IP |
| Modification sans effet | pas de rechargement | `systemctl reload nagios` |

---

## 5. Questions de révision

1. Qu'est-ce que la supervision et quel est son intérêt ?
2. Quels sont les composants de Nagios (démon, plugins, interface web) ?
3. Quels états peut avoir un hôte ? un service ? À quels codes de retour correspondent-ils ?
4. Différence entre état *soft* et *hard* ? Rôle de `max_check_attempts` ?
5. À quoi servent les templates (`use`) ?
6. Que signifie `check_ping!100.0,20%!500.0,60%` ?
7. Pourquoi ne faut-il pas changer `contact_name` ? Quelle commande vérifie la configuration ?
8. Que fait le script d'installation à l'étape [6/7] ?
