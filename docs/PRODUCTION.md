# Guide de Déploiement en Production
## Plateforme Nationale Formation Enseignants — Moodle 4.5 LTS

> **Stack cible :** Ubuntu 22.04 LTS | Nginx 1.24+ | PHP 8.2-FPM | MariaDB 10.11 LTS | Redis 7.x  
> **Audience :** Administrateur système responsable du déploiement  
> **Prérequis :** Accès root SSH au serveur, nom de domaine configuré avec DNS pointant vers le serveur

---

## Table des matières

1. [Architecture serveur recommandée](#1-architecture-serveur-recommandée)
2. [Préparation du serveur](#2-préparation-du-serveur)
3. [Installation et sécurisation de MariaDB](#3-installation-et-sécurisation-de-mariadb)
4. [Installation de PHP 8.2-FPM](#4-installation-de-php-82-fpm)
5. [Installation de Redis](#5-installation-de-redis)
6. [Installation et configuration de Nginx](#6-installation-et-configuration-de-nginx)
7. [Certificat SSL avec Let's Encrypt](#7-certificat-ssl-avec-lets-encrypt)
8. [Déploiement de Moodle](#8-déploiement-de-moodle)
9. [Installation via CLI Moodle](#9-installation-via-cli-moodle)
10. [Configuration post-installation](#10-configuration-post-installation)
11. [Cron système](#11-cron-système)
12. [Sécurité](#12-sécurité)
13. [Sauvegardes automatiques](#13-sauvegardes-automatiques)
14. [Monitoring et logs](#14-monitoring-et-logs)
15. [Checklist avant mise en ligne](#15-checklist-avant-mise-en-ligne)

---

## 1. Architecture serveur recommandée

### Configuration minimale (500–1 000 utilisateurs, ~100 sessions concurrentes)

| Serveur | CPU | RAM | Stockage | Rôle |
|---|---|---|---|---|
| **App (Moodle)** | 8 vCPU | 16 Go | 200 Go SSD | Nginx + PHP-FPM + Moodledata |
| **Base de données** | 4 vCPU | 16 Go | 100 Go SSD | MariaDB 10.11 |
| **BigBlueButton** | 8 vCPU | 16 Go | 100 Go SSD | BBB 2.7+ (VM dédiée obligatoire) |

> **Hébergement recommandé :** OVH (Gravelines/Paris), Scaleway, ou Hetzner — latence < 50 ms depuis le Maroc, conformité CNDP facilitée.

> Pour un démarrage avec budget limité, MariaDB peut être installé sur le même serveur que l'application. Migrer vers un serveur dédié DB dès que la charge le justifie.

### Nommage des domaines

```
moodle.votredomaine.com    → Serveur applicatif (port 80/443)
bbb.votredomaine.com       → Serveur BigBlueButton (port 80/443 + UDP)
```

---

## 2. Préparation du serveur

### Mise à jour système

```bash
apt update && apt upgrade -y
apt install -y curl wget gnupg2 ca-certificates lsb-release \
               apt-transport-https software-properties-common \
               unzip git ufw fail2ban
```

### Pare-feu (UFW)

```bash
ufw default deny incoming
ufw default allow outgoing
ufw allow 22/tcp      # SSH — restreindre à votre IP en prod idéalement
ufw allow 80/tcp      # HTTP (redirection vers HTTPS)
ufw allow 443/tcp     # HTTPS
ufw enable
ufw status
```

### Créer l'utilisateur applicatif

```bash
# www-data est déjà présent sur Ubuntu — vérifier
id www-data
```

### Répertoires applicatifs

```bash
mkdir -p /var/www/moodle
mkdir -p /var/www/moodledata
chown -R www-data:www-data /var/www/moodle /var/www/moodledata
chmod 750 /var/www/moodledata
```

---

## 3. Installation et sécurisation de MariaDB

### Installation MariaDB 10.11

```bash
# Ajouter le dépôt officiel MariaDB 10.11
curl -LsS https://r.mariadb.com/downloads/mariadb_repo_setup \
  | bash -s -- --mariadb-server-version="mariadb-10.11"

apt update
apt install -y mariadb-server mariadb-client
systemctl enable --now mariadb
```

### Sécurisation initiale

```bash
mariadb-secure-installation
# Répondre :
# - Set root password: OUI — choisir un mot de passe fort
# - Remove anonymous users: OUI
# - Disallow root login remotely: OUI
# - Remove test database: OUI
# - Reload privilege tables: OUI
```

### Créer la base de données et l'utilisateur Moodle

```bash
mariadb -u root -p
```

```sql
CREATE DATABASE moodle CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'moodle'@'localhost' IDENTIFIED BY 'MOT_DE_PASSE_FORT_ICI';
GRANT ALL PRIVILEGES ON moodle.* TO 'moodle'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

> **Mot de passe :** Utiliser un générateur (`openssl rand -base64 32`) — ne jamais réutiliser les mots de passe du `.env` local.

### Optimisation MariaDB pour Moodle

Créer `/etc/mysql/conf.d/moodle.cnf` :

```ini
[mysqld]
# Encodage — obligatoire Moodle 4.x (emojis, arabe)
character-set-server  = utf8mb4
collation-server      = utf8mb4_unicode_ci

# InnoDB
innodb_file_per_table     = 1
innodb_buffer_pool_size   = 4G      # ~25-30% de la RAM disponible
innodb_log_file_size      = 256M
innodb_flush_log_at_trx_commit = 2  # Légère perte de durabilité, gain perf
innodb_flush_method       = O_DIRECT

# Connexions
max_connections           = 200
thread_cache_size         = 50
table_open_cache          = 4000

# Query cache (désactivé en MariaDB 10.9+)
query_cache_type          = 0
query_cache_size          = 0

# Logs
slow_query_log            = 1
slow_query_log_file       = /var/log/mysql/slow.log
long_query_time           = 2

[client]
default-character-set = utf8mb4
```

```bash
systemctl restart mariadb
```

---

## 4. Installation de PHP 8.2-FPM

### Ajouter le dépôt Ondřej Surý (PHP versions récentes)

```bash
add-apt-repository ppa:ondrej/php -y
apt update
```

### Installer PHP 8.2 et toutes les extensions requises par Moodle 4.5

```bash
apt install -y \
  php8.2-fpm \
  php8.2-cli \
  php8.2-mysql \
  php8.2-xml \
  php8.2-mbstring \
  php8.2-curl \
  php8.2-zip \
  php8.2-gd \
  php8.2-intl \
  php8.2-soap \
  php8.2-ldap \
  php8.2-xsl \
  php8.2-bcmath \
  php8.2-exif \
  php8.2-opcache \
  php8.2-redis \
  php8.2-xmlrpc
```

> **Note sur `xmlrpc` :** Retiré du core PHP en 8.0, disponible via le dépôt Ondřej sous `php8.2-xmlrpc`. Ne pas utiliser `pecl install xmlrpc` en production — préférer le paquet système.

### Configuration PHP pour Moodle

Éditer `/etc/php/8.2/fpm/conf.d/99-moodle.ini` :

```ini
; Limites d'upload (supports pédagogiques, vidéos)
upload_max_filesize = 200M
post_max_size       = 200M
max_file_uploads    = 50

; Exécution
max_execution_time  = 300
max_input_time      = 300
memory_limit        = 256M
max_input_vars      = 5000

; Sécurité
expose_php          = Off
display_errors      = Off
log_errors          = On
error_log           = /var/log/php8.2-fpm-moodle.log

; Sessions
session.save_handler = redis
session.save_path    = "tcp://127.0.0.1:6379"

; OPcache — fortement recommandé en production
opcache.enable              = 1
opcache.memory_consumption  = 256
opcache.interned_strings_buffer = 16
opcache.max_accelerated_files   = 10000
opcache.revalidate_freq     = 60
opcache.save_comments       = 1
```

### Configuration PHP-FPM pool

Éditer `/etc/php/8.2/fpm/pool.d/moodle.conf` :

```ini
[moodle]
user  = www-data
group = www-data

listen = /run/php/php8.2-fpm-moodle.sock
listen.owner = www-data
listen.group = www-data
listen.mode  = 0660

pm = dynamic
pm.max_children      = 50
pm.start_servers     = 10
pm.min_spare_servers = 5
pm.max_spare_servers = 20
pm.max_requests      = 500

; Logs lents
request_slowlog_timeout = 10s
slowlog = /var/log/php8.2-fpm-slow.log
```

```bash
systemctl enable --now php8.2-fpm
systemctl restart php8.2-fpm
```

---

## 5. Installation de Redis

```bash
apt install -y redis-server
```

Éditer `/etc/redis/redis.conf` :

```conf
# Écouter uniquement en local (pas exposé sur le réseau)
bind 127.0.0.1 ::1

# Mot de passe (recommandé même en local)
requirepass MOT_DE_PASSE_REDIS_ICI

# Mémoire maximale
maxmemory 512mb
maxmemory-policy allkeys-lru

# Persistance
appendonly yes
appendfsync everysec

# Socket Unix (plus rapide que TCP pour PHP-FPM en local)
unixsocket /var/run/redis/redis.sock
unixsocketperm 770
```

```bash
usermod -aG redis www-data
systemctl enable --now redis-server
systemctl restart redis-server
```

---

## 6. Installation et configuration de Nginx

```bash
apt install -y nginx
systemctl enable nginx
```

### Virtual host Moodle

Créer `/etc/nginx/sites-available/moodle` :

```nginx
server {
    listen 80;
    server_name moodle.votredomaine.com;

    # Redirection HTTPS — à décommenter après avoir obtenu le certificat SSL
    # return 301 https://$host$request_uri;

    root /var/www/moodle;
    index index.php;

    client_max_body_size 200M;
    client_body_timeout 300s;

    # Compression
    gzip on;
    gzip_types text/plain text/css application/json application/javascript
               text/xml application/xml application/xml+rss text/javascript;
    gzip_min_length 256;

    # Cache statiques
    location ~* \.(ico|css|js|gif|jpg|jpeg|png|svg|woff|woff2|ttf|eot)$ {
        expires 30d;
        add_header Cache-Control "public, no-transform";
        log_not_found off;
        access_log off;
    }

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    # PHP-FPM via socket Unix
    location ~ [^/]\.php(/|$) {
        fastcgi_split_path_info ^(.+?\.php)(/.*)$;
        if (!-f $document_root$fastcgi_script_name) { return 404; }

        fastcgi_pass   unix:/run/php/php8.2-fpm-moodle.sock;
        fastcgi_index  index.php;
        fastcgi_buffers 16 16k;
        fastcgi_buffer_size 32k;

        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param PATH_INFO       $fastcgi_path_info;
        fastcgi_param HTTPS           on;

        fastcgi_read_timeout 300s;
        fastcgi_send_timeout 300s;
    }

    # Protections Moodle
    location ~ /\.ht              { deny all; }
    location ~ /vendor/.*\.php$   { deny all; }
    location ~ /node_modules/     { deny all; }
    location /dataroot/            { deny all; }

    access_log /var/log/nginx/moodle_access.log;
    error_log  /var/log/nginx/moodle_error.log warn;
}
```

```bash
ln -s /etc/nginx/sites-available/moodle /etc/nginx/sites-enabled/
rm /etc/nginx/sites-enabled/default
nginx -t && systemctl reload nginx
```

---

## 7. Certificat SSL avec Let's Encrypt

```bash
apt install -y certbot python3-certbot-nginx

# Obtenir le certificat (Nginx doit répondre sur le port 80)
certbot --nginx -d moodle.votredomaine.com \
  --email admin@votredomaine.com \
  --agree-tos --no-eff-email
```

Certbot modifie automatiquement le vhost Nginx. Vérifier que la redirection HTTP→HTTPS est active :

```bash
curl -I http://moodle.votredomaine.com
# Doit retourner : 301 Moved Permanently → https://...
```

Renouvellement automatique (déjà configuré par certbot) :
```bash
systemctl status certbot.timer
```

> **Important :** Configurer SSL **avant** l'installation Moodle. Le `wwwroot` dans `config.php` doit être `https://moodle.votredomaine.com` dès le départ — le changer après installation nécessite une purge totale des caches et peut casser des liens.

---

## 8. Déploiement de Moodle

### Cloner ou copier les sources

```bash
# Option A — depuis le dépôt officiel Moodle 4.5
cd /var/www
git clone -b MOODLE_405_STABLE --depth 1 \
  https://github.com/moodle/moodle.git moodle

# Option B — copier depuis l'environnement de développement
rsync -avz --exclude='.git' ./moodle/ root@serveur:/var/www/moodle/
```

### Permissions

```bash
chown -R www-data:www-data /var/www/moodle
find /var/www/moodle -type f -exec chmod 644 {} \;
find /var/www/moodle -type d -exec chmod 755 {} \;

# moodledata — séparé du webroot, non accessible via HTTP
chown -R www-data:www-data /var/www/moodledata
chmod 750 /var/www/moodledata
```

---

## 9. Installation via CLI Moodle

```bash
sudo -u www-data php /var/www/moodle/admin/cli/install.php \
  --lang=fr \
  --wwwroot=https://moodle.votredomaine.com \
  --dataroot=/var/www/moodledata \
  --dbtype=mariadb \
  --dbhost=localhost \
  --dbport=3306 \
  --dbname=moodle \
  --dbuser=moodle \
  --dbpass=MOT_DE_PASSE_FORT_ICI \
  --fullname="Plateforme Nationale Formation Enseignants" \
  --shortname=LMS \
  --adminuser=admin \
  --adminpass="MOT_DE_PASSE_ADMIN_ICI" \
  --adminemail=admin@formation.ma \
  --non-interactive \
  --agree-license
```

> **`--dbtype=mariadb` est obligatoire.** Ne pas utiliser `mysqli` avec MariaDB 10.6+. Le driver `mariadb` natif de Moodle offre un meilleur support des fonctionnalités MariaDB (notamment pour les index full-text et la gestion des colonnes JSON). Utiliser `mysqli` déclenche un avertissement critique dans l'interface d'administration Moodle.

### Permissions du config.php généré

```bash
chown www-data:www-data /var/www/moodle/config.php
chmod 640 /var/www/moodle/config.php
```

> Le CLI Moodle génère `config.php` avec le propriétaire du processus courant. Si exécuté en root, les permissions seront `root:root` et PHP-FPM (www-data) ne pourra pas lire le fichier → erreur 500.

---

## 10. Configuration post-installation

### Compléter config.php

Éditer `/var/www/moodle/config.php` et ajouter après la ligne `require_once` :

```php
// Sécurité
$CFG->reverseproxy     = true;   // Si derrière un load balancer
$CFG->sslproxy         = true;   // HTTPS via proxy/Nginx
$CFG->cookiesecure     = true;
$CFG->cookiehttponly   = true;

// Redis — Sessions
$CFG->session_handler_class = '\core\session\redis';
$CFG->session_redis_host    = '127.0.0.1';
$CFG->session_redis_port    = 6379;
$CFG->session_redis_auth    = 'MOT_DE_PASSE_REDIS_ICI';
$CFG->session_redis_database = 0;
$CFG->session_redis_acquire_lock_timeout = 120;
$CFG->session_redis_lock_expire = 7200;

// Performance
$CFG->pathtogs       = '/usr/bin/gs';       // Ghostscript (aperçus PDF)
$CFG->pathtopython   = '/usr/bin/python3';
$CFG->aspellpath     = '/usr/bin/aspell';

// Taille max upload (doit correspondre à php.ini)
$CFG->maxbytes       = 209715200;  // 200 Mo
$CFG->maxfilesize    = 209715200;
```

### Cache universel (MUC) Redis

Dans l'interface d'administration :
1. Administration du site > Plugins > Caches > Configuration
2. Ajouter un store Redis :
   - Serveur : `127.0.0.1:6379`
   - Authentification : mot de passe Redis
3. Mapper `application` et `session` sur ce store

### Désactiver les fonctions inutiles en production

Administration du site > Sécurité > Politique du site :
- Désactiver l'auto-inscription publique
- Activer la politique mots de passe renforcée (min. 10 caractères)
- Désactiver les profils publics d'utilisateurs

Administration du site > Serveur > Nettoyage :
- Activer le nettoyage des fichiers temporaires

---

## 11. Cron système

Le cron Moodle doit s'exécuter **chaque minute** en production (contrairement au conteneur Docker qui utilise une boucle toutes les 60 secondes).

```bash
crontab -u www-data -e
```

Ajouter :
```
* * * * * /usr/bin/php /var/www/moodle/admin/cli/cron.php >> /var/log/moodle-cron.log 2>&1
```

Vérifier l'exécution :
```bash
tail -f /var/log/moodle-cron.log
```

---

## 12. Sécurité

### Fail2ban — protection brute force

```bash
apt install -y fail2ban
```

Créer `/etc/fail2ban/jail.d/moodle.conf` :

```ini
[moodle]
enabled  = true
port     = http,https
filter   = moodle
logpath  = /var/log/nginx/moodle_access.log
maxretry = 5
bantime  = 3600
findtime = 600
```

Créer `/etc/fail2ban/filter.d/moodle.conf` :

```ini
[Definition]
failregex = ^<HOST> .* "POST /login/index.php.*" (401|403|429)
ignoreregex =
```

```bash
systemctl restart fail2ban
```

### Permissions critiques à vérifier

```bash
# config.php — lisible uniquement par www-data
ls -la /var/www/moodle/config.php
# attendu : -rw-r----- 1 www-data www-data

# moodledata — hors du webroot, non accessible via HTTP
ls -la /var/www/moodledata
# attendu : drwxr-x--- 1 www-data www-data

# S'assurer que moodledata n'est pas dans le webroot Nginx
# et qu'aucun alias Nginx ne pointe vers ce répertoire
```

### Headers de sécurité Nginx

Ajouter dans le bloc `server` HTTPS du vhost :

```nginx
add_header X-Frame-Options "SAMEORIGIN" always;
add_header X-Content-Type-Options "nosniff" always;
add_header X-XSS-Protection "1; mode=block" always;
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;
```

### Désactiver l'accès root SSH (après avoir créé un utilisateur sudo)

```bash
# Dans /etc/ssh/sshd_config
PermitRootLogin no
PasswordAuthentication no   # si clés SSH configurées
MaxAuthTries 3

systemctl restart sshd
```

### Moodle — points de sécurité spécifiques

- Administration > Sécurité > Notifications de sécurité : activer les alertes par email
- Administration > Utilisateurs > Authentification : désactiver tous les plugins d'auth non utilisés
- Administration > Avancé > Débogage : mettre `NONE` en production
- Activer `tool_mfa` (MFA) pour les comptes administrateurs et responsables pédagogiques
- Activer les logs de sécurité : Administration > Rapports > Logs

---

## 13. Sauvegardes automatiques

### Script de sauvegarde

Créer `/usr/local/bin/moodle-backup.sh` :

```bash
#!/bin/bash
set -euo pipefail

BACKUP_DIR="/var/backups/moodle"
DATE=$(date +%Y%m%d_%H%M)
DB_USER="moodle"
DB_PASS="MOT_DE_PASSE_FORT_ICI"
DB_NAME="moodle"
MOODLEDATA="/var/www/moodledata"
RETENTION_DAYS=30

mkdir -p "$BACKUP_DIR/db" "$BACKUP_DIR/data"

# 1. Dump de la base de données
mariadb-dump \
  --user="$DB_USER" \
  --password="$DB_PASS" \
  --single-transaction \
  --routines \
  --triggers \
  "$DB_NAME" | gzip > "$BACKUP_DIR/db/moodle_db_${DATE}.sql.gz"

# 2. Snapshot moodledata (incrémental avec tar)
tar czf "$BACKUP_DIR/data/moodledata_${DATE}.tar.gz" \
  --exclude="$MOODLEDATA/cache" \
  --exclude="$MOODLEDATA/temp" \
  --exclude="$MOODLEDATA/sessions" \
  "$MOODLEDATA"

# 3. Purger les sauvegardes de plus de 30 jours
find "$BACKUP_DIR" -type f -mtime +$RETENTION_DAYS -delete

echo "[$DATE] Sauvegarde terminée."
```

```bash
chmod +x /usr/local/bin/moodle-backup.sh
```

### Planifier la sauvegarde nocturne

```bash
crontab -e
```

```
# Sauvegarde Moodle — chaque nuit à 2h00
0 2 * * * /usr/local/bin/moodle-backup.sh >> /var/log/moodle-backup.log 2>&1
```

### Sync S3 (optionnel mais recommandé)

```bash
apt install -y awscli  # ou rclone pour Scaleway/OVH Object Storage

# Ajouter après la sauvegarde dans le script
aws s3 sync "$BACKUP_DIR" s3://votre-bucket-moodle/backups/ \
  --delete \
  --storage-class STANDARD_IA
```

### Test de restauration

À faire au moins une fois avant la mise en production :

```bash
# Restaurer la base de données
gunzip < /var/backups/moodle/db/moodle_db_YYYYMMDD_HHmm.sql.gz \
  | mariadb -u moodle -p moodle_restauration

# Restaurer moodledata
tar xzf /var/backups/moodle/data/moodledata_YYYYMMDD_HHmm.tar.gz \
  -C /var/www/moodledata_restauration/
```

---

## 14. Monitoring et logs

### Logs à surveiller

| Fichier | Contenu |
|---|---|
| `/var/log/nginx/moodle_access.log` | Accès HTTP |
| `/var/log/nginx/moodle_error.log` | Erreurs Nginx |
| `/var/log/php8.2-fpm-moodle.log` | Erreurs PHP-FPM |
| `/var/log/php8.2-fpm-slow.log` | Requêtes PHP lentes (> 10s) |
| `/var/log/mysql/slow.log` | Requêtes SQL lentes (> 2s) |
| `/var/log/moodle-cron.log` | Exécution cron Moodle |
| `/var/log/moodle-backup.log` | Sauvegardes |

### Vérification rapide de santé

```bash
# Statut des services
systemctl status nginx php8.2-fpm mariadb redis-server

# Test HTTP
curl -I https://moodle.votredomaine.com
# Attendu : HTTP/2 200

# Cron Moodle actif
tail -5 /var/log/moodle-cron.log

# Redis opérationnel
redis-cli -a MOT_DE_PASSE_REDIS_ICI ping
# Attendu : PONG

# MariaDB accessible
mariadb -u moodle -p -e "SELECT COUNT(*) FROM moodle.mdl_user;"
```

### Alertes (V1 — simple)

```bash
# Alerte email si le site ne répond pas (à planifier via cron)
curl -sf https://moodle.votredomaine.com > /dev/null || \
  mail -s "[ALERTE] Moodle ne répond pas" admin@formation.ma <<< "Vérifier le serveur"
```

> Pour la V2, déployer Prometheus + Grafana avec les exporters Nginx, PHP-FPM, MariaDB et Redis pour un monitoring complet.

---

## 15. Checklist avant mise en ligne

### Infrastructure

- [ ] Serveur Ubuntu 22.04 LTS provisionné et accès SSH confirmé
- [ ] UFW configuré (ports 22, 80, 443 uniquement)
- [ ] Fail2ban actif
- [ ] Accès root SSH désactivé

### Services

- [ ] MariaDB 10.11 installé, sécurisé (`mariadb-secure-installation`)
- [ ] Base `moodle` créée en `utf8mb4_unicode_ci`
- [ ] PHP 8.2-FPM installé avec toutes les extensions (vérifier avec `php -m`)
- [ ] OPcache activé
- [ ] Redis installé, protégé par mot de passe, socket Unix actif
- [ ] Nginx installé, vhost Moodle configuré

### SSL et domaine

- [ ] DNS `moodle.votredomaine.com` → IP serveur (propagé)
- [ ] Certificat Let's Encrypt obtenu et actif
- [ ] Redirection HTTP → HTTPS active
- [ ] Headers de sécurité Nginx ajoutés
- [ ] HSTS activé

### Moodle

- [ ] Sources Moodle 4.5 déployées dans `/var/www/moodle`
- [ ] `config.php` généré avec `--dbtype=mariadb` (pas `mysqli`)
- [ ] Permissions `config.php` : `640 www-data:www-data`
- [ ] Permissions `moodledata` : `750 www-data:www-data`
- [ ] `moodledata` hors du webroot Nginx
- [ ] Redis configuré dans `config.php` (sessions)
- [ ] Cache MUC Redis configuré dans l'interface admin
- [ ] Cron système actif (vérifier `/var/log/moodle-cron.log`)
- [ ] Mode débogage désactivé (Administration > Avancé)
- [ ] Auto-inscription publique désactivée
- [ ] MFA activé pour les administrateurs (`tool_mfa`)

### Sauvegardes

- [ ] Script de sauvegarde testé manuellement
- [ ] Cron de sauvegarde nocturne configuré
- [ ] Test de restauration effectué sur un environnement de test
- [ ] Sync S3/Object Storage configuré (optionnel mais recommandé)

### Validation finale

- [ ] Connexion admin fonctionnelle sur `https://moodle.votredomaine.com`
- [ ] Upload d'un fichier test (> 10 Mo)
- [ ] Email de test envoyé depuis Moodle (Administration > Serveur > Email sortant)
- [ ] Cron exécuté sans erreur
- [ ] Aucune alerte rouge dans Administration > Notifications

---

## Points d'attention spécifiques MariaDB

| Sujet | Ce qu'il faut faire | Ce qu'il ne faut pas faire |
|---|---|---|
| **Driver Moodle** | `$CFG->dbtype = 'mariadb'` dans `config.php` | ~~`mysqli`~~ — déclenche des avertissements dans l'admin Moodle avec MariaDB 10.6+ |
| **CLI install** | `--dbtype=mariadb` | ~~`--dbtype=mysqli`~~ |
| **Encodage** | `utf8mb4_unicode_ci` pour la base et toutes les tables | ~~`utf8_general_ci`~~ — troncature silencieuse des emojis et certains caractères arabes |
| **Dump** | `mariadb-dump` (binaire MariaDB) | ~~`mysqldump`~~ — peut manquer des fonctionnalités MariaDB-spécifiques |
| **Client** | `mariadb` (CLI natif) | ~~`mysql`~~ — fonctionne mais `mariadb` est le binaire officiel |
| **InnoDB buffer pool** | 25-30% de la RAM | Ne pas dépasser 70% — laisser de la mémoire pour l'OS et les connexions |

---

*Document créé le 30 avril 2026 — Projet RFC Digital / Wave Inc*
