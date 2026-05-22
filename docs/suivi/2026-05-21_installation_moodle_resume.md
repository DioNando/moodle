# Suivi — Installation initiale Moodle (résumé)

> **Date :** 21 mai 2026 (mise à jour 22 mai 2026)
> **Environnement :** Serveur preprod cPanel (mutualisé, CloudLinux + CageFS)
> **URL :** http://lms-moodle.preprod.io/moodle/
> **Statut :** ✅ Moodle 4.5.10 installé et fonctionnel — extension `sodium` installée par l'hébergeur le 22/05/2026, login débloqué

---

## Étapes réalisées

1. **Préparation locale**
   - Téléchargement du ZIP Moodle 4.5 LTS
   - Initialisation du dépôt Git et push vers GitHub

2. **Mise en place serveur preprod**
   - Provisionnement du serveur preprod cPanel
   - Configuration de l'accès SSH pour Claude Code
   - Clone du dépôt GitHub dans `~/public_html`
   - Création du fichier `.env` (URL, credentials BDD, admin Moodle)

3. **Vérification environnement**
   - PHP disponibles : 5.6 → 8.5 (8.4 actif par défaut → trop récent)
   - MariaDB **10.11.16** (✅ correspond à la cible projet)

4. **Bascule PHP en 8.3**
   - `.htaccess` mis à jour : handler `ea-php83`
   - PHP Selector CloudLinux : extensions `sodium`, `opcache`, `intl`, `fileinfo`, `soap`, `nd_mysqli` activées

5. **Création de la base de données**
   - Via cPanel → MySQL Databases
   - Base : `lmsmoodle_moodle` / Utilisateur : `lmsmoodle_moodle` (ALL PRIVILEGES)

6. **Ajustement paramètres PHP** (via cPanel → Select PHP Version → Options)
   - `max_input_vars = 5000`
   - `memory_limit = 256M`
   - `post_max_size = 200M`
   - `upload_max_filesize = 200M`
   - `max_execution_time = 300`

7. **Création du moodledata**
   - `/home/lmsmoodle/moodledata` (hors `public_html`, permissions 750)

8. **Installation Moodle via CLI**
   - `admin/cli/install.php` (génération `config.php`)
   - `admin/cli/install_database.php` (création des tables)
   - Driver : `mariadb` (natif, pas `mysqli`)

---

## Accès

| Élément | Valeur |
|---|---|
| URL | http://lms-moodle.preprod.io/moodle/ |
| Admin | `admin` / `Admin@2026!` (⚠ à changer) |
| BDD | `lmsmoodle_moodle` (préfixe tables `mdl_`) |
| moodledata | `/home/lmsmoodle/moodledata` |
| PHP CLI | `/opt/alt/php83/usr/bin/php` |

---

---

## ✅ Point bloquant — Extension `sodium` (RÉSOLU)

Moodle 4.5 exige l'extension PHP `sodium` (cryptographie). Sur ce serveur cPanel mutualisé :

| Variante PHP | sodium dispo | FPM compatible | Verdict |
|---|---|---|---|
| `ea-php83` (EasyApache) | ✅ paquet installé le 22/05/2026 par l'hébergeur | ✅ (actif) | ✅ Site et login OK |
| `alt-php83` (CloudLinux Selector) | ✅ activable self-service | ❌ (FPM ne suit pas) | Non retenu (incompatible FPM) |

### Résolution (22 mai 2026)
- ✅ L'hébergeur a installé le paquet **`ea-php83-php-sodium`** côté serveur
- ✅ Login admin web fonctionnel — plus d'erreur `SODIUM_CRYPTO_SECRETBOX_NONCEBYTES`
- ✅ UI Moodle accessible — Sprints 1+ débloqués
- ✅ Stack conservée : `ea-php83` + PHP-FPM (perf optimales)

### Historique du blocage (pour mémoire)
- ❌ Login web impossible — erreur `Undefined constant core\SODIUM_CRYPTO_SECRETBOX_NONCEBYTES`
- ❌ Toute configuration via l'UI Moodle était bloquée (plugins, profils, rôles, catégories)
- ✅ Outils CLI Moodle (`admin/cli/*`) restaient utilisables pendant l'attente

---

## ℹ️ Point d'environnement — Redis indisponible

Vérification effectuée le 21/05/2026 : **aucun serveur Redis disponible** sur ce serveur cPanel mutualisé.

| Composant | État |
|---|---|
| Démon `redis-server` | ❌ pas de process sur port 6379 |
| Binaire `redis-cli` | ❌ non installé |
| Extension PHP `redis.so` (client) | ⚠️ activable via PHP Selector mais inutile sans serveur |

### Impact
- Moodle utilisera son cache **fichier par défaut** (`moodledata/cache/`) — fonctionnel mais ≈ ×3 moins rapide qu'avec Redis
- Suffisant pour la preprod (tests, validation fonctionnelle)
- ⚠️ **Non viable pour la production** (500-1000 users prévus)

### Décision
- Preprod : on reste sur cache fichier
- Prod (VM dédiée Ubuntu 22.04) : Redis 7 sera installé nativement comme prévu dans `RESUME_PROJET.md`

---

## Configuration Git / GitHub (22 mai 2026)

Mise en place d'une **clé SSH GitHub** pour le compte du serveur preprod afin de simplifier les `git pull`/`push` (plus de saisie de token HTTPS) :

```bash
ssh-keygen -t rsa -b 4096 -C "your_email@example.com"
# clé générée dans ~/.ssh/id_rsa (+ id_rsa.pub)
```

- Clé publique (`~/.ssh/id_rsa.pub`) copiée dans **GitHub → Settings → SSH and GPG keys → New SSH key**
- Permet d'utiliser l'URL `git@github.com:...` pour le dépôt (au lieu de HTTPS)
- Authentification serveur ↔ GitHub désormais transparente

## Prochaines étapes

- [x] ~~**[BLOQUANT]** Obtenir résolution sodium côté hébergeur~~ ✅ Résolu 22/05/2026
- [x] ~~Configurer l'accès SSH GitHub pour le serveur preprod~~ ✅ Clé SSH ajoutée 22/05/2026
- [ ] Changer le mot de passe admin au premier login
- [ ] Configurer le cron Moodle (cPanel → Cron Jobs, chaque minute)
- [ ] Activer HTTPS (Let's Encrypt cPanel) + mise à jour `$CFG->wwwroot`
- [ ] Installer les plugins requis (Sprint 3 — BBB, Attendance, Customcert, H5P, etc.)
- [ ] Créer les champs profil `validation_module_1..5` et rôles custom
- [ ] Mettre en place l'arborescence de catégories cible
