# Plateforme Nationale de Formation des Enseignants

> **Client :** Ministère / institution publique (Maroc)
> **Prestataire :** RFC Digital
> **Solution :** Moodle 4.5 LTS personnalisé
> **Statut :** En cours de développement — V1

Plateforme LMS institutionnelle pour la formation à grande échelle d'enseignants
issus d'établissements privés marocains. Le système gère des parcours hybrides
(présentiel + distanciel), un séquencement strict en 5 modules avec validation
inter-modules, et un dispositif de certification avec jury.

---

## Objectifs

- Inscrire et gérer plusieurs milliers d'enseignants-stagiaires
- Déployer des parcours hybrides (présentiel + distanciel)
- Verrouiller la progression module par module (validation explicite par le
  responsable pédagogique)
- Offrir un feedback individualisé
- Piloter la progression via des KPIs en temps réel
- Exporter les données pédagogiques (Excel, PDF, CSV)

## Stack technique cible (production)

| Composant | Version |
|---|---|
| OS | Ubuntu Server 22.04 LTS |
| LMS | **Moodle 4.5 LTS** |
| Web | Nginx 1.24+ |
| PHP | 8.2-FPM |
| BDD | MariaDB 10.11 LTS |
| Cache | Redis 7.x |
| Classes virtuelles | BigBlueButton 2.7+ (VM dédiée) |
| SSL | Let's Encrypt |

L'environnement de **preprod** tourne actuellement sur cPanel mutualisé
(CloudLinux + CageFS, `ea-php83`, MariaDB 10.11.16, sans Redis).

## Architecture pédagogique

```
Module 1 ──[validation]──►
Module 2 ──[validation]──►
Module 3 ──[validation]──►
Module 4 ──[validation]──►
Module 5 (Synthèse + Jury) ──► Certification PDF
```

Le module N+1 n'est accessible qu'après validation explicite du module N par
le responsable pédagogique (champ profil `validation_module_N`, alimenté par
un plugin custom `local_modulvalidation`).

Chaque module est découpé en **Situations Professionnelles (SP)** combinant
Page, Folder, URL/Vidéo, H5P, Forum, et une évaluation (Quiz ou Assignment).

## Plugins requis

BigBlueButtonBN, Attendance, Custom Certificate, H5P (natif), Configurable
Reports, Completion Progress, Scheduler, Group Choice, Checklist, PoodLL
(optionnel).

## Développement custom

- **`local_modulvalidation`** — interface de validation inter-modules pour le
  responsable pédagogique (formulaire, mise à jour champs profil,
  notifications email)
- **Thème** dérivé de Boost — charte client (logo, couleurs, accueil)
- **8 rapports SQL** via Configurable Reports (KPI institutionnels)

## Documentation projet

Toute la documentation projet se trouve dans [`docs/`](./docs/) :

| Fichier | Contenu |
|---|---|
| [`docs/RESUME_PROJET.md`](./docs/RESUME_PROJET.md) | Vue d'ensemble fonctionnelle, stack, sprints, livrables |
| [`docs/MOODLE_SETUP.md`](./docs/MOODLE_SETUP.md) | Procédure d'installation et de configuration Moodle |
| [`docs/PRODUCTION.md`](./docs/PRODUCTION.md) | Déploiement et exploitation en production |
| [`docs/DOCKER_ENV.md`](./docs/DOCKER_ENV.md) | Environnement Docker (dev local) |
| [`docs/suivi/`](./docs/suivi/) | Journaux de suivi datés (installation, incidents, décisions) |
| `docs/Plateforme_Nationale_Formation_Enseignants_V2.pdf` | Spécification fonctionnelle client |
| `docs/Plateforme_Enseignants_Document_Technique_Moodle.pdf` | Document technique de référence |

## Planning — 13 semaines de développement

| Sprint | Durée | Thème |
|---|---|---|
| Sprint 0 | 1 sem. | Préparation environnement (VMs, SSH, Git, backlog) |
| Sprint 1 | 2 sem. | Installation Moodle + structure de base |
| Sprint 2 | 2 sem. | Template SP + séquencement modules |
| Sprint 3 | 2 sem. | Hybridation (BBB, H5P, présence) |
| Sprint 4 | 2 sem. | Plugin validation + Module 5 |
| Sprint 5 | 2 sem. | Reporting & certification |
| Sprint 6 | 2 sem. | Recette, mise en production, formation |

## État courant

- ✅ Sprint 0 — environnement preprod provisionné, accès SSH, repo Git, clé SSH GitHub
- ✅ Sprint 1 — Moodle 4.5.10 installé, login web fonctionnel (extension `sodium`
  installée par l'hébergeur le 22/05/2026)
- 🔜 Sprint 2 — template SP + verrouillage Module N+1

Voir [`docs/suivi/`](./docs/suivi/) pour le détail des journaux d'installation
et d'incidents.

## Accès preprod (résumé)

| Élément | Valeur |
|---|---|
| URL | http://lms-moodle.preprod.io/moodle/ |
| Version | Moodle 4.5.10 |
| PHP | 8.3 (ea-php83 + FPM) |
| BDD | `lmsmoodle_moodle` (MariaDB 10.11.16) |
| moodledata | `/home/lmsmoodle/moodledata` |

Credentials admin et BDD : voir le coffre d'équipe — **jamais en clair dans
le dépôt**.

---

## À propos de Moodle

Ce dépôt embarque le code source de [Moodle](https://moodle.org), plateforme
LMS open source distribuée sous **GNU GPL v3**. La documentation officielle
Moodle reste disponible sur :

- [docs.moodle.org](https://docs.moodle.org/) — documentation utilisateur
- [moodledev.io](https://moodledev.io) — documentation développeur
- [moodle.org](https://moodle.org) — communauté

## Licence

- Code Moodle : **GNU GPL v3** (voir `COPYING.txt`)
- Développements custom (`local_modulvalidation`, thème, rapports SQL) :
  GPL v3 compatible, propriété intellectuelle conforme au contrat
  RFC Digital ↔ client.
