# 🐧 SAR – Linux Services Labs

Dépôt de révision des **TP / Ateliers de « Services et Administration des Réseaux »** (ESPRIT, 2026-2027).

Chaque TP est un fichier Markdown autonome organisé de la même façon :

1. **Théorie** : les notions à comprendre avant de taper une commande
2. **Plan** : les grandes étapes du TP
3. **Pratique** : les commandes, **chacune expliquée**
4. **Vérification / dépannage** : comment savoir si ça marche, et que faire sinon
5. **Questions de révision**

---

## 📚 Sommaire

| # | TP | Distribution | Sujets |
|---|----|--------------|--------|
| 01 | [Noyau : modules et compilation](tp/01-noyau-modules-compilation.md) | AlmaLinux 8 | module `.ko`, `insmod`/`modprobe`, compilation du noyau 5.15 |
| 02 | [Noyau : patching](tp/02-noyau-patching.md) | Ubuntu / Debian | `patch -p1`, 6.1 → 6.1.1, GRUB |
| 03 | [DNS BIND 9](tp/03-dns-bind9.md) | Ubuntu 22.04 | zones directe/inverse, maître/esclave, AXFR, failover |
| 04 | [Serveur Web Apache](tp/04-web-apache.md) | Ubuntu 22.04 | virtual hosts (nom, IP, port), UFW, durcissement |
| 05 | [Messagerie Postfix + Dovecot + Fail2ban](tp/05-mail-postfix-dovecot-fail2ban.md) | Ubuntu 22.04 | SMTP, IMAP/POP3, Maildir, Thunderbird, anti brute-force |
| 06 | [FTP : vsftpd et Pure-FTPd](tp/06-ftp-vsftpd-pureftpd.md) | AlmaLinux 8 | chroot, anonyme, partage |
| 07 | [NFS](tp/07-nfs.md) | AlmaLinux 8 | `/etc/exports`, montage, `root_squash` |
| 08 | [Samba](tp/08-samba.md) | AlmaLinux 8 | `smb.conf`, utilisateurs Samba, accès Windows/Linux |

---

## 🗂️ Architecture du dépôt

```text
sar-linux-services-labs/
├── README.md
├── tp/
│   ├── 01-noyau-modules-compilation.md
│   ├── 02-noyau-patching.md
│   ├── 03-dns-bind9.md
│   ├── 04-web-apache.md
│   ├── 05-mail-postfix-dovecot-fail2ban.md
│   ├── 06-ftp-vsftpd-pureftpd.md
│   ├── 07-nfs.md
│   └── 08-samba.md

```

---

## 🌐 Plan d'adressage utilisé dans les TP

| Rôle | Adresse | TP |
|------|---------|----|
| Serveur principal (DNS maître, Apache) | `192.168.1.100` | DNS, Web, Mail |
| Alias IP 2 (site IP-based `esprit3`) | `192.168.1.101` | DNS, Web |
| Alias IP 3 (DNS esclave) | `192.168.1.102` | DNS |
| Serveur NFS / Samba | `192.168.18.6` | NFS, Samba |
| Client NFS | `192.168.18.4` | NFS |

> Adapte ces adresses à ton propre réseau (`ip a` pour les connaître).

---

## ✍️ Conventions

- `$` = commande en utilisateur normal, `#` = commande en root.
- `<...>` = valeur à remplacer par la tienne.
- ⚠️ = piège classique ; 💡 = astuce ; 🔒 = remarque de sécurité.

---

## ⚠️ Avertissement

Ces notes sont destinées à un **environnement de laboratoire** (VM). Plusieurs réglages (SELinux désactivé, authentification en clair, pare-feu coupé) ne doivent **pas** être reproduits en production ; les passages concernés sont signalés par 🔒.
