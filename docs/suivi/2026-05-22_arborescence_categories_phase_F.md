# Suivi — Arborescence des catégories (Phase F)

> **Date :** 22 mai 2026
> **Contexte :** Sprint 1 — Phase F du `MOODLE_SETUP.md`
> **Statut :** ✅ Arborescence créée, catégorie par défaut nettoyée

---

## 1. Arborescence cible

```
Formation Enseignants (FE)              [visible]
├── Modèles (FE_MODELES)                [cachée — non visible des apprenants]
└── Sessions (FE_SESSIONS)              [visible]
```

Sous `Sessions`, on créera plus tard des sous-catégories filles, une par cohorte (`Session 2026-01 — Cohorte A — Casablanca`, etc.) — Phase K.

---

## 2. Approche technique

### Pourquoi pas en SQL brut ?
Créer une catégorie Moodle n'est pas juste un `INSERT` dans `mdl_course_categories` : il faut aussi
- gérer le `path` (ex. `/2/3`) et `depth`
- créer une entrée dans `mdl_context` (contextlevel=40)
- initialiser les capabilities héritées
- caches internes Moodle (course catalog cache)

→ Pour éviter d'oublier un de ces points, on appelle directement l'API Moodle `core_course_category::create()` via un petit script CLI ad-hoc.

### Script utilisé
Placé temporairement dans `admin/cli/create_categories_phase_f.php`, supprimé après usage.

```php
<?php
define('CLI_SCRIPT', true);
require(__DIR__ . '/../../config.php');
require_once($CFG->libdir . '/clilib.php');
// core_course_category est auto-chargée depuis Moodle 3.6+

$root = core_course_category::create((object)[
    'name'        => 'Formation Enseignants',
    'idnumber'    => 'FE',
    'description' => 'Catégorie racine du programme national de formation des enseignants.',
    'descriptionformat' => FORMAT_HTML,
    'parent'      => 0,
    'visible'     => 1,
]);

core_course_category::create((object)[
    'name'        => 'Modèles',
    'idnumber'    => 'FE_MODELES',
    'description' => 'Cours-types réutilisables (non visibles des apprenants).',
    'descriptionformat' => FORMAT_HTML,
    'parent'      => $root->id,
    'visible'     => 0,  // ← masqué
]);

core_course_category::create((object)[
    'name'        => 'Sessions',
    'idnumber'    => 'FE_SESSIONS',
    'description' => 'Sessions de formation actives (catégories filles : une par cohorte).',
    'descriptionformat' => FORMAT_HTML,
    'parent'      => $root->id,
    'visible'     => 1,
]);
```

Exécution :
```bash
/opt/alt/php83/usr/bin/php /home/lmsmoodle/public_html/moodle/admin/cli/create_categories_phase_f.php
```

### Nettoyage : suppression de la catégorie par défaut Moodle
Moodle crée une catégorie générique « Catégorie 1 » à l'installation. Vérifiée vide :
```sql
SELECT id, fullname FROM mdl_course WHERE category=1; -- 0 ligne
```

Supprimée via `core_course_category::delete_full(false)` (gère aussi le context).

Le script ad-hoc a ensuite été **supprimé** du serveur (`admin/cli/create_categories_phase_f.php`).

---

## 3. Vérification post-création

```sql
SELECT id, name, idnumber, parent, depth, path, sortorder, visible
FROM mdl_course_categories
ORDER BY path;
```

| id | name | idnumber | parent | depth | path | sortorder | visible |
|---|---|---|---|---|---|---|---|
| 2 | Formation Enseignants | FE | 0 | 1 | /2 | 20000 | 1 |
| 3 | Modèles | FE_MODELES | 2 | 2 | /2/3 | 30000 | **0** |
| 4 | Sessions | FE_SESSIONS | 2 | 2 | /2/4 | 40000 | 1 |

✅ Tous les champs cohérents (`depth`, `path` correctement enchaînés).

---

## 4. Pièges évités

| Piège | Détail |
|---|---|
| `coursecatlib.php` n'existe plus en 4.x | Première version du script avait `require_once($CFG->libdir . '/coursecatlib.php')` → erreur. La classe `core_course_category` est désormais auto-chargée. |
| Encodage UTF-8 | Le script PHP gère bien l'UTF-8 nativement (pas de problème de double-encodage comme en mysql CLI). |
| Catégorie par défaut Moodle | Si on la laisse, elle apparaît à côté de notre arbo dans les listes de cours, semer la confusion. Supprimée. |

---

## 5. Reste à faire — Phase suivante (G)

- [ ] **Phase G** — créer les 5 cours modèles dans la catégorie `Modèles` :
  - `[MODÈLE] Module 1 — Fondamentaux`
  - `[MODÈLE] Module 2 — Didactique`
  - `[MODÈLE] Module 3 — Évaluation`
  - `[MODÈLE] Module 4 — Innovation pédagogique`
  - `[MODÈLE] Module 5 — Synthèse & Certification`
- [ ] **Phase H** — créer le template SP (Situation Professionnelle) qui sera dupliqué dans chaque module

---

## 6. Bilan Sprint 1

| Phase | Statut |
|---|---|
| A — Installation Moodle | ✅ |
| B — Config globale (scriptable) | ✅ (reste fourniture client : CNDP, charte, SMTP) |
| C — Plugins | ✅ (8 plugins) |
| D — Champs profil | ✅ (3 catégories, 8 champs) |
| E — Rôles custom | ✅ (responsable_pedago 15 caps, jury_member 10 caps) |
| F — Arborescence catégories | ✅ |

**Sprint 1 du brief technique : terminé** 🎉 — on passe au Sprint 2 (Template SP + séquencement).

---

*Document rédigé le 22 mai 2026 — Clôture Sprint 1 (Phase F)*
