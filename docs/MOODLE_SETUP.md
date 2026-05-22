# Plateforme Nationale de Formation des Enseignants
# Guide d'installation et de mise en place initiale Moodle

> **Version :** 1.0 — Avril 2026
> **Périmètre :** Mise en place initiale (équivalent Sprints 1 + 2 du brief technique).
> Couvre : installation Moodle, configuration globale, plugins, champs profil, rôles
> custom, arborescence de catégories, cours modèles, template SP, séquencement,
> création d'une première session pilote, spec du plugin custom et installation BBB.
> **Hors périmètre :** développement effectif du plugin `local_modulvalidation`,
> Sprints 3 à 6 (configuration BBB en exploitation, reporting Configurable Reports,
> Module 5/jury, recette, MEP, formation utilisateurs).
> **Audience :** administrateur fonctionnel (pas-à-pas UI Moodle) **et**
> installeur / DevOps (commandes CLI, fichiers de config).

---

## Sommaire

- [1. Vue d'ensemble](#1-vue-densemble)
- [2. Variables à fixer avant de commencer](#2-variables-à-fixer-avant-de-commencer)
- [3. Pré-requis](#3-pré-requis)
- [4. Phase A — Installation de Moodle](#4-phase-a--installation-de-moodle)
  - [A.1 Environnement de développement (Docker)](#a1-environnement-de-développement-docker)
  - [A.2 Environnement de production (Ubuntu 22.04)](#a2-environnement-de-production-ubuntu-2204)
- [5. Phase B — Configuration globale du site](#5-phase-b--configuration-globale-du-site)
- [6. Phase C — Installation des plugins requis](#6-phase-c--installation-des-plugins-requis)
- [7. Phase D — Champs personnalisés du profil utilisateur](#7-phase-d--champs-personnalisés-du-profil-utilisateur)
- [8. Phase E — Rôles personnalisés](#8-phase-e--rôles-personnalisés)
- [9. Phase F — Arborescence des catégories de cours](#9-phase-f--arborescence-des-catégories-de-cours)
- [10. Phase G — Création des 5 cours modèles](#10-phase-g--création-des-5-cours-modèles)
- [11. Phase H — Template SP (Situation Professionnelle)](#11-phase-h--template-sp-situation-professionnelle)
- [12. Phase I — Suivi d'achèvement & restrictions d'accès](#12-phase-i--suivi-dachèvement--restrictions-daccès)
- [13. Phase J — Comptes initiaux et imports](#13-phase-j--comptes-initiaux-et-imports)
- [14. Phase K — Création d'une session pilote (Cohorte A — Casablanca)](#14-phase-k--création-dune-session-pilote-cohorte-a--casablanca)
- [15. Phase L — Spécification du plugin `local_modulvalidation`](#15-phase-l--spécification-du-plugin-local_modulvalidation)
- [16. Phase M — Installation BigBlueButton (VM dédiée)](#16-phase-m--installation-bigbluebutton-vm-dédiée)
- [17. Annexes](#17-annexes)

---

## 1. Vue d'ensemble

### 1.1 Architecture cible Moodle

```
Formation Enseignants                          [catégorie racine]
│
├── Modèles                                    [catégorie — cours-types]
│   ├── [MODÈLE] Module 1 — Fondamentaux
│   ├── [MODÈLE] Module 2 — Didactique
│   ├── [MODÈLE] Module 3 — Évaluation
│   ├── [MODÈLE] Module 4 — Innovation pédagogique
│   └── [MODÈLE] Module 5 — Synthèse & Certification
│
└── Sessions                                   [catégorie — sessions live]
    ├── Session 2026-01 — Cohorte A — Casablanca
    │   ├── Module 1, 2, 3, 4, 5             [cours dupliqués depuis Modèles]
    ├── Session 2026-01 — Cohorte B — Rabat
    └── Session 2026-01 — Cohorte C — Marrakech
```

> **Règle structurante :** une session de formation = **1 catégorie + 1 cohorte
> globale + 5 cours** dupliqués depuis les modèles. L'inscription des apprenants
> aux cours se fait exclusivement par **synchronisation cohorte**.

### 1.2 Mécanique de séquencement

Le verrouillage des modules combine deux fonctionnalités natives Moodle :

1. **Activity Completion** sur le cours Module N (toutes les SP doivent être
   achevées).
2. **Restrict Access** sur le cours Module N+1 :
   `Achèvement(Module N) = complete` **ET** `profile_field(validation_module_N) = 1`.

Le champ `validation_module_N` est activé manuellement par le **responsable
pédagogique** via le plugin custom `local_modulvalidation` (voir § 15).

### 1.3 Stack technique cible

| Couche                  | Composant retenu                              |
|-------------------------|-----------------------------------------------|
| LMS                     | Moodle 4.5 LTS                                |
| Serveur web             | Nginx 1.24+                                   |
| Langage                 | PHP 8.2 (FPM)                                 |
| Base de données         | MariaDB 10.11 LTS                             |
| Cache mémoire/sessions  | Redis 7.x                                     |
| Classes virtuelles      | BigBlueButton 2.7+ (serveur dédié, voir § 16) |
| OS prod                 | Ubuntu Server 22.04 LTS                       |

---

## 2. Variables à fixer avant de commencer

Remplir ce tableau **avant** de démarrer l'installation. Chaque ligne TBD doit
être renseignée par le responsable de projet ou le client.

| Variable                       | Valeur retenue                          | Statut    |
|--------------------------------|-----------------------------------------|-----------|
| `WWWROOT_PROD`                 | `https://<DOMAINE_PROD>`                | **TBD**   |
| `WWWROOT_DEV`                  | `http://localhost`                      | OK (Docker) |
| `BBB_DOMAIN`                   | `bbb.<DOMAINE_PROD>`                    | **TBD**   |
| `EMAIL_ADMIN`                  | `admin@<DOMAINE_PROD>`                  | **TBD**   |
| `EMAIL_NOREPLY`                | `noreply@<DOMAINE_PROD>`                | **TBD**   |
| Mot de passe admin initial     | `Admin@2026!` (à changer après MEP)     | OK        |
| Mot de passe BDD Moodle        | `moodle_secret_2026` (à régénérer prod) | **TBD prod** |
| Logo client                    | Fichier PNG/SVG `logo-client.png`       | **À fournir** |
| Couleur primaire (charte)      | `#______`                               | **À fournir** |
| Couleur secondaire (charte)    | `#______`                               | **À fournir** |
| Police principale              | `Arial / Roboto / Open Sans / …`        | **À fournir** |
| Liste des établissements       | (menu déroulant champ profil)           | **À compléter** |
| Liste des villes               | (menu déroulant champ profil)           | **À compléter** |

---

## 3. Pré-requis

### 3.1 Pré-requis humains

- 1 **administrateur technique** disposant d'un accès root/sudo serveur prod et
  des droits administrateur Moodle.
- 1 **administrateur fonctionnel** habilité à créer les sessions, cohortes,
  catégories.
- Validation client de la charte graphique (logo + couleurs + police).

### 3.2 Pré-requis matériels (cibles Sprint 1+2)

| Environnement | CPU      | RAM   | Disque         | Notes                                      |
|---------------|----------|-------|----------------|--------------------------------------------|
| Dev local     | 4 vCPU   | 8 Go  | 30 Go SSD      | Docker Desktop ou Linux + docker-compose   |
| Prod app      | 8 vCPU   | 16 Go | 200 Go SSD     | Moodle + Nginx + PHP-FPM + Redis           |
| Prod BDD      | 4 vCPU   | 16 Go | 100 Go SSD     | MariaDB 10.11 (mutualisable phase initiale)|
| Prod BBB      | 8 vCPU   | 16 Go | 100 Go SSD     | + bande passante 1 Gbps (cf. § 16)         |

### 3.3 Pré-requis logiciels

- Docker Engine 24+ et docker-compose v2 (dev) **ou** Ubuntu 22.04 + accès root (prod).
- Git.
- Un éditeur de texte côté serveur (vim/nano).
- Domaines DNS pointant vers les IP des serveurs (prod uniquement).

---

## 4. Phase A — Installation de Moodle

### A.1 Environnement de développement (Docker)

La stack Docker fournie dans le repo (`docker-compose.yml`) embarque Nginx +
PHP-FPM 8.2 + MariaDB 10.11 + Redis + outils optionnels (phpMyAdmin, Mailpit).

#### A.1.1 Démarrer la stack

```bash
# Depuis la racine du projet
cp .env.example .env                 # ajuster si besoin
docker compose up -d                 # base : nginx + php + mariadb + redis + cron
docker compose --profile tools up -d # ajoute phpmyadmin + mailpit
```

#### A.1.2 Vérifier la connectivité

```bash
docker compose ps
curl -I http://localhost              # doit répondre 200/302
```

| Service        | URL locale                  | Identifiants par défaut                     |
|----------------|-----------------------------|---------------------------------------------|
| Moodle         | http://localhost            | (à créer à l'install — voir A.1.3)          |
| phpMyAdmin     | http://localhost:8080       | server `mariadb`, user `root`, pwd `root_secret_2026` |
| Mailpit        | http://localhost:8025       | —                                           |

#### A.1.3 Lancer l'installation Moodle (CLI dans le conteneur PHP)

L'installation web (`http://localhost/install.php`) fonctionne aussi, mais le
mode CLI est reproductible et scriptable :

```bash
docker compose exec -u www-data php \
  php /var/www/html/admin/cli/install_database.php \
    --lang=fr \
    --adminuser=admin \
    --adminpass='Admin@2026!' \
    --adminemail=admin@formation.ma \
    --fullname='Plateforme Nationale Formation Enseignants' \
    --shortname='PNFE' \
    --summary='Programme de formation des enseignants — Maroc' \
    --agree-license
```

> ⚠️ Le `config.php` est déjà présent dans `moodle/config.php` (BDD `mariadb`,
> Redis sessions). Si le fichier est absent ou modifié, lancer d'abord
> `admin/cli/install.php` (génère `config.php` puis crée la BDD).

#### A.1.4 Premier login

1. Ouvrir `http://localhost`.
2. Se connecter avec `admin / Admin@2026!`.
3. Compléter le profil admin (email vérifié, langue par défaut FR).
4. Configurer le mot de passe permanent.

### A.2 Environnement de production (Ubuntu 22.04)

#### A.2.1 Mise à jour système et paquets de base

```bash
sudo apt update && sudo apt -y upgrade
sudo apt -y install nginx mariadb-server redis-server git unzip \
  software-properties-common ca-certificates curl
```

#### A.2.2 PHP 8.2 + extensions

```bash
sudo add-apt-repository -y ppa:ondrej/php
sudo apt update
sudo apt -y install php8.2 php8.2-fpm php8.2-cli php8.2-common \
  php8.2-mysql php8.2-curl php8.2-gd php8.2-intl php8.2-mbstring \
  php8.2-soap php8.2-xml php8.2-xmlrpc php8.2-zip php8.2-bcmath \
  php8.2-redis php8.2-opcache php8.2-exif php8.2-ldap
```

#### A.2.3 Configuration MariaDB

```bash
sudo mysql_secure_installation
sudo mysql -u root -p <<'SQL'
CREATE DATABASE moodle CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'moodle'@'localhost' IDENTIFIED BY 'CHANGEME_STRONG_PASS';
GRANT ALL PRIVILEGES ON moodle.* TO 'moodle'@'localhost';
FLUSH PRIVILEGES;
SQL
```

Recommandations dans `/etc/mysql/mariadb.conf.d/99-moodle.cnf` :

```ini
[mysqld]
innodb_file_per_table = ON
innodb_buffer_pool_size = 4G
max_allowed_packet = 64M
character-set-server = utf8mb4
collation-server = utf8mb4_unicode_ci
```

#### A.2.4 Récupération du code Moodle

```bash
sudo mkdir -p /var/www && cd /var/www
sudo git clone --branch MOODLE_405_STABLE --depth 1 \
  https://github.com/moodle/moodle.git html
sudo mkdir -p /var/www/moodledata
sudo chown -R www-data:www-data /var/www/html /var/www/moodledata
sudo chmod -R 0750 /var/www/moodledata
```

#### A.2.5 Installation CLI

```bash
sudo -u www-data /usr/bin/php8.2 /var/www/html/admin/cli/install.php \
  --lang=fr \
  --wwwroot=https://<DOMAINE_PROD> \
  --dataroot=/var/www/moodledata \
  --dbtype=mariadb \
  --dbhost=localhost \
  --dbname=moodle \
  --dbuser=moodle \
  --dbpass='CHANGEME_STRONG_PASS' \
  --fullname='Plateforme Nationale Formation Enseignants' \
  --shortname='PNFE' \
  --adminuser=admin \
  --adminpass='Admin@2026!' \
  --adminemail=admin@<DOMAINE_PROD> \
  --agree-license \
  --non-interactive
```

#### A.2.6 Configuration Nginx

`/etc/nginx/sites-available/moodle.conf` :

```nginx
server {
    listen 80;
    server_name <DOMAINE_PROD>;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    server_name <DOMAINE_PROD>;
    root /var/www/html;
    index index.php index.html;

    ssl_certificate     /etc/letsencrypt/live/<DOMAINE_PROD>/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/<DOMAINE_PROD>/privkey.pem;

    client_max_body_size 200M;

    location / { try_files $uri $uri/ /index.php?$query_string; }

    location ~ [^/]\.php(/|$) {
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_pass unix:/run/php/php8.2-fpm.sock;
        fastcgi_index index.php;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param PATH_INFO $fastcgi_path_info;
    }
}
```

```bash
sudo ln -s /etc/nginx/sites-available/moodle.conf /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
```

#### A.2.7 Certificat SSL (Let's Encrypt)

```bash
sudo apt -y install certbot python3-certbot-nginx
sudo certbot --nginx -d <DOMAINE_PROD> -m <EMAIL_ADMIN> --agree-tos --redirect
```

#### A.2.8 Sessions Redis

Ajouter à `/var/www/html/config.php` (avant `require_once .../setup.php`) :

```php
$CFG->session_handler_class          = '\\core\\session\\redis';
$CFG->session_redis_host             = '127.0.0.1';
$CFG->session_redis_port             = 6379;
$CFG->session_redis_database         = 0;
$CFG->session_redis_prefix           = 'mdl_sess_';
$CFG->session_redis_acquire_lock_timeout = 120;
$CFG->session_redis_lock_expire      = 7200;
```

#### A.2.9 Cron Moodle (impératif)

```bash
sudo crontab -u www-data -e
# Ajouter :
* * * * * /usr/bin/php8.2 /var/www/html/admin/cli/cron.php >/dev/null
```

> ⚠️ **Sans cron actif, les notifications, sauvegardes auto, complétions
> programmées et envoi de mails ne fonctionnent pas.**

---

## 5. Phase B — Configuration globale du site

> **Pré-requis :** connecté en tant qu'administrateur Moodle.

### 5.1 Langue et fuseau horaire

`Administration du site > Langue > Paquets de langue`

- Installer le pack **Français (fr)**.
- `Administration du site > Langue > Paramètres de langue` : *Langue par défaut* = `Français (fr)`.
- Décocher *Afficher le menu de langue* (V1 mono-langue).

`Administration du site > Localisation > Paramètres de localisation`

- *Pays par défaut* = `Maroc`.
- *Fuseau horaire par défaut* = `Africa/Casablanca`.

### 5.2 Activer les fonctionnalités avancées

`Administration du site > Fonctionnalités avancées`

| Paramètre                           | Valeur |
|-------------------------------------|--------|
| Activer le suivi d'achèvement       | **Oui** |
| Activer l'achèvement du cours       | **Oui** |
| Activer les restrictions d'accès    | **Oui** |
| Activer les rubriques               | Oui    |
| Activer les badges (V1)             | Non    |
| Activer les portfolios              | Non    |
| Activer les compétences             | Non (V1) |

### 5.3 Politique de mots de passe

`Administration du site > Sécurité > Politique des mots de passe`

| Paramètre                              | Valeur     |
|----------------------------------------|------------|
| Longueur minimale                      | 10         |
| Au moins un chiffre                    | Oui        |
| Au moins une majuscule                 | Oui        |
| Au moins un caractère spécial          | Oui        |
| Au moins une minuscule                 | Oui        |
| Mots de passe consécutifs identiques   | Refuser les 5 derniers |
| Verrouillage après échecs              | 5 tentatives → 30 min |

### 5.4 Authentification

`Administration du site > Plugins > Authentification > Gérer l'authentification`

- **Activer** : *Comptes manuels* (Manual accounts) — méthode unique en V1.
- **Désactiver** : *Self-registration* (auto-inscription).
  Sur la page, *Self-registration* = **Désactivée**.
- *Empêcher la création de comptes utilisateur* = **Oui**.
- SSO/LDAP : prévu en V2, ne rien activer en V1.

### 5.5 Inscription par défaut sur les cours

`Administration du site > Plugins > Inscription > Gérer les méthodes d'inscription`

- **Activer** : *Synchronisation des cohortes* (`enrol_cohort`).
- **Désactiver** : *Auto-inscription* (`enrol_self`) et *Accès anonyme* (`enrol_guest`).

### 5.6 Politique de confidentialité (CNDP — loi 09-08)

`Administration du site > Utilisateurs > Confidentialité et politiques`

1. *Politiques du site* → *Nouvelle politique* :
   - Type : Politique de confidentialité.
   - Public : Tous les utilisateurs.
   - Statut : Actif.
   - Coller le contenu fourni par le client (à compléter).
2. *Acceptation obligatoire* à la première connexion : **Oui**.

### 5.7 Paramètres SMTP (envois d'email)

`Administration du site > Serveur > Email > Configuration de la messagerie sortante`

| Paramètre        | Valeur dev (Mailpit)        | Valeur prod                      |
|------------------|-----------------------------|----------------------------------|
| Hôte SMTP        | `mailpit:1025`              | (fourni par client / Mailgun / SES) |
| Sécurité SMTP    | Aucune (dev) / TLS (prod)   | TLS                              |
| Identifiant SMTP | (vide)                      | (fourni)                         |
| Adresse expéditeur | `noreply@formation.local` | `<EMAIL_NOREPLY>`                |

Tester avec : *Envoyer un test* depuis l'interface.

### 5.8 Logs et traçabilité

`Administration du site > Plugins > Stores de log > Gérer les stores de log`

- *Standard log* (logstore_standard) : **Activé**.
- `Administration du site > Serveur > Tâches programmées > Nettoyage des logs` :
  - Rétention : **18 mois** minimum (36 mois recommandé pour la traçabilité
    certification).

### 5.9 Apparence

`Administration du site > Apparence > Logos`

- Charger le logo client (`logo-client.png` ou SVG, **à fournir**).
- Logo compact (favicon).

`Administration du site > Apparence > Thèmes > Boost`

- Couleur principale, couleur de fond : valeurs charte client (**à fournir**).
- Police : valeur charte client.
- Page d'accueil : sélectionner un layout simple, ajouter un message
  d'accueil + image bandeau.

### 5.10 Page d'accueil (frontpage)

`Administration du site > Page d'accueil > Réglages de la page d'accueil`

- *Nom complet* : `Plateforme Nationale de Formation des Enseignants`.
- *Nom court* : `PNFE`.
- *Page d'accueil pour utilisateurs non connectés* : Page combinée (texte +
  description du programme).
- *Page d'accueil pour utilisateurs connectés* : Tableau de bord (`dashboard`).

---

## 6. Phase C — Installation des plugins requis

### 6.1 Liste des plugins à installer

| Plugin                | Identifiant                | Type     | V1     | Rôle                                 |
|-----------------------|----------------------------|----------|--------|--------------------------------------|
| BigBlueButtonBN       | `mod_bigbluebuttonbn`      | Activity | Requis | Classes virtuelles synchrones        |
| Attendance            | `mod_attendance`           | Activity | Requis | Gestion des présences                |
| Custom Certificate    | `mod_customcert`           | Activity | Requis | Génération PDF + QR code             |
| Configurable Reports  | `block_configurable_reports` | Block  | Requis | Rapports SQL custom                  |
| Completion Progress   | `block_completion_progress` | Block   | Requis | Barre de progression apprenant       |
| Scheduler             | `mod_scheduler`            | Activity | Requis | Créneaux de soutenance (Module 5)    |
| Checklist             | `mod_checklist`            | Activity | Requis | Listes de contrôle pédagogiques      |
| Group Choice          | `mod_choicegroup`          | Activity | Optionnel | Auto-inscription aux groupes      |
| PoodLL Filter         | `filter_poodll`            | Filter   | Optionnel | Feedback audio/vidéo              |

> **Natifs Moodle 4.5 (rien à installer) :** H5P (Content Bank), Course
> Overview (`block_myoverview`), MFA (`tool_mfa`).

### 6.2 Installation via interface (par plugin)

Pour chaque plugin :

1. Télécharger le ZIP correspondant à **Moodle 4.5** depuis
   <https://moodle.org/plugins/>.
2. `Administration du site > Plugins > Installer des plugins`.
3. Glisser-déposer le ZIP, valider la zone de destination.
4. *Installer le plugin* → suivre l'assistant → exécuter le script de mise à
   niveau de la BDD.
5. Vérifier le statut sur `Administration du site > Plugins > Vue d'ensemble
   des plugins`.

### 6.3 Installation via CLI (recommandé en prod)

```bash
# Exemple pour Attendance — adapter pour chaque plugin
cd /tmp && wget -O attendance.zip \
  'https://moodle.org/plugins/download.php/.../mod_attendance_moodle45.zip'

sudo unzip -d /var/www/html/mod attendance.zip
sudo chown -R www-data:www-data /var/www/html/mod/attendance

sudo -u www-data /usr/bin/php8.2 /var/www/html/admin/cli/upgrade.php \
  --non-interactive
```

| Destination ZIP                          | Type          |
|------------------------------------------|---------------|
| `/var/www/html/mod/<nom>/`               | Activity      |
| `/var/www/html/blocks/<nom>/`            | Block         |
| `/var/www/html/filter/<nom>/`            | Filter        |
| `/var/www/html/local/<nom>/`             | Local plugin  |

> 💡 Sur Docker dev, exécuter dans le conteneur :
> `docker compose exec -u www-data php php /var/www/html/admin/cli/upgrade.php --non-interactive`

### 6.4 Vérification après installation

`Administration du site > Plugins > Vue d'ensemble des plugins` : tous les
plugins ci-dessus doivent apparaître en **vert** (à jour) sans erreur de
version.

---

## 7. Phase D — Champs personnalisés du profil utilisateur

`Administration du site > Utilisateurs > Comptes > Champs profil utilisateur`

### 7.1 Créer les catégories de champs

Cliquer *Créer une nouvelle catégorie* pour chacune :

| Ordre | Nom de catégorie     |
|-------|----------------------|
| 1     | `Identité`           |
| 2     | `Progression`        |
| 3     | `Certification`      |

### 7.2 Créer les champs

Cliquer *Créer un nouveau champ profil* pour chacun.

#### 7.2.1 Champs de progression

| Nom court              | Nom affiché                                | Type     | Catégorie     | Verrouillage |
|------------------------|--------------------------------------------|----------|---------------|--------------|
| `validation_module_1`  | Module 1 validé par responsable pédago     | Checkbox | Progression   | Oui (admin)  |
| `validation_module_2`  | Module 2 validé par responsable pédago     | Checkbox | Progression   | Oui (admin)  |
| `validation_module_3`  | Module 3 validé par responsable pédago     | Checkbox | Progression   | Oui (admin)  |
| `validation_module_4`  | Module 4 validé par responsable pédago     | Checkbox | Progression   | Oui (admin)  |
| `validation_module_5`  | Module 5 validé par responsable pédago     | Checkbox | Progression   | Oui (admin)  |
| `certification_obtenue`| Certificat final délivré                   | Checkbox | Certification | Oui (admin)  |

> Pour chacun : *Visible* = **Pas visible** (réservé interne) ; *Modifiable par
> l'utilisateur* = **Non** ; *Champ requis* = **Non** ; *Valeur par défaut* = **Non coché**.

#### 7.2.2 Champs d'identité

| Nom court       | Nom affiché              | Type           | Catégorie | Notes |
|-----------------|--------------------------|----------------|-----------|-------|
| `etablissement` | Établissement d'origine  | Menu déroulant | Identité  | Liste à compléter par le client (cf. § 2). Format Moodle : un item par ligne. |
| `ville`         | Ville de l'établissement | Menu déroulant | Identité  | Liste à compléter par le client.                                              |

> *Visible* = **Visible à tout le monde** ; *Champ requis* = **Oui** ;
> *Modifiable par l'utilisateur* = **Non** (saisi par l'admin via import CSV).

### 7.3 Vérification

Créer un utilisateur de test → l'écran d'édition doit présenter les 8 champs
classés sous les 3 catégories. Les checkbox de validation ne sont visibles que
pour les administrateurs/responsables (via le plugin custom — § 15).

---

## 8. Phase E — Rôles personnalisés

### 8.1 Création du rôle `responsable_pedago`

`Administration du site > Utilisateurs > Permissions > Définir les rôles >
Ajouter un nouveau rôle`

| Champ                        | Valeur                                                     |
|------------------------------|------------------------------------------------------------|
| Utiliser un préréglage de rôle | *Aucun* (créer de zéro) ou copier depuis *Manager* puis adapter |
| Nom complet                  | Responsable pédagogique                                    |
| Nom court                    | `responsable_pedago`                                       |
| Description                  | Valide la progression des apprenants module par module et accède au tableau de bord de validation. |
| Archétype de rôle            | *Aucun*                                                    |
| Types de contexte autorisés  | **Catégorie de cours**                                     |

Capabilities à activer (= *Allow*) — extrait Annexe A du brief :

```
moodle/category:viewcourselist
moodle/course:view
moodle/course:viewhiddencourses
moodle/course:viewparticipants
moodle/site:viewparticipants
gradereport/grader:view
gradereport/user:view
report/completion:view
report/courseoverview:view
report/log:view
mod/attendance:view
mod/attendance:viewreports
moodle/user:viewdetails
moodle/user:viewalldetails
moodle/user:update
local/modulvalidation:validate    (existe après install du plugin custom)
local/modulvalidation:view        (existe après install du plugin custom)
```

### 8.2 Création du rôle `jury_member`

Idem § 8.1, valeurs :

| Champ                       | Valeur                                                |
|-----------------------------|-------------------------------------------------------|
| Nom complet                 | Membre du jury                                        |
| Nom court                   | `jury_member`                                         |
| Description                 | Consulte les mémoires de synthèse, remplit la grille d'évaluation jury et participe à la délibération. |
| Archétype                   | *Aucun*                                               |
| Types de contexte autorisés | **Cours** (uniquement Module 5)                       |

Capabilities :

```
moodle/course:view
moodle/course:viewparticipants
mod/assign:view
mod/assign:viewblinddetails
mod/assign:grade
mod/assign:reviewotherusers
gradereport/user:view
moodle/user:viewdetails
mod/forum:viewdiscussion
mod/forum:replypost
```

### 8.3 Vérification

`Administration du site > Utilisateurs > Permissions > Définir les rôles` : les
deux rôles apparaissent dans la liste avec leurs noms courts.

---

## 9. Phase F — Arborescence des catégories de cours

`Administration du site > Cours > Gérer les cours et catégories`

### 9.1 Étapes

1. *Créer une catégorie* à la racine :
   - Nom : `Formation Enseignants`
   - Identifiant : `FE`
   - Description : *Catégorie racine du programme national de formation des enseignants.*

2. À l'intérieur de `Formation Enseignants`, créer 2 sous-catégories :
   - `Modèles` (id `FE_MODELES`) — *Cours-types réutilisables (non visibles des apprenants).*
   - `Sessions` (id `FE_SESSIONS`) — *Sessions de formation actives (catégories filles : une par cohorte).*

### 9.2 Visibilité

- `Formation Enseignants` : **Visible**.
- `Modèles` : **Cachée** (Settings → Visibilité = Non visible). Les apprenants
  ne doivent jamais voir les modèles.
- `Sessions` : **Visible**.

### 9.3 Résultat attendu

```
Formation Enseignants                     [visible]
├── Modèles                               [cachée]
└── Sessions                              [visible]
```

---

## 10. Phase G — Création des 5 cours modèles

> **Objectif :** créer 5 cours **vides** dans `Modèles` qui serviront de base
> de duplication pour chaque session future.

### 10.1 Créer chaque cours modèle

`Cours > Gérer les cours et catégories > Modèles > Créer un nouveau cours`

| #   | Nom complet                                        | Nom court (idnumber) | Format        | Visibilité |
|-----|----------------------------------------------------|----------------------|---------------|------------|
| 1   | [MODÈLE] Module 1 — Fondamentaux                   | `MOD_M1`             | Sections (Topics) | Cachée |
| 2   | [MODÈLE] Module 2 — Didactique                     | `MOD_M2`             | Sections (Topics) | Cachée |
| 3   | [MODÈLE] Module 3 — Évaluation                     | `MOD_M3`             | Sections (Topics) | Cachée |
| 4   | [MODÈLE] Module 4 — Innovation pédagogique         | `MOD_M4`             | Sections (Topics) | Cachée |
| 5   | [MODÈLE] Module 5 — Synthèse & Certification       | `MOD_M5`             | Sections (Topics) | Cachée |

### 10.2 Paramètres communs à chaque cours modèle

`Paramètres du cours` :

| Paramètre                                      | Valeur                                |
|------------------------------------------------|---------------------------------------|
| Format de cours                                | **Sections (Topics)**                 |
| Nombre de sections                             | **1** (placeholder, sera dupliqué)    |
| Affichage des sections                         | Une section par page                  |
| Visibilité                                     | Cachée                                |
| Suivi d'achèvement                             | **Activé**                            |
| Condition d'achèvement du cours                | *Achèvement de toutes les activités*  |
| Méthode d'inscription par défaut               | *Synchronisation des cohortes* uniquement |
| Auto-inscription / Accès anonyme               | Désactivés                            |

Modules supplémentaires à activer dans `Paramètres > Suivi d'achèvement` :
*Activer les conditions d'accès basées sur le profil utilisateur*.

> Le contenu pédagogique réel (SP, vidéos, H5P, quiz, devoirs) sera ajouté
> ultérieurement dans chaque cours modèle au fil de la production des contenus,
> en s'appuyant sur le **Template SP** (§ 11). Pour la mise en place initiale,
> on crée 1 SP de démonstration uniquement dans le Module 1 modèle.

---

## 11. Phase H — Template SP (Situation Professionnelle)

> **Objectif :** définir une structure standard de SP, créer une SP de
> démonstration dans le Module 1 modèle, sauvegarder cette structure en `.mbz`
> et restaurer dans les autres modules.

### 11.1 Composition standard d'une SP

Chaque SP = 1 section Moodle contenant exactement, dans cet ordre :

| #   | Type d'activité | Nom suggéré              | Rôle                                                               |
|-----|-----------------|--------------------------|--------------------------------------------------------------------|
| 1   | Page            | Introduction & Objectifs | Contexte, objectifs pédagogiques, volume horaire (HTML structuré). |
| 2   | Dossier         | Supports documentaires   | PDF, présentations, documents Word.                                |
| 3   | URL / Étiquette | Vidéos pédagogiques      | YouTube non listé, Vimeo, ou upload <200 Mo.                       |
| 4   | H5P             | Activité interactive     | Optionnel — Interactive Video, Branching Scenario, etc.            |
| 5   | Forum           | Échanges & questions     | Optionnel — discussions sur la SP.                                 |
| 6   | Quiz **OU** Devoir | Évaluation de la SP    | **Une seule** activité d'évaluation par SP.                        |

### 11.2 Création de la SP de démonstration (dans `[MODÈLE] Module 1`)

1. Ouvrir le cours `[MODÈLE] Module 1 — Fondamentaux` → *Activer le mode édition*.
2. Renommer la section 1 → **`SP 1.1 — Démo`**.
3. Ajouter dans cet ordre :

#### A. Page « Introduction & Objectifs »

`+ Ajouter une activité ou une ressource > Page`

| Paramètre               | Valeur                                                         |
|-------------------------|----------------------------------------------------------------|
| Nom                     | Introduction & Objectifs                                       |
| Description             | (vide)                                                         |
| Contenu                 | HTML modèle (cf. § 11.4)                                       |
| Suivi d'achèvement      | Achèvement automatique : *L'étudiant doit consulter cette activité* |

#### B. Dossier « Supports documentaires »

`+ Ajouter > Dossier`

| Paramètre          | Valeur                                          |
|--------------------|-------------------------------------------------|
| Nom                | Supports documentaires                          |
| Affichage          | Afficher le contenu du dossier sur la page      |
| Sous-dossiers      | Afficher                                        |
| Suivi d'achèvement | Manuel (l'étudiant marque comme fait)           |

#### C. URL « Vidéo pédagogique »

`+ Ajouter > URL`

| Paramètre          | Valeur                                                  |
|--------------------|---------------------------------------------------------|
| Nom                | Vidéo de la SP                                          |
| URL externe        | (lien YouTube non listé / Vimeo — à remplacer)          |
| Affichage          | Intégrer (embed)                                        |
| Suivi d'achèvement | *L'étudiant doit consulter cette activité*              |

#### D. Activité H5P (optionnelle)

`+ Ajouter > H5P`

| Paramètre          | Valeur                                                 |
|--------------------|--------------------------------------------------------|
| Nom                | Activité interactive                                   |
| Source             | Content Bank (à créer dans `Contenus > Banque de contenus`) |
| Suivi d'achèvement | *L'étudiant doit terminer l'activité*                  |

#### E. Forum (optionnel)

`+ Ajouter > Forum`

| Paramètre  | Valeur                                |
|------------|---------------------------------------|
| Nom        | Échanges & questions                  |
| Type       | Forum standard pour utilisation générale |

#### F. Évaluation — au choix : Quiz OU Devoir

##### F.1 Quiz (paramètres recommandés par défaut, cf. brief § 7.2)

`+ Ajouter > Test`

| Paramètre                  | Valeur                                                |
|----------------------------|-------------------------------------------------------|
| Nom                        | Évaluation SP 1.1                                     |
| Tentatives autorisées      | **2**                                                 |
| Méthode de notation        | Note la plus élevée                                   |
| Note pour passer           | **70%**                                               |
| Ordre des questions        | Aléatoire                                             |
| Comportement des questions | Feedback différé                                      |
| Durée limite               | 30 à 60 min (à fixer par le formateur)                |
| Suivi d'achèvement         | *Exiger une note* + *Note de passage* requise         |

##### F.2 Devoir / Assignment (alternative)

`+ Ajouter > Devoir`

| Paramètre                | Valeur                                                  |
|--------------------------|---------------------------------------------------------|
| Nom                      | Évaluation SP 1.1                                       |
| Types de remise          | Texte en ligne **et** Remises de fichiers               |
| Nombre max de fichiers   | 3                                                       |
| Taille max par fichier   | 20 Mo                                                   |
| Type de note             | Note (échelle **0 — 20**)                               |
| Méthode d'évaluation     | Rubrique (à définir : critères + niveaux + points)      |
| Feedback                 | Commentaires + annotations PDF                          |
| Suivi d'achèvement       | *Achevé lorsque la note est ≥ 10/20*                    |

### 11.3 Sauvegarder la SP en template `.mbz`

1. Dans `[MODÈLE] Module 1` → *Sauvegarde* (icône engrenage).
2. Choix initiaux : décocher *Inclure les utilisateurs inscrits*, *Inclure les
   journaux* ; **cocher** uniquement la section `SP 1.1 — Démo` et ses activités.
3. Lancer la sauvegarde → télécharger le fichier `.mbz`.
4. Renommer en `SP_TEMPLATE.mbz` et l'archiver dans le repo Git
   (`/templates/SP_TEMPLATE.mbz`).

### 11.4 Modèle HTML pour la page « Introduction & Objectifs »

```html
<h2>Introduction</h2>
<p>[Contexte de la situation professionnelle traitée — à compléter par le formateur]</p>

<h2>Objectifs pédagogiques</h2>
<ul>
  <li>Objectif 1</li>
  <li>Objectif 2</li>
  <li>Objectif 3</li>
</ul>

<h2>Volume horaire</h2>
<table border="1" cellpadding="6" cellspacing="0">
  <tr><th>Modalité</th><th>Durée</th></tr>
  <tr><td>Présentiel</td><td>… h</td></tr>
  <tr><td>Distanciel synchrone (BBB)</td><td>… h</td></tr>
  <tr><td>Distanciel asynchrone</td><td>… h</td></tr>
  <tr><th>Total</th><th>… h</th></tr>
</table>
```

### 11.5 Restauration du template dans les autres modules

Pour chaque module modèle (Module 2 à 5) :

1. Ouvrir le cours.
2. Engrenage → *Restaurer* → uploader `SP_TEMPLATE.mbz`.
3. Choisir *Fusionner la sauvegarde dans ce cours* → *Continuer*.
4. La section `SP 1.1 — Démo` apparaît : la **renommer** en `SP X.1 — Démo`
   (X = numéro du module).

---

## 12. Phase I — Suivi d'achèvement & restrictions d'accès

### 12.1 Achèvement par activité (à faire pour chaque activité de chaque SP)

Dans les paramètres de l'activité, section *Suivi d'achèvement* :

| Activité                  | Condition par défaut                                 |
|---------------------------|------------------------------------------------------|
| Page (Intro & Objectifs)  | *L'étudiant doit consulter*                          |
| Dossier (Supports)        | Manuel                                               |
| URL (Vidéo)               | *L'étudiant doit consulter*                          |
| H5P                       | *L'étudiant doit terminer l'activité*                |
| Forum                     | Manuel                                               |
| Quiz                      | *Exiger une note* + score ≥ note pour passer (70%)   |
| Devoir                    | *Exiger une note* (ex. ≥ 10/20)                      |

### 12.2 Achèvement du cours (chaque cours modèle Module 1 à 5)

`Paramètres du cours > Suivi d'achèvement`

- *Achèvement du cours* = *L'ensemble des conditions sélectionnées doivent
  être remplies*.
- Cocher **Achèvement des activités** → cocher toutes les activités
  d'évaluation des SP.

### 12.3 Restrictions d'accès aux modules N+1 (sur les cours modèles **et** sessions)

Pour le cours **`[MODÈLE] Module 2 — Didactique`** :

`Paramètres du cours > Restrictions d'accès > Ajouter une restriction`

```
Doit correspondre à TOUTES les conditions :
  - Achèvement de l'activité : "[MODÈLE] Module 1 — Fondamentaux"  =  doit être marqué comme achevé
  - Champ du profil utilisateur : validation_module_1               =  est égal à  1
```

Identique sur Module 3 (référence Module 2 + `validation_module_2`), Module 4
(Module 3 + `validation_module_3`), Module 5 (Module 4 + `validation_module_4`).

> ⚠️ **Important :** la restriction *Achèvement de l'activité* qui pointe vers
> un autre cours utilise la condition Moodle native *Course completion*. Pour
> que cela fonctionne, l'achèvement du cours Module N doit être configuré (§ 12.2).

### 12.4 Bloc *Completion Progress* sur le tableau de bord

`Tableau de bord > Personnaliser cette page > Ajouter un bloc > Completion Progress`

- Configurer le bloc pour afficher la barre de progression du cours en cours.
- Sauvegarder la configuration par défaut du *Dashboard* (Administration du
  site > Apparence > Pages par défaut > Tableau de bord).

---

## 13. Phase J — Comptes initiaux et imports

### 13.1 Comptes de test à créer (10 comptes — cf. brief Sprint 1)

| Login   | Prénom    | Nom        | Email                          | Rôle attribué                      |
|---------|-----------|------------|--------------------------------|------------------------------------|
| `admin` | Admin     | Système    | `<EMAIL_ADMIN>`                | Administrateur du site (créé à l'install) |
| `ADM001`| Admin     | Fonctionnel| `adm001@formation.local`       | Manager (sur catégorie racine)     |
| `FOR001`| Formateur | Test 1     | `for001@formation.local`       | Teacher (à affecter par cours)     |
| `FOR002`| Formateur | Test 2     | `for002@formation.local`       | Teacher (à affecter par cours)     |
| `RP001` | Resp.     | Pédago     | `rp001@formation.local`        | `responsable_pedago` (sur catégorie session) |
| `JUR001`| Jury      | Membre 1   | `jur001@formation.local`       | `jury_member` (sur cours Module 5) |
| `JUR002`| Jury      | Membre 2   | `jur002@formation.local`       | `jury_member` (sur cours Module 5) |
| `ENS001`| Ahmed     | Benali     | `ens001@formation.local`       | Student (via cohorte)              |
| `ENS002`| Fatima    | Alaoui     | `ens002@formation.local`       | Student (via cohorte)              |
| `ENS003`| Karim     | Idrissi    | `ens003@formation.local`       | Student (via cohorte)              |

> Mot de passe initial à imposer : `Test@2026!` (forcer changement à la première connexion).

### 13.2 Création manuelle (interface)

`Administration du site > Utilisateurs > Comptes > Ajouter un utilisateur`

Pour chaque compte, renseigner : login, mot de passe, prénom, nom, email,
ville, pays = `Maroc`, langue = `Français`, et — pour les apprenants — les
champs `etablissement` et `ville` (depuis le menu déroulant).

### 13.3 Création par import CSV (recommandé pour les apprenants)

`Administration du site > Utilisateurs > Comptes > Importer des utilisateurs`

#### Format CSV attendu (UTF-8, séparateur virgule)

```csv
username,firstname,lastname,email,password,city,country,profile_field_etablissement,profile_field_ville,cohort1
ENS001,Ahmed,Benali,ens001@formation.local,Test@2026!,Casablanca,MA,École privée X,Casablanca,COHORTE_2026_01_CASA
ENS002,Fatima,Alaoui,ens002@formation.local,Test@2026!,Rabat,MA,École privée Y,Rabat,COHORTE_2026_01_CASA
ENS003,Karim,Idrissi,ens003@formation.local,Test@2026!,Marrakech,MA,École privée Z,Marrakech,COHORTE_2026_01_CASA
```

#### Paramètres d'import

| Paramètre                                 | Valeur                                |
|-------------------------------------------|---------------------------------------|
| Type d'envoi de fichier                   | Fichier CSV                           |
| Encodage                                  | UTF-8                                 |
| Délimiteur CSV                            | Virgule                               |
| Action en cas de doublons                 | Ajouter, sauf si le compte existe déjà|
| Nouveau mot de passe                      | Forcer le changement à la connexion   |

> 💡 Le champ `cohort1` permet d'ajouter automatiquement l'utilisateur à une
> cohorte **déjà existante** dont le `idnumber` correspond. Créer la cohorte
> **avant** l'import (cf. § 14).

### 13.4 Affectation des rôles globaux

| Compte    | Action                                                                |
|-----------|------------------------------------------------------------------------|
| `ADM001`  | `Administration du site > Utilisateurs > Permissions > Attribuer des rôles système` → ne **pas** lui donner Administrateur du site (réserver à l'admin technique). En revanche : sur la catégorie `Formation Enseignants` lui attribuer le rôle natif **Manager**. |
| `FOR001/2`| Affectation par cours, lors de la création de session (§ 14.4).        |
| `RP001`   | À affecter au niveau de la **catégorie de session** lors de sa création (§ 14.5). |
| `JUR001/2`| À affecter sur le **cours Module 5** de la session, après création.    |
| `ENS001-3`| Inscrits via cohorte (synchronisation cohorte sur les 5 cours).        |

---

## 14. Phase K — Création d'une session pilote (Cohorte A — Casablanca)

> **Objectif :** dérouler de bout en bout la création d'**une** session de
> formation. Les sessions Cohorte B (Rabat) et Cohorte C (Marrakech) se créent
> en répétant les mêmes étapes avec les noms et idnumbers adaptés.

### 14.1 Créer la sous-catégorie de session

`Cours > Gérer les cours et catégories > Sessions > Créer une catégorie`

| Paramètre        | Valeur                                              |
|------------------|-----------------------------------------------------|
| Nom              | Session 2026-01 — Cohorte A — Casablanca            |
| Identifiant (idnumber) | `SESS_2026_01_CASA`                          |
| Description      | *Première session pilote — Casablanca, démarrage Q1 2026.* |
| Visibilité       | Visible                                             |

### 14.2 Dupliquer les 5 cours modèles dans cette catégorie

Pour **chaque** module (1 à 5) :

#### Méthode A — Sauvegarde + Restauration (UI)

1. Ouvrir `[MODÈLE] Module N` → engrenage → *Sauvegarde*.
2. Décocher *Inclure les utilisateurs* et *Inclure les journaux*.
3. Lancer la sauvegarde → télécharger ou laisser dans le *Cours user backups*.
4. Engrenage → *Restaurer* → choisir le `.mbz` du modèle.
5. *Restaurer en tant que nouveau cours* → cible : catégorie
   `Session 2026-01 — Cohorte A — Casablanca`.
6. Lors de la restauration, **renommer** :
   - Nom complet → `Module N — <Thème>` (ex. `Module 1 — Fondamentaux`).
   - Nom court → `M1_2026_01_CASA` (et M2…M5 respectivement).
7. *Visibilité* du cours dupliqué : **Visible**.

#### Méthode B — CLI (gain de temps si plusieurs sessions)

```bash
# Backup
sudo -u www-data php /var/www/html/admin/cli/backup.php \
  --courseshortname=MOD_M1 \
  --destination=/tmp/m1.mbz

# Restore (note : pas de CLI native fournie dans le core pour restore + create
# course — utiliser un script PHP custom ou la fonction async via mod web).
```

> 💡 En l'absence de CLI restore stable dans le core, l'approche manuelle UI
> est recommandée pour la phase initiale (5 restorations × 1 session = ~30 min).
> Une **automatisation Sprint 2** est documentée plus loin (§ 14.6).

### 14.3 Créer la cohorte globale

`Administration du site > Utilisateurs > Comptes > Cohortes > Ajouter une nouvelle cohorte`

| Paramètre   | Valeur                                                  |
|-------------|---------------------------------------------------------|
| Nom         | Cohorte A — Casablanca — 2026-01                        |
| Identifiant | `COHORTE_2026_01_CASA`                                  |
| Contexte    | **Système** (cohorte globale)                           |
| Description | Apprenants de la première session pilote — Casablanca.  |
| Visible     | Oui                                                     |

### 14.4 Inscrire les apprenants à la cohorte

Soit par **import CSV** comme § 13.3 (la colonne `cohort1` doit valoir
`COHORTE_2026_01_CASA`), soit via :

`Administration du site > Utilisateurs > Comptes > Cohortes > Membres assignés` →
sélectionner les utilisateurs `ENS001`, `ENS002`, `ENS003`.

### 14.5 Synchroniser la cohorte aux 5 cours de la session

Pour **chaque** cours `Module 1` à `Module 5` de la session :

1. Ouvrir le cours → *Participants* → engrenage → *Méthodes d'inscription*.
2. *Ajouter méthode* → **Synchronisation des cohortes**.
3. Paramètres :
   - Cohorte → `Cohorte A — Casablanca — 2026-01`.
   - Rôle assigné → **Étudiant**.
   - Activé → Oui.
4. *Ajouter la méthode*.

> Effet : tous les membres actuels et futurs de la cohorte sont inscrits
> automatiquement comme `Étudiant` sur le cours.

### 14.6 Affecter les formateurs et le responsable pédagogique

#### Formateurs (rôle natif Teacher) — par cours

Pour chaque cours `Module 1`…`Module 5` de la session :

1. *Participants* → *Inscrire des utilisateurs*.
2. Rechercher `FOR001` (et/ou `FOR002`).
3. Rôle = *Enseignant* (Teacher) → *Inscrire*.

> Si un formateur intervient sur un sous-groupe de la cohorte uniquement,
> créer un **groupe** dans le cours (`Participants > Groupes`) et utiliser le
> mode de groupe *Groupes séparés* dans les activités concernées.

#### Responsable pédagogique — au niveau de la catégorie

1. Aller dans la catégorie `Session 2026-01 — Cohorte A — Casablanca`.
2. Engrenage → *Attribuer des rôles* (`Assigner des rôles`).
3. Choisir le rôle **Responsable pédagogique** → ajouter `RP001`.

> Effet : `RP001` voit tous les cours de cette session, peut consulter la
> progression, et — une fois le plugin custom déployé — accéder au tableau de
> bord de validation (§ 15).

### 14.7 Activer les restrictions d'accès Module N+1 dans les cours session

Si la duplication a bien été faite depuis les modèles (qui ont déjà les
restrictions configurées en § 12.3), les conditions de restriction sont
préservées **mais** elles pointent vers le cours **modèle** d'origine, pas vers
le cours session équivalent. Il faut **réajuster** :

Pour chaque cours `Module 2`…`Module 5` de la session :

1. Engrenage → *Modifier les paramètres* → *Restrictions d'accès*.
2. Modifier la condition *Achèvement de l'activité* :
   - Référence → cours `Module N` de la **même session** (ex. pour Module 2
     dans `SESS_2026_01_CASA`, référencer `M1_2026_01_CASA`).
3. Conserver la condition `validation_module_N = 1`.
4. *Enregistrer*.

### 14.8 Procédure récap (5–10 min par session après le 1er déploiement)

```
[ ] Créer catégorie session « Session YYYY-NN — Cohorte X — Ville »   (§ 14.1)
[ ] Dupliquer les 5 cours modèles dans la nouvelle catégorie          (§ 14.2)
[ ] Créer la cohorte globale « COHORTE_YYYY_NN_VILLE »                (§ 14.3)
[ ] Importer les apprenants (CSV) dans la cohorte                     (§ 14.4)
[ ] Synchroniser cohorte → 5 cours (rôle Étudiant)                    (§ 14.5)
[ ] Affecter les formateurs (cours par cours, rôle Teacher)           (§ 14.6)
[ ] Affecter le responsable pédago (catégorie, rôle responsable_pedago) (§ 14.6)
[ ] Réajuster les restrictions d'accès Module N+1                     (§ 14.7)
[ ] Test : se connecter en tant qu'apprenant → seul Module 1 visible
```

---

## 15. Phase L — Spécification du plugin `local_modulvalidation`

> **Statut V1 :** spec uniquement. Le développement effectif est planifié au
> Sprint 4. Cette section sert de cahier des charges au développeur.

### 15.1 Objectif

Outiller le **responsable pédagogique** pour qu'il puisse :

1. Voir la liste des apprenants de ses cohortes ayant achevé un module mais
   pas encore validé.
2. Valider le module pour un apprenant en un clic (met à jour
   `validation_module_N = 1` dans le profil).
3. Demander des corrections en saisissant un commentaire (envoyé par
   notification + email à l'apprenant et au formateur).
4. Filtrer par cohorte / module / statut.

### 15.2 Type de plugin

- **Type :** `local`
- **Nom court :** `modulvalidation`
- **Chemin :** `/var/www/html/local/modulvalidation/`
- **URL principale :** `/local/modulvalidation/index.php`
- **Référence officielle :** <https://moodledev.io/docs/apis/plugintypes/local>

### 15.3 Arborescence du plugin

```
local/modulvalidation/
├── version.php                       # Version + dépendances
├── index.php                         # Page principale (liste + actions)
├── settings.php                      # Réglages admin (optionnel)
├── lib.php                           # Hooks Moodle (extend_navigation_user)
├── locallib.php                      # Logique métier (helpers)
├── db/
│   ├── access.php                    # Capabilities du plugin
│   ├── install.xml                   # (optionnel) tables custom de log
│   └── upgrade.php                   # Scripts de mise à niveau BDD
├── classes/
│   ├── form/
│   │   └── validation_form.php       # moodleform de validation/correction
│   └── output/
│       └── renderer.php              # Rendu HTML
├── lang/
│   ├── en/local_modulvalidation.php
│   └── fr/local_modulvalidation.php
├── templates/
│   └── validation_list.mustache
└── README.md
```

### 15.4 Capabilities (db/access.php)

```php
$capabilities = [
    'local/modulvalidation:view' => [
        'captype'      => 'read',
        'contextlevel' => CONTEXT_COURSECAT,
        'archetypes'   => ['manager' => CAP_ALLOW],
    ],
    'local/modulvalidation:validate' => [
        'captype'      => 'write',
        'contextlevel' => CONTEXT_COURSECAT,
        'archetypes'   => ['manager' => CAP_ALLOW],
    ],
];
```

### 15.5 UX cible

- Lien d'accès dans le menu utilisateur du responsable pédagogique :
  *« Validation des modules »*.
- Page principale : tableau (DataTables-like) avec colonnes
  `Apprenant | Cohorte | Module | Achevé le | Statut | Actions`.
- Filtres : cohorte (multi-sélection), module (1–5), statut
  (`En attente | Validé | Corrections demandées`).
- Actions par ligne :
  - **Valider** → met à jour `user_info_data.data = '1'` pour le champ
    `validation_module_N` ; crée une entrée d'audit (datetime + validé par) ;
    envoie une notification + email à l'apprenant via l'API
    `\core\message\message`.
  - **Demander corrections** → ouvre une modale avec champ texte ;
    enregistre l'historique ; envoie une notification + email à l'apprenant
    et au formateur du cours.

### 15.6 Critères d'acceptation (pour le Sprint 4)

- Plugin installable via *Administration du site > Plugins > Installer des
  plugins* sans erreur de version.
- Les capabilities apparaissent et sont vraies pour le rôle
  `responsable_pedago` configuré § 8.
- La validation met à jour le champ profil (vérifiable dans
  `mdl_user_info_data`).
- L'apprenant voit le Module N+1 devenir accessible immédiatement après la
  validation (en respect des restrictions configurées § 12.3 / § 14.7).
- Une notification Moodle + un email sont reçus par l'apprenant.
- L'audit (qui a validé / quand) est consultable.

### 15.7 Tables BDD touchées

| Table                  | Lecture | Écriture | Notes                                            |
|------------------------|---------|----------|--------------------------------------------------|
| `mdl_user`             | Oui     | Non      | Liste des apprenants                             |
| `mdl_cohort`           | Oui     | Non      | Filtrage par cohorte                             |
| `mdl_cohort_members`   | Oui     | Non      |                                                  |
| `mdl_course`           | Oui     | Non      | Mapping module ↔ cours via `shortname`           |
| `mdl_course_completions` | Oui   | Non      | Détection de l'achèvement                        |
| `mdl_user_info_field`  | Oui     | Non      | Récupération de l'ID du champ `validation_module_N` |
| `mdl_user_info_data`   | Oui     | **Oui**  | Mise à jour du flag de validation                |
| `mdl_local_modulvalidation_log` (custom) | Oui | **Oui** | Audit trail (qui/quand/quoi)        |

### 15.8 Mocks d'écran (pour brief développeur)

```
┌──────────────────────────────────────────────────────────────────────┐
│ Validation des modules                                  [responsable]│
│                                                                      │
│ Cohorte: [▼ Toutes]   Module: [▼ Tous]   Statut: [▼ En attente]      │
├──────────────────────────────────────────────────────────────────────┤
│ Apprenant   │ Cohorte    │ Module │ Achevé le │ Statut       │ Actions
│ Benali A.   │ Casa A     │ M1     │ 04/05/26  │ En attente   │ [Valider] [Corrections]
│ Alaoui F.   │ Casa A     │ M1     │ 04/05/26  │ En attente   │ [Valider] [Corrections]
│ Idrissi K.  │ Casa A     │ M2     │ 02/05/26  │ Validé       │ —
└──────────────────────────────────────────────────────────────────────┘
```

---

## 16. Phase M — Installation BigBlueButton (VM dédiée)

> **Objectif :** déployer un serveur BigBlueButton 2.7+ sur une VM dédiée
> Ubuntu 22.04 LTS, le sécuriser en HTTPS, et brancher Moodle au plugin
> `mod_bigbluebuttonbn`.

### 16.1 Pré-requis serveur BBB

| Élément           | Valeur                                                       |
|-------------------|--------------------------------------------------------------|
| OS                | **Ubuntu Server 22.04 LTS** (impératif — pas 24.04)          |
| CPU               | 8 vCPU min                                                   |
| RAM               | 16 Go min                                                    |
| Disque            | 100 Go SSD min (prévoir 500 Go → 1 To si forte rétention enregistrements) |
| Réseau            | IP publique dédiée + 1 Gbps                                  |
| Domaine           | `bbb.<DOMAINE_PROD>` (DNS A → IP publique)                   |
| Ports ouverts     | `80/TCP`, `443/TCP`, `16384–32768/UDP` (WebRTC)              |
| Certificat SSL    | Let's Encrypt (généré par le script d'install)               |

### 16.2 Installation BBB

Sur la VM BBB, en tant que root (ou via sudo) :

```bash
sudo apt update && sudo apt -y upgrade

# Pré-requis
sudo apt -y install language-pack-en
sudo update-locale LANG=en_US.UTF-8

# Installation officielle scriptée (BBB 2.7)
wget -qO- https://ubuntu.bigbluebutton.org/bbb-install-2.7.sh | bash -s -- \
  -v focal-270 \
  -s bbb.<DOMAINE_PROD> \
  -e admin@<DOMAINE_PROD>
```

> Le script :
> - configure les dépôts BBB,
> - installe FreeSWITCH, Kurento, nginx, etcd,
> - obtient un certificat Let's Encrypt pour `bbb.<DOMAINE_PROD>`,
> - démarre tous les services.

### 16.3 Vérification BBB

```bash
sudo bbb-conf --check          # diagnostic complet
sudo bbb-conf --status         # statut des services
sudo bbb-conf --secret         # affiche URL + secret API
```

`bbb-conf --secret` retourne quelque chose comme :

```
URL: https://bbb.<DOMAINE_PROD>/bigbluebutton/
Secret: Xx2yH9kLp3mN4qR5sT6uV7wX8yZ9aB0cD1e...

Link to API-Mate:
https://mconf.github.io/api-mate/#server=...&sharedSecret=...
```

Tester en ouvrant le lien API-Mate → exécuter une requête `getMeetings` →
retour XML `<returncode>SUCCESS</returncode>`.

### 16.4 Branchement Moodle ↔ BBB

`Administration du site > Plugins > Modules d'activité > BigBlueButton`

| Paramètre                            | Valeur                                       |
|--------------------------------------|----------------------------------------------|
| URL du serveur BigBlueButton         | `https://bbb.<DOMAINE_PROD>/bigbluebutton/`  |
| Secret partagé                       | (valeur retournée par `bbb-conf --secret`)   |

Cliquer **Tester la connexion** → message vert *Connexion réussie*.

### 16.5 Paramètres par défaut des salles

`Administration du site > Plugins > Modules d'activité > BigBlueButton > Paramètres par défaut des salles`

| Paramètre                                  | Valeur recommandée |
|--------------------------------------------|--------------------|
| Salle d'attente                            | Activée            |
| Tous les utilisateurs rejoignent comme modérateur | Désactivé   |
| Activité enregistrable par défaut          | Activée            |
| Durée maximale de la session (minutes)     | 180                |
| Nombre max de participants                 | 50                 |

### 16.6 Test fonctionnel (à faire avant la fin de la Phase M)

1. Dans `[MODÈLE] Module 1`, ajouter une activité **BigBlueButton** de test.
2. Cliquer *Lancer la session*.
3. Vérifier que la salle s'ouvre dans un nouvel onglet sur `bbb.<DOMAINE_PROD>`.
4. Tester avec 2 navigateurs distincts (audio, vidéo, partage écran).
5. Activer l'enregistrement → terminer la session → attendre 15-30 min →
   vérifier que l'enregistrement apparaît dans le cours.

### 16.7 Stockage des enregistrements

| Paramètre                | Recommandation                       |
|--------------------------|---------------------------------------|
| Volume SSD dédié         | 500 Go (≈ 50 Mo/heure × 10 000 h)    |
| Rétention                | 12 mois après fin de session         |
| Suppression automatique  | Cron BBB natif (`recordings`)        |

---

## 17. Annexes

### Annexe A — Capabilities exhaustives des rôles custom

#### `responsable_pedago` (contexte = Catégorie de cours)

```
moodle/category:viewcourselist        Allow
moodle/course:view                    Allow
moodle/course:viewhiddencourses       Allow
moodle/course:viewparticipants        Allow
moodle/site:viewparticipants          Allow
gradereport/grader:view               Allow
gradereport/user:view                 Allow
report/completion:view                Allow
report/courseoverview:view            Allow
report/log:view                       Allow
mod/attendance:view                   Allow
mod/attendance:viewreports            Allow
moodle/user:viewdetails               Allow
moodle/user:viewalldetails            Allow
moodle/user:update                    Allow
local/modulvalidation:validate        Allow   (après install plugin custom)
local/modulvalidation:view            Allow   (après install plugin custom)
```

#### `jury_member` (contexte = Cours, exclusivement Module 5)

```
moodle/course:view                    Allow
moodle/course:viewparticipants        Allow
mod/assign:view                       Allow
mod/assign:viewblinddetails           Allow
mod/assign:grade                      Allow
mod/assign:reviewotherusers           Allow
gradereport/user:view                 Allow
moodle/user:viewdetails               Allow
mod/forum:viewdiscussion              Allow
mod/forum:replypost                   Allow
```

### Annexe B — Liste de vérification de fin de mise en place initiale

```
Installation
[ ] Moodle 4.5 LTS accessible en HTTPS sans erreur
[ ] Cron Moodle s'exécute toutes les minutes
[ ] Sessions Redis opérationnelles (vérifier mdl_sess_* dans Redis)
[ ] Mailpit / SMTP : envoi de test reçu

Configuration globale
[ ] Langue par défaut FR, fuseau Africa/Casablanca
[ ] Suivi d'achèvement + restrictions d'accès activés
[ ] Politique mots de passe conforme (10+ char, classes, verrouillage)
[ ] Self-registration désactivée
[ ] Politique de confidentialité publiée + acceptation obligatoire
[ ] Logs : rétention ≥ 18 mois

Plugins
[ ] BigBlueButtonBN, Attendance, Custom Certificate installés
[ ] Configurable Reports, Completion Progress installés
[ ] Scheduler, Checklist installés
[ ] Tous en vert dans « Vue d'ensemble des plugins »

Champs profil
[ ] 5 × validation_module_N (Checkbox, catégorie Progression)
[ ] certification_obtenue (Checkbox, catégorie Certification)
[ ] etablissement (Menu, catégorie Identité)  — liste à compléter
[ ] ville (Menu, catégorie Identité)         — liste à compléter

Rôles
[ ] responsable_pedago créé, contexte Catégorie, capabilities ✓
[ ] jury_member créé, contexte Cours, capabilities ✓

Catégories & cours
[ ] Catégorie racine « Formation Enseignants »
[ ] Sous-catégorie « Modèles » (cachée)
[ ] Sous-catégorie « Sessions »
[ ] 5 cours modèles créés (Module 1 → 5), tous cachés, format Sections
[ ] Achèvement de cours configuré sur chaque modèle
[ ] Restrictions d'accès Module N+1 configurées sur chaque modèle

Template SP
[ ] SP 1.1 — Démo créée dans Module 1 modèle (6 activités types)
[ ] Sauvegarde SP_TEMPLATE.mbz produite et archivée dans le repo
[ ] Restauration testée dans Module 2 (validation visuelle)

Comptes & session pilote
[ ] 10 comptes de test créés (cf. § 13.1)
[ ] Cohorte « COHORTE_2026_01_CASA » créée
[ ] Apprenants ENS001–003 inscrits dans la cohorte
[ ] Catégorie « Session 2026-01 — Cohorte A — Casablanca » créée
[ ] 5 cours session dupliqués
[ ] Synchronisation cohorte appliquée aux 5 cours session
[ ] Formateurs FOR001/2 affectés aux cours
[ ] Responsable RP001 affecté à la catégorie session
[ ] Restrictions d'accès des cours session réajustées (références internes)
[ ] Test apprenant : seul Module 1 visible avant validation

BBB
[ ] VM BBB provisionnée, OS Ubuntu 22.04 LTS
[ ] BBB 2.7+ installé, bbb-conf --check OK
[ ] DNS bbb.<DOMAINE_PROD> opérationnel + SSL valide
[ ] Plugin BBBN paramétré dans Moodle, test de connexion ✓
[ ] Test de classe virtuelle 2 utilisateurs : audio/vidéo/écran/enreg ✓

Documentation
[ ] Variables TBD du § 2 toutes renseignées
[ ] Procédure « créer une session » testée et documentée
[ ] Cahier des charges plugin local_modulvalidation transmis au dev
```

### Annexe C — Mapping métier ↔ Moodle (rappel)

| Métier                         | Moodle                                  |
|--------------------------------|-----------------------------------------|
| Parcours de formation          | Catégorie                               |
| Module                         | Cours                                   |
| SP (Situation Professionnelle) | Section / Topic                         |
| Cohorte / Promotion            | Cohort (système) + Group (par cours)    |
| Session de formation           | 1 catégorie + 1 cohorte + 5 cours       |
| Séance                         | Activité Attendance (séance)            |
| Classe virtuelle               | Activité BigBlueButton                  |
| Évaluation                     | Quiz (auto) ou Devoir (manuel)          |
| Certificat                     | Activité Custom Certificate             |

### Annexe D — Troubleshooting fréquent

| Symptôme                                           | Cause probable                                | Action                                                                 |
|----------------------------------------------------|-----------------------------------------------|------------------------------------------------------------------------|
| « Modèle 2 reste verrouillé » après validation     | Restrictions pointent vers cours modèles      | Réajuster les restrictions vers cours session (§ 14.7)                 |
| Apprenants pas inscrits aux cours après import     | Cohorte non synchronisée sur le cours         | Ajouter méthode *Synchronisation cohorte* sur le cours (§ 14.5)        |
| Tableau de bord vide                               | Bloc *Course Overview* non placé              | Ajouter le bloc dans `Pages par défaut > Tableau de bord`              |
| Champs `validation_module_N` invisibles            | Champs créés mais *Visible* = `Pas visible`   | Pour les responsables : la visibilité est gérée par le plugin (normal) |
| BBB : « Connexion échouée »                        | URL ou secret incorrect, ou pare-feu          | `bbb-conf --secret`, retester ; vérifier ports `16384-32768/UDP`       |
| Cron Moodle ne s'exécute pas                       | Crontab manquant côté serveur prod            | Vérifier `crontab -u www-data -l`                                      |
| Notifications email non reçues                     | SMTP mal configuré                            | Tester `Configuration messagerie sortante > Tester`                    |
| Quiz : score d'achèvement non comptabilisé         | *Note pour passer* non définie                | Définir la note pour passer dans le quiz **et** dans l'achèvement      |

### Annexe E — Références documentaires officielles

- Documentation administrateur Moodle : <https://docs.moodle.org/405/fr/>
- Documentation développeur Moodle : <https://moodledev.io>
- Plugin local — guide : <https://moodledev.io/docs/apis/plugintypes/local>
- Marketplace plugins : <https://moodle.org/plugins/>
- BigBlueButton — installation : <https://docs.bigbluebutton.org/2.7/install.html>
- Plugin BigBlueButtonBN : <https://moodle.org/plugins/mod_bigbluebuttonbn>
- Custom Certificate : <https://moodle.org/plugins/mod_customcert>
- Configurable Reports : <https://moodle.org/plugins/block_configurable_reports>

---

*Document généré dans le cadre du projet Plateforme Nationale de Formation des
Enseignants — V1, mise en place initiale. À mettre à jour à l'issue de la
recette interne (Sprint 2).*
