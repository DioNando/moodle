# Suivi — Installation initiale Moodle (détaillé)

> **Date :** 21 mai 2026 (mise à jour 22 mai 2026)
> **Environnement :** Serveur preprod cPanel (CloudLinux + CageFS)
> **URL cible :** http://lms-moodle.preprod.io/moodle/
> **Statut final :** ✅ Moodle 4.5.10 installé et fonctionnel — extension `sodium` installée par l'hébergeur le 22/05/2026 (cf. § 11), login web débloqué. Clé SSH GitHub configurée pour le serveur (cf. § 13).

---

## 1. Préparation locale

### 1.1 Récupération du code Moodle
- Téléchargement de l'archive officielle Moodle **4.5 LTS** (`moodle-4.5.10.zip`)
- Extraction dans le dossier projet
- Conservation du ZIP et du dossier `moodle-4.5.10/` comme référence figée

### 1.2 Versioning Git
- Initialisation du dépôt local
- Push initial vers GitHub (code Moodle + dossier `docs/`)
- Le dépôt sert de source de vérité pour les déploiements ultérieurs

---

## 2. Mise en place du serveur preprod

### 2.1 Provisionnement
- Création d'un compte cPanel sur le serveur de préproduction
- Domaine attribué : `lms-moodle.preprod.io`
- Environnement détecté :
  - OS : Linux 5.14.0 (RHEL 9)
  - CloudLinux + CageFS (isolation utilisateur)
  - Web : Nginx
  - Versions PHP disponibles via EasyApache : 5.6 → 8.5
  - MariaDB 10.11.16 (✅ identique à la cible projet)

### 2.2 Accès SSH pour Claude
- Configuration de la clé SSH dans cPanel → SSH Access
- Activation de l'accès shell pour le compte `lmsmoodle`
- Test : connexion SSH fonctionnelle, accès `~/public_html`

### 2.3 Récupération du code sur le serveur
```bash
cd ~/public_html
git clone <repo_github> .
```

### 2.4 Configuration `.env`
Copie de `.env.example` vers `.env` puis remplissage avec les valeurs preprod :
```env
MOODLE_WWWROOT=http://lms-moodle.preprod.io
MOODLE_SITE_NAME=Plateforme Nationale Formation Enseignants
MOODLE_ADMIN_USER=admin
MOODLE_ADMIN_PASSWORD=Admin@2026!
MOODLE_ADMIN_EMAIL=admin@formation.ma
DB_NAME=lmsmoodle_moodle
DB_USER=lmsmoodle_moodle
DB_PASSWORD=<mot_de_passe_fort>
```

---

## 3. Vérification de l'environnement

### 3.1 Versions détectées
```bash
$ php -v
PHP 8.4.x  ❌ (trop récent — Moodle 4.5 supporte 8.1 → 8.3)

$ mysql --version
mysql Ver 15.1 Distrib 10.11.16-MariaDB  ✅
```

### 3.2 Conclusion
- **MariaDB OK** — correspond à la stack cible
- **PHP à basculer en 8.3** — version max supportée par Moodle 4.5

---

## 4. Bascule PHP vers la version 8.3

### 4.1 Modification du handler Apache via `.htaccess`
**Avant :**
```apache
AddHandler application/x-httpd-ea-php84 .php .php8 .phtml
```
**Après :**
```apache
AddHandler application/x-httpd-ea-php83 .php .php8 .phtml
```

### 4.2 Audit des extensions PHP requises par Moodle
Test initial sur `ea-php83` :

| Extension | Statut initial | Action |
|---|---|---|
| iconv, mbstring, curl, soap, ctype, zip, gd | ✅ | — |
| simplexml, dom, xml, xmlreader, intl, json, hash | ✅ | — |
| fileinfo, pdo, exif, openssl, mysqli, xsl | ✅ | — |
| **sodium** | ❌ | À activer |
| **opcache** | ❌ | À activer |
| xmlrpc | ❌ | Optionnel (déprécié) |
| redis | ❌ | Non disponible mutualisé |

### 4.3 Bascule vers CloudLinux PHP Selector (alt-php)
- `ea-php83` (EasyApache) ne contient pas `sodium.so`
- `alt-php83` (CloudLinux PHP Selector) propose `sodium`, `opcache`, `redis`, `xmlrpc`
- → Utilisation de **alt-php83** pour bénéficier du sélecteur d'extensions

### 4.4 Activation des extensions
Via cPanel → **Select PHP Version** → Extensions :
- ✅ `sodium` (requis Moodle 4.5)
- ✅ `opcache` (performance)
- ✅ `intl`
- ✅ `fileinfo`
- ✅ `soap`
- ✅ `nd_mysqli` (équivalent CloudLinux de `mysqli`)

Commandes équivalentes (côté serveur) :
```bash
selectorctl --enable-user-extensions=sodium,opcache,intl,fileinfo,soap --user=lmsmoodle --version=8.3
```

Vérification :
```bash
/opt/alt/php83/usr/bin/php -r "
foreach (['sodium','opcache','intl','fileinfo','soap','nd_mysqli'] as \$e)
  echo \$e.': '.(extension_loaded(\$e)?'OK':'KO').PHP_EOL;
"
# → toutes OK
```

---

## 5. Création de la base de données

### 5.1 Via cPanel → MySQL® Databases
1. **Create New Database** : `moodle` → devient `lmsmoodle_moodle` (préfixe compte)
2. **Add New User** : `moodle` → devient `lmsmoodle_moodle`, mot de passe fort
3. **Add User To Database** : ALL PRIVILEGES

### 5.2 Test de connexion
```bash
$ mysql -u 'lmsmoodle_moodle' -p'<mdp>' -e "SHOW DATABASES;"
Database
information_schema
lmsmoodle_moodle
```
✅ Connexion OK

---

## 6. Ajustement des paramètres PHP

### 6.1 Valeurs cibles (recommandations Moodle 4.5)
Via cPanel → **Select PHP Version** → **Options** :

| Paramètre | Avant | Après | Motif |
|---|---|---|---|
| `max_input_vars` | 1000 | **5000** | Exigence Moodle (gros formulaires) |
| `memory_limit` | 128M | **256M** | Marge confortable installation + plugins |
| `post_max_size` | 8M | **200M** | Upload vidéos/PDF cours |
| `upload_max_filesize` | 2M | **200M** | Idem |
| `max_execution_time` | 30 | **300** | Sauvegardes/restaurations longues |

### 6.2 Note sur le CLI
Les options du PHP Selector ne s'appliquent **pas** au CLI. Pour l'installation, les valeurs sont passées directement via `-d` :
```bash
/opt/alt/php83/usr/bin/php \
  -d max_input_vars=5000 \
  -d memory_limit=512M \
  -d post_max_size=200M \
  -d upload_max_filesize=200M \
  -d max_execution_time=600 \
  admin/cli/install_database.php ...
```

---

## 7. Création du `moodledata`

```bash
mkdir -p /home/lmsmoodle/moodledata
chmod 750 /home/lmsmoodle/moodledata
```

> ⚠ **Hors `public_html`** : impératif de sécurité. Le `moodledata` contient les fichiers uploadés, sessions, cache — il ne doit jamais être servi par le web.

---

## 8. Installation Moodle via CLI

### 8.1 Génération du `config.php`
```bash
/opt/alt/php83/usr/bin/php /home/lmsmoodle/public_html/moodle/admin/cli/install.php \
  --lang=fr \
  --wwwroot='http://lms-moodle.preprod.io/moodle' \
  --dataroot=/home/lmsmoodle/moodledata \
  --dbtype=mariadb \
  --dbhost=localhost \
  --dbname='lmsmoodle_moodle' \
  --dbuser='lmsmoodle_moodle' \
  --dbpass='<mdp>' \
  --prefix=mdl_ \
  --fullname='Plateforme Nationale Formation Enseignants' \
  --shortname='PNFE' \
  --adminuser='admin' \
  --adminpass='Admin@2026!' \
  --adminemail='admin@formation.ma' \
  --non-interactive --agree-license
```

> ⚠ Important : `--dbtype=mariadb` (driver natif), **pas** `mysqli`.

### 8.2 Installation des tables
```bash
/opt/alt/php83/usr/bin/php \
  -d max_input_vars=5000 -d memory_limit=512M \
  -d post_max_size=200M -d upload_max_filesize=200M \
  -d max_execution_time=600 \
  /home/lmsmoodle/public_html/moodle/admin/cli/install_database.php \
  --lang=fr \
  --adminuser='admin' --adminpass='Admin@2026!' \
  --adminemail='admin@formation.ma' \
  --fullname='Plateforme Nationale Formation Enseignants' \
  --shortname='PNFE' \
  --agree-license
```

Résultat :
```
...
-->factor_webauthn
++ Succès (0,03 secondes) ++
-->upgrade_noncore()
++ Succès (0,48 secondes) ++
Installation terminée avec succès.
```

### 8.3 Vérification web
```bash
$ curl -sI http://lms-moodle.preprod.io/moodle/
HTTP/1.1 200 OK
Server: nginx
Content-Type: text/html; charset=utf-8
Content-Language: fr
```
✅ Page de connexion accessible.

---

## 9. Récapitulatif des accès

| Élément | Valeur |
|---|---|
| URL | http://lms-moodle.preprod.io/moodle/ |
| Identifiant admin | `admin` |
| Mot de passe admin initial | `Admin@2026!` (⚠ à changer) |
| Email admin | `admin@formation.ma` |
| Base de données | `lmsmoodle_moodle` |
| Utilisateur BDD | `lmsmoodle_moodle` |
| Préfixe tables | `mdl_` |
| moodledata | `/home/lmsmoodle/moodledata` |
| PHP CLI | `/opt/alt/php83/usr/bin/php` |
| PHP version | 8.3.31 (alt-php CloudLinux) |
| MariaDB | 10.11.16 |
| Version Moodle | 4.5.10 (Build 20260216) |

---

## 10. Points d'attention pour la suite

### Différences avec la stack cible production
| Aspect | Preprod cPanel (actuel) | Cible production |
|---|---|---|
| OS | CloudLinux mutualisé | Ubuntu 22.04 dédié |
| Web | Nginx managé cPanel | Nginx natif (config maîtrisée) |
| PHP | alt-php 8.3 (Selector) | PHP 8.2-FPM natif |
| Cache | Aucun (pas de Redis) | Redis 7.x |
| SSL | À activer (Let's Encrypt cPanel) | Let's Encrypt + certbot |
| BBB | Non installé | VM dédiée 8 vCPU/16 Go |
| MFA | Non configuré | `tool_mfa` obligatoire |

### Prochaines étapes (Sprint 1 — suite)
1. **Sécurité immédiate**
   - [ ] Changer le mot de passe admin au premier login
   - [ ] Vérifier que `config.php` est en 640 / www-data
2. **Cron Moodle**
   - [ ] cPanel → Cron Jobs → toutes les minutes :
     ```
     /opt/alt/php83/usr/bin/php /home/lmsmoodle/public_html/moodle/admin/cli/cron.php >/dev/null 2>&1
     ```
3. **HTTPS**
   - [ ] Activer Let's Encrypt via cPanel → SSL/TLS Status
   - [ ] Modifier `$CFG->wwwroot` en `https://...`
4. **Configuration globale Moodle** (Phase B du `MOODLE_SETUP.md`)
   - [ ] Pays par défaut : MA — Fuseau : Africa/Casablanca — Langue : fr
   - [ ] Politique de mot de passe renforcée
   - [ ] Désactiver l'auto-inscription publique
5. **Plugins requis** (Phase C)
   - [ ] BigBlueButtonBN, Attendance, Customcert, Configurable Reports, Completion Progress, Scheduler, Checklist
6. **Champs profil & rôles** (Phases D & E)
   - [ ] `validation_module_1..5`, `certification_obtenue`, `etablissement`, `ville`
   - [ ] Rôles `responsable_pedago`, `jury_member`
7. **Arborescence catégories** (Phase F) — `Formation Enseignants > Modèles | Sessions`

---

---

## 11. ✅ Incident post-installation — Extension `sodium` (RÉSOLU le 22/05/2026)

> **Mise à jour 22 mai 2026 :** L'hébergeur a installé le paquet `ea-php83-php-sodium` côté serveur (option 1 du ticket, cf. § 11.4). L'extension `sodium` est désormais chargée sur `ea-php83`, le login admin web fonctionne, l'UI Moodle est entièrement accessible. Les sections ci-dessous sont conservées pour mémoire de l'incident.

---


### 11.1 Symptôme
Au premier login admin sur `http://lms-moodle.preprod.io/moodle/login/index.php` :

```
Exception : Undefined constant "core\SODIUM_CRYPTO_SECRETBOX_NONCEBYTES"
```

Moodle 4.5 utilise l'extension PHP **`sodium`** (libsodium) pour la cryptographie (chiffrement de données sensibles, tokens, signatures). Sans cette extension, le login et de nombreuses fonctions internes échouent.

### 11.2 Diagnostic
Le serveur de preprod expose **deux familles PHP** :

| PHP | Origine | `sodium` | SAPI web | Contrôle utilisateur |
|---|---|---|---|---|
| `ea-php83` | cPanel EasyApache | ❌ paquet `ea-php83-php-sodium` non installé côté serveur | PHP-FPM (déjà actif) | Aucun (root WHM uniquement) |
| `alt-php83` | CloudLinux PHP Selector | ✅ activable self-service via cPanel | mod_lsapi | Self-service ✅ |

Tentative de bascule sur `alt-php83` (via cPanel "Select PHP Version") :
- ✅ CLI fonctionne : `/opt/alt/php83/usr/bin/php` → `sodium = OK`
- ❌ Web : retourne **HTTP 503**
  - Cause : PHP-FPM est configuré au niveau Apache/Nginx pour `ea-php83` ; en basculant le handler vers `alt-php83`, le socket FPM ne correspond plus
  - `alt-php` n'utilise pas FPM mais `mod_lsapi` → il faut désactiver FPM pour ce domaine
  - cPanel → **MultiPHP Manager** refuse la désactivation de FPM (verrouillé administrativement)
  - Les fichiers de config `/var/cpanel/userdata/lmsmoodle/lms-moodle.preprod.io.php-fpm.cache` sont en root-only → non modifiables sans accès admin

### 11.3 Décision temporaire
Retour à **`ea-php83`** dans le PHP Selector pour restaurer l'accès au site (HTTP 200). Le login reste bloqué tant que sodium n'est pas dispo, mais cela permet :
- d'accéder à la page de connexion (vérification visuelle de l'install)
- d'utiliser les outils CLI Moodle pour la suite (`admin/cli/*`)
- d'éviter le 503

### 11.4 Action de déblocage — Ticket hébergeur
À adresser au fournisseur du serveur preprod (RFC Digital ou hébergeur sous-jacent) :

> **Objet :** Installation extension PHP sodium pour ea-php83 / domaine `lms-moodle.preprod.io`
>
> Bonjour,
> Pour faire fonctionner Moodle 4.5 LTS, j'ai besoin que l'extension PHP **`sodium`** soit disponible sur le PHP servant mon domaine `lms-moodle.preprod.io`.
>
> Deux options possibles (par ordre de préférence) :
> 1. **Installer le paquet `ea-php83-php-sodium`** au niveau serveur (`yum install ea-php83-php-sodium`)
> 2. **OU désactiver PHP-FPM** pour mon domaine dans MultiPHP Manager, afin que je puisse utiliser `alt-php83` (CloudLinux PHP Selector) où sodium est déjà disponible
>
> L'option 1 est préférable car elle conserve les performances FPM.
> Merci !

### 11.5 Impact sur le planning
- ✅ Sprint 0 (préparation env) : terminé
- ✅ Sprint 1 (installation + config globale) : install OK, **UI débloquée le 22/05/2026**
- ✅ Sprints 2+ : peuvent démarrer

**Mitigation appliquée pendant l'attente (21–22 mai)** :
- Préparation de la documentation des sprints suivants (plugins, structure pédagogique, rôles, champs profil)
- Utilisation des CLI Moodle (`admin/cli/install_plugins.php`, `admin/cli/upgrade.php`, etc.) pour ce qui est scriptable
- Préparation du template SP en local pour import ultérieur

### 11.6 Résolution effective (22 mai 2026)
- Ticket hébergeur traité — option 1 retenue : installation du paquet `ea-php83-php-sodium` côté serveur
- Vérification post-installation :
```bash
/opt/cpanel/ea-php83/root/usr/bin/php -m | grep -i sodium
# → sodium
```
- Login admin web : ✅ OK (plus d'exception `SODIUM_CRYPTO_SECRETBOX_NONCEBYTES`)
- Aucun changement de configuration côté Moodle nécessaire — l'install reposait déjà sur `ea-php83` + FPM
- Stack conservée : performances FPM préservées

---

---

## 12. ℹ️ Vérification environnement — Redis

### 12.1 Contexte
Le `RESUME_PROJET.md` prévoit **Redis 7.x** dans la stack cible (cache MUC + sessions, gain ≈ ×3 perf). Vérification de la disponibilité sur le serveur preprod.

### 12.2 Diagnostic effectué
```bash
# Binaire client
which redis-cli                  # → introuvable

# Démon en écoute
ss -tln | grep :6379             # → aucun résultat
ls /var/run/redis* /tmp/redis*   # → aucun socket

# Extension PHP côté CLI
/opt/alt/php83/usr/bin/php -m | grep -i redis     # → vide
/opt/cpanel/ea-php83/root/usr/bin/php -m | grep -i redis  # → vide

# État dans le PHP Selector
selectorctl --list-user-extensions --user=lmsmoodle --version=8.3 --all | grep redis
# → "- redis" (extension cliente disponible mais désactivée)
```

### 12.3 Conclusion
| Composant | État | Note |
|---|---|---|
| Serveur Redis | ❌ Non installé | Typique mutualisé cPanel |
| Extension PHP cliente `redis.so` | ⚠️ Disponible non activée | Inutile sans serveur |

### 12.4 Décision et mitigation
- **Preprod (actuelle)** : utilisation du cache **fichier** Moodle par défaut (`moodledata/cache/`). Fonctionnel, ≈ ×3 plus lent qu'avec Redis selon les benchmarks Moodle officiels. Acceptable pour une preprod de validation (volume utilisateurs réduit).
- **Production (Ubuntu 22.04 dédié à venir)** : Redis 7 installé nativement (`apt install redis-server php8.2-redis`), configuration MUC + sessions conforme au plan initial.

### 12.5 Optionnel — Demande complémentaire à l'hébergeur
Si on veut tester les performances cible dès la preprod, on peut ajouter au ticket sodium :

> Serait-il possible de mettre à disposition un service Redis (ou Memcached à défaut) sur le serveur preprod ?
> Notre application en bénéficierait pour la mise en cache (gain ×3 sur les performances).

Non bloquant — peut être différé.

---

## 13. 🔑 Configuration de la clé SSH GitHub (22 mai 2026)

### 13.1 Contexte
Le dépôt projet est hébergé sur GitHub. Le serveur preprod doit pouvoir effectuer des `git pull`/`git fetch` (et éventuellement `push`) de manière non interactive — sans saisie de PAT HTTPS à chaque opération. Mise en place d'une **authentification par clé SSH**.

### 13.2 Génération de la clé sur le serveur
Connecté en SSH sur le compte `lmsmoodle`, exécution dans le terminal :

```bash
ssh-keygen -t rsa -b 4096 -C "your_email@example.com"
```

- Algorithme : RSA 4096 bits
- Emplacement par défaut : `~/.ssh/id_rsa` (clé privée) + `~/.ssh/id_rsa.pub` (clé publique)
- Passphrase : (laissée vide pour permettre l'usage non interactif)

### 13.3 Ajout de la clé publique sur GitHub
1. Affichage de la clé publique :
   ```bash
   cat ~/.ssh/id_rsa.pub
   ```
2. Copie du contenu intégral (ligne `ssh-rsa AAAA... your_email@example.com`)
3. GitHub → **Settings** → **SSH and GPG keys** → **New SSH key**
   - Title : `preprod lms-moodle` (ou équivalent identifiant la machine)
   - Key type : `Authentication Key`
   - Key : collage du contenu de `id_rsa.pub`
4. Validation (Add SSH key)

### 13.4 Vérification de la connexion
```bash
ssh -T git@github.com
# → Hi <user>! You've successfully authenticated, but GitHub does not provide shell access.
```

### 13.5 Bascule (optionnelle) du remote vers SSH
Si le dépôt local a été cloné en HTTPS, on peut basculer son remote :

```bash
cd ~/public_html
git remote -v                                    # vérifier l'URL actuelle
git remote set-url origin git@github.com:<org>/<repo>.git
git remote -v                                    # confirmer la bascule
```

### 13.6 Bénéfices
- ✅ `git pull` / `git fetch` / `git push` non interactifs (plus de prompt PAT)
- ✅ Automatisable dans un cron de déploiement le cas échéant
- ✅ Clé révocable côté GitHub à tout moment (Settings → SSH keys → Delete)

### 13.7 Hygiène sécurité
- La clé privée `~/.ssh/id_rsa` reste **sur le serveur preprod uniquement** — jamais copiée ailleurs
- Permissions vérifiées : `~/.ssh` en `700`, `id_rsa` en `600` (défaut `ssh-keygen`)
- En cas de compromission : supprimer la clé depuis GitHub puis régénérer

---

*Document rédigé le 21 mai 2026 — Suivi installation Sprint 0/1*
*Mise à jour 21 mai 2026 — Ajout § 11 (incident sodium) + § 12 (vérif Redis)*
*Mise à jour 22 mai 2026 — § 11 marqué RÉSOLU (sodium installé par l'hébergeur) + ajout § 13 (clé SSH GitHub)*
