# Plateforme Nationale de Formation des Enseignants — Résumé du Projet

> **Prestataire :** RFC Digital  
> **Version :** 1.0 — Avril 2026  
> **Statut :** En cours de développement (V1)  
> **Confidentialité :** Document à usage interne

---

## 1. Contexte & Objectifs

### Problématique client
Formation à grande échelle d'enseignants issus d'établissements privés marocains, avec des profils numériques hétérogènes répartis géographiquement. Le client a besoin d'un outil institutionnel capable de :

- Inscrire et gérer des milliers d'utilisateurs
- Déployer des parcours hybrides (présentiel + distanciel)
- Offrir un feedback individualisé à chaque enseignant-stagiaire
- Piloter la progression via des KPIs en temps réel
- Exporter des données pédagogiques (Excel, PDF)
- Être autonome et évolutif dans le temps (V2, V3)

### Solution retenue
**Moodle LTS sur mesure**, clé en main, avec personnalisation complète, intégration BigBlueButton et développement de plugins custom.

---

## 2. Architecture Fonctionnelle

### Parcours pédagogique
Le programme est structuré en **5 modules thématiques séquentiels** :

```
Module 1 ──[validation]──►
Module 2 ──[validation]──►
Module 3 ──[validation]──►
Module 4 ──[validation]──►
Module 5 (Synthèse + Jury) ──► Certification PDF
```

**Règle clé :** Le module N+1 n'est accessible qu'après validation **explicite** du module N par le responsable pédagogique (via un champ profil utilisateur `validation_module_N` activé par un plugin custom).

### Structure d'un Module (SP — Situation Professionnelle)
Chaque module est découpé en plusieurs SP. Une SP contient :

| # | Type Moodle | Contenu |
|---|-------------|---------|
| 1 | Page | Introduction & Objectifs (contexte, compétences, volume horaire) |
| 2 | Dossier (Folder) | Supports documentaires (PDF, DOCX, présentations) |
| 3 | URL / Étiquette | Vidéos pédagogiques (YouTube non-listé, Vimeo) |
| 4 | H5P | Contenus interactifs (Interactive Video, Branching Scenario…) |
| 5 | Forum | Échanges & questions (optionnel) |
| 6 | Quiz **ou** Assignment | Évaluation unique de la SP |

### Module 5 — Spécificité
Module terminal combinant parcours en ligne et soutenance en présentiel devant un jury :
- Dépôt du mémoire de synthèse (Assignment PDF)
- Planification des créneaux de soutenance (plugin Scheduler)
- Espace jury réservé (rôle custom `jury_member`)
- Génération automatique du certificat après délibération favorable

### Profils Utilisateurs & Rôles

| Rôle métier | Rôle Moodle | Niveau | Principales permissions |
|---|---|---|---|
| Apprenant / Enseignant-stagiaire | Student (natif) | Cours via cohorte | Consulter, soumettre, passer quiz |
| Formateur / Tuteur | Teacher (natif) | Cours par groupe | Corriger, feedback, animer BBB |
| Responsable pédagogique | **Custom** `responsable_pedago` | Catégorie session | Valider modules, dashboard de validation |
| Membre du jury | **Custom** `jury_member` | Cours Module 5 uniquement | Consulter mémoires, grille jury, délibération |
| Administrateur fonctionnel | Manager (natif) | Catégorie racine | Créer sessions, gérer cohortes |
| Administrateur technique | Site Administrator (natif) | Global | Accès complet système |

---

## 3. Stack Technique

### Composants Serveur

| Composant | Version recommandée | Justification |
|---|---|---|
| **Système d'exploitation** | Ubuntu Server 22.04 LTS | Support jusqu'en 2027, compatibilité Moodle |
| **LMS** | **Moodle 4.5 LTS** | Support général déc. 2027, sécurité déc. 2028 |
| **Serveur web** | Nginx 1.24+ | Performances élevées, meilleure gestion concurrence |
| **Langage serveur** | **PHP 8.2 FPM** | Version officielle recommandée Moodle 4.5 |
| **Base de données** | **MariaDB 10.11 LTS** | Compatible Moodle, maintenu jusqu'en 2028 |
| **Cache mémoire** | **Redis 7.x** | Cache applicatif Moodle (MUC), sessions, ×3 perf |
| **Classes virtuelles** | **BigBlueButton 2.7+** | Serveur dédié (VM séparée), intégration via plugin BBBN |
| **Stockage fichiers** | SSD local + backup S3-compatible | Moodledata SSD, snapshots stockage objet |
| **SSL / Reverse proxy** | Nginx + Let's Encrypt | Certificats gratuits, renouvellement automatique |

### Configuration Serveur Minimale (500–1000 utilisateurs, ~100 sessions concurrentes)

| Serveur | CPU | RAM | Stockage | Réseau |
|---|---|---|---|---|
| Moodle (app) | 8 vCPU | 16 Go | 200 Go SSD | — |
| Base de données | 4 vCPU | 16 Go | 100 Go SSD | — |
| BigBlueButton | 8 vCPU | 16 Go | 100 Go SSD | **1 Gbps** |
| Backups | — | — | S3-compatible, rétention 30j | — |

> **Hébergement recommandé :** OVH (Paris/Gravelines), Scaleway, ou Hetzner — proximité géographique Maroc (latence < 50 ms), conformité CNDP facilitée par localisation européenne des données.

### Plugins Moodle Requis

| Plugin | Identifiant technique | Type | Usage |
|---|---|---|---|
| BigBlueButtonBN | `mod_bigbluebuttonbn` | Activity | Classes virtuelles synchrones |
| Attendance | `mod_attendance` | Activity | Gestion présence présentiel & distanciel |
| Custom Certificate | `mod_customcert` | Activity | Génération certificats PDF avec QR code |
| H5P Content Bank | `contentbank_h5p` (natif 4.5) | Natif | Contenus interactifs (vidéo annotée, quiz…) |
| Configurable Reports | `block_configurable_reports` | Block | Rapports SQL personnalisés, exports Excel/PDF |
| Completion Progress | `block_completion_progress` | Block | Barre de progression visuelle apprenant |
| Scheduler | `mod_scheduler` | Activity | Planification créneaux de soutenance (Module 5) |
| Group Choice | `mod_choicegroup` | Activity | Auto-inscription aux groupes (optionnel) |
| Checklist | `mod_checklist` | Activity | Listes de contrôle pédagogiques tuteur |
| PoodLL / AudioRec | `filter_poodll` | Filter | Feedback audio/vidéo sur devoirs (optionnel) |

### Développement Custom

| Livrable | Type | Description |
|---|---|---|
| `local_modulvalidation` | Plugin local PHP | Interface de validation inter-modules pour le responsable pédagogique — formulaire Moodle, mise à jour champs profil, notifications email |
| Thème personnalisé | Boost custom | Charte graphique client (logo, couleurs, page d'accueil) |
| Rapports SQL | Configurable Reports | 8 rapports KPI institutionnels |

---

## 4. Architecture Moodle — Structuration des Catégories

```
Formation Enseignants (catégorie racine)
│
├── Modèles (cours-types réutilisables)
│   ├── [MODÈLE] Module 1 — Fondamentaux
│   ├── [MODÈLE] Module 2 — Didactique
│   ├── [MODÈLE] Module 3 — Évaluation
│   ├── [MODÈLE] Module 4 — Innovation pédagogique
│   └── [MODÈLE] Module 5 — Synthèse & Certification
│
├── Session 2026-01 (Cohorte A — Casablanca)
│   ├── Module 1, 2, 3, 4, 5
│
├── Session 2026-01 (Cohorte B — Rabat)
│   └── ...
│
└── Session 2026-02 (Cohorte C — Marrakech)
    └── ...
```

Chaque session = une **catégorie** + une **cohorte** + **5 cours** (modules) dupliqués depuis les modèles. L'inscription se fait par synchronisation cohorte (pas d'auto-inscription publique).

---

## 5. Mécanique de Séquencement (Workflow Validation)

Le verrouillage progressif combine deux mécanismes natifs Moodle :

1. **Activity Completion** — suivi d'achèvement activité par activité (score quiz ≥ 70% ou Assignment corrigé)
2. **Restrict Access** — le cours Module N+1 n'est visible que si :
   - Achèvement du Module N = `complete`
   - Champ profil `validation_module_N` = `1` (activé par le responsable pédagogique via le plugin custom)

**Champs profil personnalisés créés :**

| Champ | Type | Description |
|---|---|---|
| `validation_module_1` à `_5` | Checkbox | Module N validé par le responsable pédagogique |
| `certification_obtenue` | Checkbox | Certificat final délivré après jury |
| `etablissement` | Menu déroulant | Établissement d'origine |
| `ville` | Menu déroulant | Ville de l'établissement |

---

## 6. Reporting & KPIs

### KPIs institutionnels mis en place

- Nombre d'inscrits (total et par cohorte)
- **Taux d'activation** — % ayant effectué leur première connexion
- **Utilisateurs actifs 30 jours**
- **Taux de complétion par module**
- **Taux de validation par module** (responsable pédagogique)
- **Taux de certification** — % ayant obtenu le certificat final
- Temps moyen par module
- Taux d'assiduité aux séances
- Note moyenne aux évaluations

### Niveaux de Reporting par profil

| Profil | Outils |
|---|---|
| Apprenant | Block Completion Progress, page Notes |
| Formateur | Rapports natifs cours, Attendance |
| Responsable pédagogique | Dashboard plugin `local_modulvalidation` + Configurable Reports |
| Direction / Institution | Configurable Reports (requêtes SQL custom), exports Excel/PDF/CSV |

---

## 7. Sécurité & Conformité

### Authentification
- Accès exclusivement par email institutionnel contrôlé
- Auto-inscription publique **désactivée**
- **2FA obligatoire** pour administrateurs et responsables pédagogiques (`tool_mfa`)
- Politique mots de passe renforcée (min. 10 caractères, complexité, verrouillage 5 échecs)
- SSO / LDAP prévu en **V2**

### Conformité CNDP (loi 09-08 Maroc)
- Politique de confidentialité publiée avec acceptation obligatoire à la première connexion
- Export et suppression des données utilisateurs sur demande (Privacy API Moodle 4.x)
- Logs d'accès : rétention 18 mois minimum (36 mois recommandé)
- Hébergement européen (OVH/Scaleway) facilitant la conformité

### Sauvegardes

| Type | Fréquence | Rétention | Outil |
|---|---|---|---|
| Base de données | Quotidienne (nuit) | 30 jours | mariabackup + rsync S3 |
| Moodledata (fichiers) | Quotidienne incrémentale | 30 jours | rsync / borgbackup |
| Sauvegarde Moodle complète | Hebdomadaire | 12 semaines | admin/cli/backup.php |
| Snapshot VM | Hebdomadaire | 4 semaines | Fournisseur cloud |

---

## 8. Planning de Développement — Découpage en Sprints

**Durée totale : 13 semaines de développement**  
(+ cadrage amont + recette/formation aval = ~16–18 semaines projet complet)

| Sprint | Durée | Thème | Livrable principal |
|---|---|---|---|
| **Sprint 0** | 1 semaine | Préparation environnement | VMs provisionnées, accès SSH, repo Git, backlog |
| **Sprint 1** | 2 semaines | Installation Moodle + structure de base | Moodle 4.5 installé, catégories, rôles, thème, plugins |
| **Sprint 2** | 2 semaines | Template SP + séquencement modules | Template SP .mbz, verrouillage Module N+1, barre progression |
| **Sprint 3** | 2 semaines | Hybridation (BigBlueButton, H5P, présence) | Classes virtuelles, contenus H5P, gestion présence |
| **Sprint 4** | 2 semaines | Plugin validation + Module 5 | `local_modulvalidation` livré, Module 5 + jury configuré |
| **Sprint 5** | 2 semaines | Reporting & certification | 8 rapports KPI, certificat PDF + QR code |
| **Sprint 6** | 2 semaines | Recette, MEP & formation | Prod en ligne, guides livrés, équipes formées, PV signé |

---

## 9. Livrables du Projet

| # | Livrable | Format |
|---|---|---|
| L1 | Plateforme Moodle opérationnelle en production | URL + accès admin |
| L2 | Plugin custom `local_modulvalidation` | Code source Git + installé |
| L3 | Thème personnalisé charte client | Code source + installé |
| L4 | Guide administrateur | PDF + DOCX |
| L5 | Guide formateur | PDF + DOCX |
| L6 | Guide apprenant + vidéo onboarding (10 min) | PDF + vidéo |
| L7 | Documentation technique (architecture, déploiement) | Markdown + PDF |
| L8 | Procédures opérationnelles (session, sauvegarde, restauration) | PDF |
| L9 | Template SP (.mbz) | Fichier Moodle backup |
| L10 | Rapports Configurable Reports (8 KPI) | Installés + SQL exportés |
| L11 | PV de recette signé | PDF signé |
| L12–L13 | Sessions de formation (admins 0,5j + formateurs 1j) | Feuilles de présence |

---

## 10. Roadmap — Perspectives d'Évolution

### Version 2 (court/moyen terme)
- Authentification SSO (SAML / OAuth2) + synchronisation LDAP
- Authentification MFA étendue
- Interface admin dédiée à la création de sessions (automatisation complète)
- Application mobile Moodle Mobile (personnalisée)
- Support multilingue : ajout de l'arabe
- BI avancée via connecteur Metabase ou Power BI
- Scalelite pour cluster BigBlueButton (>100 classes simultanées)

### Version 3 (vision long terme)
- Analyse automatisée des copies par IA (correction assistée)
- Recommandations pédagogiques personnalisées (Moodle Analytics + ML)
- Dashboard exécutif national consolidant toutes les sessions
- Corrélation données de formation ↔ résultats des élèves
- Plateforme multi-tenant (déploiement multi-institutions/ministères)
- Modélisation du transfert de compétences en classe

---

## 11. Correspondance Terminologie Métier ↔ Moodle

| Terme métier | Équivalent Moodle |
|---|---|
| Parcours de formation | Category (catégorie de cours) |
| Module | Course (cours) |
| SP (Situation Professionnelle) | Section / Topic |
| Groupe / Promotion | Cohort (global) + Group (par cours) |
| Session de formation | Category + Cohort + 5 instances de cours |
| Séance | Attendance session |
| Classe virtuelle | BigBlueButton activity |
| Évaluation | Quiz (QCM auto) / Assignment (rédactionnel) |
| Certificat | Custom Certificate (plugin) |

---

*Document généré le 28 avril 2026 — Sources : `Plateforme_Nationale_Formation_Enseignants_V2.pdf` & `Plateforme_Enseignants_Document_Technique_Moodle.pdf`*
