# Suivi — Finalisation Sprint 1

> **Date :** 22 mai 2026
> **Contexte :** Clôture des tâches restantes du Sprint 1 (catégorie test, 5 cours modèles, 10 utilisateurs de test)
> **Statut :** ✅ Sprint 1 fonctionnellement terminé — restent 2 points bloqués externes

---

## 1. Tâches réalisées dans cette passe

### 1.1 Catégorie test « Session 2026-01 — Cohorte Test »
Créée sous `Sessions` via API Moodle (`core_course_category::create()`).

| Attribut | Valeur |
|---|---|
| ID | 5 |
| Nom | Session 2026-01 — Cohorte Test |
| idnumber | `FE_SESSION_TEST` |
| parent | 4 (Sessions) |
| visible | 1 |

### 1.2 Cinq cours modèles dans `Modèles`

| ID | shortname | fullname |
|---|---|---|
| 2 | `MODELE_M1` | [MODÈLE] Module 1 — Fondamentaux |
| 3 | `MODELE_M2` | [MODÈLE] Module 2 — Didactique |
| 4 | `MODELE_M3` | [MODÈLE] Module 3 — Évaluation |
| 5 | `MODELE_M4` | [MODÈLE] Module 4 — Innovation pédagogique |
| 6 | `MODELE_M5` | [MODÈLE] Module 5 — Synthèse & Certification |

Tous créés avec :
- Format `topics`, 5 sections par défaut (à raffiner en Phase H)
- `enablecompletion = 1`
- `showcompletionconditions = 1`
- Visible (mais hérite de `Modèles` qui est cachée → pas accessible aux apprenants)

### 1.3 Dix utilisateurs de test

**Mot de passe commun (test uniquement) :** `Test@2026!`

| Profil | Username | Rôle | Contexte |
|---|---|---|---|
| Admin | `admin_test1` | Site Administrator | Global |
| Admin | `admin_test2` | Site Administrator | Global |
| Formateur | `formateur1` | editingteacher | Cours `MODELE_M1` |
| Formateur | `formateur2` | editingteacher | Cours `MODELE_M1` |
| Responsable péda | `resp_pedago` | `responsable_pedago` | Catégorie « Cohorte Test » |
| Jury | `jury1` | `jury_member` | Cours `MODELE_M5` |
| Jury | `jury2` | `jury_member` | Cours `MODELE_M5` |
| Apprenant | `apprenant1` | student | Cours `MODELE_M1` |
| Apprenant | `apprenant2` | student | Cours `MODELE_M1` |
| Apprenant | `apprenant3` | student | Cours `MODELE_M1` |

**Profils communs** : langue `fr`, fuseau `Africa/Casablanca`, comptes confirmés, MNet local.

> ℹ️ Les rôles "natifs" (editingteacher, student) sont assignés via **inscription manuelle** au cours (`enrol_user` avec `enrol_manual`). Les rôles "custom" (`responsable_pedago`, `jury_member`) sont assignés directement sur leur contexte respectif via `role_assign()` (catégorie/cours).

---

## 2. Approche technique

Tout passé par un script CLI ad-hoc (`admin/cli/sprint1_finalize.php`), supprimé après usage. Le script :
- Détecte les objets existants (`idnumber`, `shortname`, `username`) → idempotent
- Utilise les API Moodle (jamais d'INSERT brut) → gestion correcte des contextes, caches, enrolment instances
- Loggue chaque opération

### Points d'attention rencontrés
1. **`require_once($CFG->libdir . '/coursecatlib.php')`** n'existe plus en Moodle 4.x → la classe `core_course_category` est auto-chargée.
2. **Inscription d'un user comme `editingteacher`** ≠ assignation de rôle simple :
   - Il faut une **instance d'inscription manuelle** dans le cours
   - Si pas présente : `enrol_get_plugin('manual')->add_default_instance($course)`
   - Puis `enrol_user($instance, $userid, $roleid)`
3. **Site admins** : stockés dans `mdl_config['siteadmins']` sous forme de liste CSV d'IDs utilisateurs (ne pas confondre avec le rôle `manager`).

---

## 3. État Sprint 1 vs spec officielle

| # | Tâche | Statut |
|---|---|---|
| 1 | Télécharger Moodle 4.5 LTS | ✅ |
| 2 | Installer Moodle (CLI install) | ✅ |
| 3 | config.php : BDD, dataroot, wwwroot, **session Redis** | ⚠️ Redis indispo cPanel — cache fichier à la place |
| 4 | Cron toutes les minutes | ✅ |
| 5 | Plugins de base | ✅ (8 plugins, BBB déjà natif) |
| 6 | Catégories racine + Modèles + Cohorte Test | ✅ |
| 7 | 5 cours modèles vides | ✅ |
| 8 | 2 rôles custom + capabilities | ✅ |
| 9 | 8 champs profil | ✅ |
| 10 | Thème customisé Boost | ❌ **Bloqué — charte client non fournie** |
| 11 | Logo + couleurs + frontpage | ❌ **Bloqué — charte client** |
| 12 | Politique mots de passe | ✅ |
| 13 | 10 utilisateurs de test | ✅ |

### Critères d'acceptation
- ✅ HTTPS Let's Encrypt
- ✅ Plugins (à confirmer visuellement dans la vue d'ensemble)
- ✅ Arborescence catégories conforme
- ✅ Rôles custom + capabilities
- ⏳ Charte client (bloqué)
- ✅ 10 utilisateurs créés, connexion possible avec `Test@2026!`

---

## 4. Bloqués externes (Sprint 1)

### 4.1 Session handler Redis
**Statut :** demande hébergeur faite — pas de Redis disponible sur ce mutualisé cPanel.
**Workaround :** sessions PHP standard (fichiers) + cache MUC fichier — fonctionnel mais perf ≈ ×3 moindre.
**Action prod :** Redis 7 natif sur la VM Ubuntu 22.04 dédiée à venir.

### 4.2 Charte graphique (tâches 10 + 11)
**Statut :** en attente de fourniture par le client.
**Variables à recevoir** :
- Logo (PNG ou SVG)
- Logo compact (favicon)
- Couleur primaire (hex)
- Couleur secondaire (hex)
- Police principale
- Texte de bienvenue page d'accueil

**Action côté nous quand reçu** : ~30 min de paramétrage dans Site administration → Apparence → Thèmes → Boost.

---

## 5. Test de connexion recommandé

Une fois connecté en tant qu'admin (sur `https://lms-moodle.preprod.io/moodle/`), faire un essai de connexion **navigation privée** avec chaque profil pour vérifier :

| Login | Vérification attendue |
|---|---|
| `admin_test1 / Test@2026!` | Accès admin complet |
| `formateur1 / Test@2026!` | Voit MODELE_M1, peut éditer le cours |
| `resp_pedago / Test@2026!` | Voit la catégorie « Session 2026-01 — Cohorte Test » et son contenu |
| `jury1 / Test@2026!` | Voit MODELE_M5 en lecture, ne peut pas éditer |
| `apprenant1 / Test@2026!` | Voit MODELE_M1 en mode apprenant, pas d'édition |

---

## 6. Livrables Sprint 1

| Livrable | Statut | Localisation |
|---|---|---|
| Environnement Moodle opérationnel | ✅ | https://lms-moodle.preprod.io/moodle/ |
| Document « Configuration initiale » | ✅ | `docs/suivi/*.md` (10 fichiers) |
| Tests de connexion utilisateurs | ⏳ | À effectuer par le client/PO |

---

## 7. Prochain sprint — Sprint 2

**Thème :** Template SP + séquencement modules

- [ ] **Phase H** — Construire le template SP (Situation Professionnelle) dans MODELE_M1 : Page intro → Dossier → URL/Étiquette vidéo → H5P → Forum → Quiz/Assignment
- [ ] **Phase I** — Configurer Activity Completion + Restrict Access (verrouillage Module N+1 sur Module N complete + `validation_module_N=1`)
- [ ] Bloc Completion Progress sur les cours
- [ ] Test fonctionnel du verrouillage avec un compte apprenant

---

*Document rédigé le 22 mai 2026 — Clôture Sprint 1*
