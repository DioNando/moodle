# Environnement Docker — Plateforme Nationale Formation Enseignants

> Stack : **Nginx 1.24** | **PHP 8.2-FPM** | **MariaDB 10.11 LTS** | **Redis 7**  
> Moodle : source locale dans `./moodle/` (version **4.5.x LTS**)  
> BigBlueButton : serveur externe (voir section dédiée)

---

## Table des matières

1. [Prérequis](#1-prérequis)
2. [Structure des fichiers](#2-structure-des-fichiers)
3. [Démarrage rapide](#3-démarrage-rapide)
4. [Configuration (.env)](#4-configuration-env)
5. [Services et rôles](#5-services-et-rôles)
6. [BigBlueButton — cas particulier](#6-bigbluebutton--cas-particulier)
7. [Installation de Moodle](#7-installation-de-moodle)
8. [Configuration Redis dans Moodle](#8-configuration-redis-dans-moodle)
9. [Commandes utiles](#9-commandes-utiles)
10. [Profils Docker (outils optionnels)](#10-profils-docker-outils-optionnels)
11. [Données persistantes et volumes](#11-données-persistantes-et-volumes)
12. [Différences avec la production](#12-différences-avec-la-production)
13. [Passage en production](#13-passage-en-production)

---

## 1. Prérequis

| Outil | Version minimale | Installation |
|---|---|---|
| Docker Desktop | 24.x+ | [docs.docker.com](https://docs.docker.com/get-docker/) |
| Docker Compose | v2.x (intégré) | Inclus avec Docker Desktop |
| RAM disponible | 4 Go minimum | 8 Go recommandé |
| Espace disque | 5 Go minimum | Pour images + volumes |

Vérification :
```bash
docker --version          # Docker version 24.x+
docker compose version    # Docker Compose version v2.x
```

---

## 2. Structure des fichiers

```
LMS/
├── docker-compose.yml          ← Orchestration de tous les services
├── Dockerfile                  ← Image PHP 8.2-FPM + extensions Moodle
├── .env.example                ← Template des variables d'environnement
├── .env                        ← Variables locales (à créer, non versionné)
│
├── moodle/                     ← Source Moodle 4.5 LTS (montée en volume)
│   ├── config.php              ← Créé automatiquement par le CLI installer
│   ├── index.php
│   ├── admin/
│   ├── lib/
│   └── ...
│
└── config/
    ├── nginx/
    │   ├── default.conf        ← Virtual host Nginx pour Moodle
    │   └── certs/              ← Certificats SSL (optionnel en local)
    ├── php/
    │   └── moodle.ini          ← Paramètres PHP optimisés Moodle
    └── mariadb/
        └── moodle.cnf          ← Paramètres MariaDB (utf8mb4, InnoDB…)
```

---

## 3. Démarrage rapide

### Étape 1 — Copier et adapter le fichier d'environnement

```bash
cp .env.example .env
```

Éditer `.env` pour définir au minimum :
- `MOODLE_WWWROOT` : URL d'accès (ex. `http://localhost`)
- `DB_PASSWORD`, `DB_ROOT_PASSWORD` : mots de passe de la base
- `MOODLE_ADMIN_PASSWORD` : mot de passe admin Moodle

### Étape 2 — Construire et démarrer les conteneurs

```bash
# Première fois : build de l'image PHP + démarrage
docker compose up --build -d

# Les fois suivantes (image déjà construite)
docker compose up -d
```

### Étape 3 — Installer Moodle via la CLI

```bash
docker exec moodle_php php /var/www/html/admin/cli/install.php \
  --lang=fr \
  --wwwroot=http://localhost \
  --dataroot=/var/www/moodledata \
  --dbtype=mariadb \
  --dbhost=mariadb \
  --dbport=3306 \
  --dbname=moodle \
  --dbuser=moodle \
  --dbpass=moodle_secret_2026 \
  --fullname="Plateforme Nationale Formation Enseignants" \
  --shortname=LMS \
  --adminuser=admin \
  --adminpass="Admin@2026!" \
  --adminemail=admin@formation.ma \
  --non-interactive \
  --agree-license
```

> **Important — driver MariaDB :** Utiliser `--dbtype=mariadb` (et **non** `mysqli`). Depuis Moodle 4.x, MariaDB dispose de son propre driver natif. Utiliser `mysqli` provoque une erreur de validation de l'environnement.

Après l'installation, corriger les permissions du `config.php` généré :

```bash
docker exec moodle_php chown www-data:www-data /var/www/html/config.php
docker exec moodle_php chmod 640 /var/www/html/config.php
```

### Étape 4 — Accéder à Moodle

Ouvrir [http://localhost](http://localhost). La plateforme doit répondre HTTP 200.

---

## 4. Configuration (.env)

| Variable | Défaut | Description |
|---|---|---|
| `MOODLE_WWWROOT` | `http://localhost` | URL publique de la plateforme |
| `MOODLE_SITE_NAME` | `Plateforme...` | Nom affiché du site |
| `MOODLE_ADMIN_USER` | `admin` | Identifiant du compte admin |
| `MOODLE_ADMIN_PASSWORD` | `Admin@2026!` | Mot de passe admin |
| `MOODLE_ADMIN_EMAIL` | `admin@formation.ma` | Email du compte admin |
| `DB_NAME` | `moodle` | Nom de la base de données |
| `DB_USER` | `moodle` | Utilisateur MariaDB |
| `DB_PASSWORD` | `moodle_secret_2026` | Mot de passe utilisateur |
| `DB_ROOT_PASSWORD` | `root_secret_2026` | Mot de passe root MariaDB |
| `HTTP_PORT` | `80` | Port HTTP exposé |
| `HTTPS_PORT` | `443` | Port HTTPS exposé |
| `PMA_PORT` | `8080` | Port phpMyAdmin (profil `tools`) |
| `MAIL_UI_PORT` | `8025` | Interface web Mailpit (profil `tools`) |
| `MAIL_SMTP_PORT` | `1025` | Port SMTP Mailpit (profil `tools`) |

> Ne jamais committer le fichier `.env` — ajouter `.env` au `.gitignore`.

---

## 5. Services et rôles

### `nginx` — Serveur web

- Image : `nginx:1.24-alpine`
- Reçoit les requêtes HTTP sur le port `80` (ou `443`)
- Sert les fichiers statiques directement (CSS, JS, images)
- Passe les requêtes PHP à `php:9000` via FastCGI
- Config montée depuis `config/nginx/default.conf`
- Racine web : `/var/www/html` (source Moodle 4.5)

### `php` — Application Moodle

- Image custom (Dockerfile) : `php:8.2-fpm` + extensions Moodle
- Extensions compilées : `gd`, `intl`, `zip`, `soap`, `pdo_mysql`, `mysqli`, `mbstring`, `curl`, `exif`, `opcache`, `ldap`, `xsl`, `bcmath`
- Extensions PECL : `xmlrpc` (retiré du core PHP en 8.0, installé via `pecl install xmlrpc-beta`), `redis`
- Le code source Moodle (`./moodle/`) est monté en volume dans `/var/www/html`
- Le répertoire `moodledata` (fichiers uploadés, cache) est dans le volume `moodle_data`

### `cron` — Tâches planifiées Moodle

- Même image que `php`
- Lance `admin/cli/cron.php` toutes les **60 secondes** en boucle
- Moodle l'exige pour : notifications email, achèvements d'activités, rapports planifiés, etc.
- Séparé du conteneur `php` pour ne pas bloquer les requêtes web

### `mariadb` — Base de données

- Image : `mariadb:10.11` (LTS, support jusqu'en 2028)
- **Driver Moodle à utiliser : `mariadb`** (pas `mysqli`) — MariaDB 10.6+ dispose d'un driver natif dans Moodle 4.x qui offre de meilleures performances et un meilleur support des fonctionnalités MariaDB-spécifiques
- Configurée avec `utf8mb4` (obligatoire Moodle 4.x pour les emojis et l'arabe)
- `innodb_file_per_table = 1` : meilleure gestion de l'espace disque
- Healthcheck actif : les services `php` et `cron` attendent que la DB soit prête

### `redis` — Cache applicatif

- Image : `redis:7-alpine`
- Configuré avec `maxmemory 256mb` et politique `allkeys-lru`
- Utilisé par Moodle comme cache universel (MUC) après configuration (voir §8)
- Réduit drastiquement la charge sur MariaDB (×3 en performances)

---

## 6. BigBlueButton — cas particulier

### Pourquoi BBB n'est pas dans le docker-compose ?

BigBlueButton est une application **monolithique** qui nécessite :
- Un serveur Ubuntu 22.04 dédié (pas un conteneur)
- Un **nom de domaine résolvable** avec certificat SSL valide
- Des ports réseau spécifiques ouverts : 80, 443, 16384–32768/UDP (WebRTC)
- Minimum **8 vCPU / 16 Go RAM** pour fonctionner correctement

Il n'existe pas d'image Docker officielle BBB stable adaptée au développement local.

### Options pour le développement

**Option A — Serveur de démo BBB (recommandé pour tester)**

Utiliser le serveur public de démonstration BigBlueButton :
- URL : `https://demo.bigbluebutton.org/bigbluebutton/`
- Secret : disponible sur [bigbluebutton.org/demo](https://bigbluebutton.org/demo)
- Limitations : sessions limitées à 60 min, usage non-production

Configurer dans Moodle : Administration du site > Plugins > Activités > BigBlueButtonBN

**Option B — VM dédiée (environnement intégration)**

Installer BBB sur une VM Ubuntu 22.04 séparée :
```bash
# Sur la VM Ubuntu 22.04 avec domaine + SSL configurés
wget -qO- https://raw.githubusercontent.com/bigbluebutton/bbb-install/v2.7.x-release/bbb-install.sh \
  | bash -s -- -v focal-270 -s bbb.votredomaine.com -e admin@votredomaine.com
```

Puis renseigner dans `.env` :
```env
BBB_URL=https://bbb.votredomaine.com/bigbluebutton/
BBB_SECRET=<secret_obtenu_via_bbb-conf_--secret>
```

---

## 7. Installation de Moodle

Après `docker compose up -d`, utiliser la **méthode CLI** (recommandée — reproductible) ou l'interface web.

### Via la CLI (recommandé)

```bash
docker exec moodle_php php /var/www/html/admin/cli/install.php \
  --lang=fr \
  --wwwroot=http://localhost \
  --dataroot=/var/www/moodledata \
  --dbtype=mariadb \
  --dbhost=mariadb \
  --dbport=3306 \
  --dbname=moodle \
  --dbuser=moodle \
  --dbpass=moodle_secret_2026 \
  --fullname="Plateforme Nationale Formation Enseignants" \
  --shortname=LMS \
  --adminuser=admin \
  --adminpass="Admin@2026!" \
  --adminemail=admin@formation.ma \
  --non-interactive \
  --agree-license
```

Corriger ensuite les permissions :
```bash
docker exec moodle_php chown www-data:www-data /var/www/html/config.php
docker exec moodle_php chmod 640 /var/www/html/config.php
```

Si `config.php` existe déjà (installation interrompue), utiliser directement :
```bash
docker exec moodle_php php /var/www/html/admin/cli/install_database.php \
  --lang=fr \
  --fullname="Plateforme Nationale Formation Enseignants" \
  --shortname=LMS \
  --adminuser=admin \
  --adminpass="Admin@2026!" \
  --adminemail=admin@formation.ma \
  --agree-license
```

### Via l'interface web

1. Ouvrir [http://localhost](http://localhost)
2. Suivre l'assistant d'installation
3. Paramètres de connexion à la base de données :

   | Champ | Valeur |
   |---|---|
   | **Type** | `MariaDB (native/mariadb)` — **ne pas choisir `mysqli`** |
   | Hôte | `mariadb` |
   | Base de données | `moodle` |
   | Utilisateur | `moodle` |
   | Mot de passe | valeur `DB_PASSWORD` du `.env` |
   | Préfixe tables | `mdl_` |

4. Définir le `moodledata` : `/var/www/moodledata`
5. Créer le compte administrateur

> **Pourquoi `mariadb` et pas `mysqli` ?** Le driver `mariadb` (natif) est obligatoire avec MariaDB 10.6+ et Moodle 4.x. Le driver `mysqli` est prévu pour MySQL et peut provoquer des erreurs de validation de l'environnement avec MariaDB récent.

---

## 8. Configuration Redis dans Moodle

Après l'installation, configurer Moodle pour utiliser Redis comme cache.

### Étape 1 — Ajouter dans config.php

```bash
docker compose exec php sh -c "cat >> /var/www/html/config.php << 'EOF'

// Redis — Sessions et cache MUC
\$CFG->session_handler_class = '\core\session\redis';
\$CFG->session_redis_host = 'redis';
\$CFG->session_redis_port = 6379;
\$CFG->session_redis_database = 0;
\$CFG->session_redis_acquire_lock_timeout = 120;
\$CFG->session_redis_lock_expire = 7200;
EOF"
```

### Étape 2 — Configurer le MUC (cache universel)

Administration du site > Plugins > Caches > Configuration :
1. Ajouter un store Redis : hôte `redis`, port `6379`
2. Mapper les définitions `application` et `session` sur ce store

---

## 9. Commandes utiles

### Cycle de vie

```bash
# Démarrer tous les services
docker compose up -d

# Arrêter sans supprimer les volumes
docker compose stop

# Arrêter et supprimer les conteneurs (volumes conservés)
docker compose down

# Tout supprimer y compris les volumes (RESET COMPLET)
docker compose down -v
```

### Logs

```bash
# Tous les services
docker compose logs -f

# Un service spécifique
docker compose logs -f php
docker compose logs -f mariadb
docker compose logs -f cron
```

### Accès aux conteneurs

```bash
# Shell PHP (pour CLI Moodle)
docker compose exec php bash

# Client MariaDB (driver natif)
docker compose exec mariadb mariadb -u moodle -pmoodle_secret_2026 moodle

# Redis CLI
docker compose exec redis redis-cli
```

### Administration Moodle CLI

```bash
# Lancer le cron manuellement
docker compose exec php php admin/cli/cron.php

# Vider tous les caches Moodle
docker compose exec php php admin/cli/purge_caches.php

# Mettre à jour les plugins après modification
docker compose exec php php admin/cli/upgrade.php --non-interactive

# Changer le mot de passe admin
docker compose exec php php admin/cli/reset_password.php --username=admin

# Activer/désactiver le mode maintenance
docker compose exec php php admin/cli/maintenance.php --enable
docker compose exec php php admin/cli/maintenance.php --disable
```

### Build

```bash
# Reconstruire l'image PHP (après modif du Dockerfile)
docker compose build php
docker compose up -d php
```

---

## 10. Profils Docker (outils optionnels)

Les outils de développement sont dans le profil `tools` pour ne pas alourdir l'environnement par défaut.

```bash
# Démarrer avec les outils de développement
docker compose --profile tools up -d
```

| Service | URL | Description |
|---|---|---|
| **phpMyAdmin** | [http://localhost:8080](http://localhost:8080) | Interface web pour explorer/modifier la base MariaDB |
| **Mailpit** | [http://localhost:8025](http://localhost:8025) | Capture tous les emails envoyés par Moodle (notifications, reset mdp…) |

Pour utiliser Mailpit, configurer le SMTP dans Moodle :
- Administration du site > Serveur > Email sortant
- Hôte SMTP : `mailpit` | Port : `1025` | Pas d'authentification

---

## 11. Données persistantes et volumes

| Volume Docker | Contenu | Suppression |
|---|---|---|
| `moodle_data` | Fichiers uploadés, cache Moodle, backups locaux (`/var/www/moodledata`) | `docker compose down -v` |
| `mariadb_data` | Données MariaDB (`/var/lib/mysql`) | `docker compose down -v` |
| `redis_data` | Données Redis persistées | `docker compose down -v` |

> Le code source Moodle (`./moodle/`) est un **bind mount** (dossier local), pas un volume Docker — il n'est jamais supprimé par Docker.

### Sauvegarde du volume moodledata

```bash
docker run --rm \
  -v lms_moodle_data:/source:ro \
  -v "$(pwd)/backups":/backup \
  alpine tar czf /backup/moodledata_$(date +%Y%m%d).tar.gz -C /source .
```

### Sauvegarde de la base de données

```bash
docker compose exec mariadb \
  mariadb-dump -u moodle -pmoodle_secret_2026 moodle \
  > backups/moodle_db_$(date +%Y%m%d).sql
```

---

## 12. Différences avec la production

| Aspect | Docker local | Production |
|---|---|---|
| OS | Conteneurs Linux (Alpine/Debian) | Ubuntu Server 22.04 LTS |
| Nginx | Conteneur `nginx:1.24-alpine` | Nginx installé nativement |
| PHP | Conteneur `php:8.2-fpm` | PHP 8.2-FPM installé nativement |
| MariaDB | Conteneur `mariadb:10.11` | MariaDB 10.11 sur VM dédiée |
| Driver DB | `mariadb` (natif) | `mariadb` (natif) — identique |
| Redis | Conteneur `redis:7-alpine` | Redis 7.x natif (socket Unix) |
| BigBlueButton | Serveur externe | VM Ubuntu dédiée 8vCPU/16Go |
| SSL | Pas de SSL (HTTP) | Let's Encrypt (certbot) |
| Stockage moodledata | Volume Docker local | SSD 200 Go + backup S3 |
| Cron | Conteneur dédié (boucle bash 60s) | `crontab` système (chaque minute) |
| Monitoring | Logs Docker | Prometheus + Grafana (V2) |
| MFA | Non configuré | `tool_mfa` obligatoire admins |

---

## 13. Passage en production

Voir le document dédié **[PRODUCTION.md](./PRODUCTION.md)** pour le guide complet de déploiement en production (Ubuntu 22.04, Nginx natif, SSL, sécurité, sauvegardes automatiques).

Points essentiels avant le passage en prod :
1. Générer des mots de passe forts (différents de ceux du `.env` local)
2. Utiliser `--dbtype=mariadb` dans le CLI installer (identique au local)
3. Configurer SSL avant l'installation Moodle (le `wwwroot` doit être `https://`)
4. Régler les permissions `moodledata` : `750`, propriétaire `www-data`
5. Activer le MFA pour les comptes administrateurs

---

*Document mis à jour le 30 avril 2026 — Projet RFC Digital / Wave Inc*
