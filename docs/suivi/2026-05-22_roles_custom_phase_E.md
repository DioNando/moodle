# Suivi — Rôles personnalisés (Phase E)

> **Date :** 22 mai 2026
> **Contexte :** Sprint 1 — Phase E du `MOODLE_SETUP.md`
> **Statut :** ✅ Deux rôles créés via SQL, capabilities assignées, contextes restreints

---

## 1. Pourquoi ces rôles

Deux profils métier ne correspondent à aucun rôle natif Moodle :

| Rôle métier | Pourquoi pas un rôle natif ? |
|---|---|
| **Responsable pédagogique** | A besoin de voir tous les cours d'une catégorie (session), de consulter les notes/achèvement, et de valider la progression module par module via le futur plugin custom. Le rôle *Manager* est trop puissant (peut modifier les cours), *Teacher* est trop limité (un seul cours). |
| **Membre du jury** | Accède uniquement au cours **Module 5** pour évaluer les mémoires. Ne doit pas pouvoir éditer les contenus pédagogiques. Le rôle *Teacher* est trop large. |

---

## 2. Rôle `responsable_pedago`

| Attribut | Valeur |
|---|---|
| Nom complet | Responsable pédagogique |
| Nom court | `responsable_pedago` |
| Archétype | *(aucun — créé de zéro)* |
| Contexte autorisé | **Catégorie de cours** (level 40) |
| Capabilities | 13 (allow) |

### 2.1 Capabilities attribuées (toutes ALLOW)

| Capability | Pourquoi |
|---|---|
| `moodle/category:viewcourselist` | Voir la liste des cours de sa catégorie de session |
| `moodle/course:view` | Entrer dans les cours pour consulter |
| `moodle/course:viewhiddencourses` | Voir les cours masqués (modules pas encore ouverts) |
| `moodle/course:viewparticipants` | Voir les inscrits |
| `moodle/site:viewparticipants` | Voir les participants au niveau site |
| `gradereport/grader:view` | Consulter le carnet de notes |
| `gradereport/user:view` | Consulter les notes d'un utilisateur |
| `report/completion:view` | Voir le rapport d'achèvement |
| `report/courseoverview:view` | Vue d'ensemble du cours |
| `report/log:view` | Voir les logs (audit) |
| `moodle/user:viewdetails` / `viewalldetails` | Consulter fiches utilisateur |
| `moodle/user:update` | Modifier les champs profil (= cocher `validation_module_N`) |

### 2.2 Capabilities prévues mais reportées
| Capability | Statut | Pourquoi |
|---|---|---|
| `mod/attendance:view` | ✅ **Ajoutée le 22/05/2026** après install Phase C | OK |
| `mod/attendance:viewreports` | ✅ **Ajoutée le 22/05/2026** après install Phase C | OK |
| `local/modulvalidation:validate` | À ajouter après dev plugin custom (Sprint 4) | Plugin pas encore créé |
| `local/modulvalidation:view` | Idem | Idem |

→ Le rôle compte désormais **15 capabilities** (13 initiales + 2 attendance).

---

## 3. Rôle `jury_member`

| Attribut | Valeur |
|---|---|
| Nom complet | Membre du jury |
| Nom court | `jury_member` |
| Archétype | *(aucun)* |
| Contexte autorisé | **Cours** (level 50) — typiquement uniquement Module 5 |
| Capabilities | 10 (allow) |

### 3.1 Capabilities attribuées

| Capability | Pourquoi |
|---|---|
| `moodle/course:view` | Entrer dans le cours Module 5 |
| `moodle/course:viewparticipants` | Voir les apprenants soutenants |
| `mod/assign:view` | Consulter les devoirs (mémoires) |
| `mod/assign:viewblinddetails` | Voir identités en évaluation anonyme |
| `mod/assign:grade` | Noter le mémoire |
| `mod/assign:reviewgrades` | Voir les notes des autres correcteurs (délibération) |
| `gradereport/user:view` | Consulter les notes des apprenants |
| `moodle/user:viewdetails` | Voir la fiche apprenant |
| `mod/forum:viewdiscussion` | Lire les discussions jury |
| `mod/forum:replypost` | Répondre dans le forum de délibération |

> Note : la spec mentionnait `mod/assign:reviewotherusers` qui n'existe pas dans Moodle 4.5. Remplacé par `mod/assign:reviewgrades` (équivalent fonctionnel pour la délibération inter-correcteurs).

---

## 4. Commandes SQL utilisées

### 4.1 Insertion du rôle (exemple `responsable_pedago`)
```sql
SET NAMES utf8mb4;

INSERT INTO mdl_role (name, shortname, description, sortorder, archetype) VALUES
  ('Responsable pédagogique',
   'responsable_pedago',
   'Valide la progression des apprenants module par module …',
   (SELECT max_so+1 FROM (SELECT MAX(sortorder) AS max_so FROM mdl_role) AS t),
   '');

SET @rid_resp := (SELECT id FROM mdl_role WHERE shortname='responsable_pedago');

-- Contexte autorisé : Catégorie (40)
INSERT INTO mdl_role_context_levels (roleid, contextlevel) VALUES (@rid_resp, 40);

-- Capabilities : ALLOW (permission=1), contextid=1 = système
INSERT INTO mdl_role_capabilities (contextid, roleid, capability, permission, timemodified, modifierid)
SELECT 1, @rid_resp, c.name, 1, UNIX_TIMESTAMP(), 2
FROM (
  SELECT 'moodle/category:viewcourselist' AS name UNION ALL
  SELECT 'moodle/course:view' UNION ALL
  -- ... liste complète ...
  SELECT 'moodle/user:update'
) c
WHERE EXISTS (SELECT 1 FROM mdl_capabilities cap WHERE cap.name = c.name);
```

### 4.2 Notes techniques
- **`contextid = 1`** = contexte système (table `mdl_context` où `contextlevel=10`). Les capabilities sont définies au niveau système puis appliquées par assignation de rôle dans un contexte donné.
- **`permission`** : `1` = ALLOW, `-1` = PREVENT, `-1000` = PROHIBIT, `0` = inherit.
- **`mdl_role_context_levels.contextlevel`** : restreint où le rôle peut être assigné. Valeurs Moodle :
  - 10 = Système | 30 = Utilisateur | 40 = Catégorie | 50 = Cours | 70 = Module | 80 = Bloc
- **Filtre `WHERE EXISTS`** sur `mdl_capabilities` : protège contre l'insertion de capabilities inexistantes (plugins pas encore installés).

---

## 5. Vérifications

```sql
-- Rôles présents
SELECT id, shortname, name, sortorder FROM mdl_role WHERE shortname IN ('responsable_pedago','jury_member');
-- → 2 lignes ✅

-- Contextes
SELECT r.shortname, rcl.contextlevel
FROM mdl_role_context_levels rcl
JOIN mdl_role r ON r.id=rcl.roleid
WHERE r.shortname IN ('responsable_pedago','jury_member');
-- → responsable_pedago=40, jury_member=50 ✅

-- Capabilities
SELECT r.shortname, COUNT(*) FROM mdl_role_capabilities rc
JOIN mdl_role r ON r.id=rc.roleid
WHERE r.shortname IN ('responsable_pedago','jury_member')
GROUP BY r.shortname;
-- → responsable_pedago=13, jury_member=10 ✅
```

### Vérification UI (recommandée)
`Administration du site > Utilisateurs > Permissions > Définir les rôles`

Les deux rôles doivent apparaître en bas de la liste après les rôles natifs (sortorder 9 et 10).

---

## 6. Reste à faire

- [ ] **[Phase C]** Installer `mod_attendance` → ajouter `mod/attendance:view` et `mod/attendance:viewreports` à `responsable_pedago`
- [ ] **[Sprint 4]** Développer `local_modulvalidation` → ajouter `local/modulvalidation:view` et `:validate` à `responsable_pedago`
- [ ] **[Phase F]** Créer l'arborescence catégories (Formation Enseignants > Modèles | Sessions) — prérequis pour assigner `responsable_pedago` à une catégorie de session
- [ ] **[Phase J — Sprint 4/5]** Créer les premiers comptes utilisateurs avec ces rôles (test fonctionnel)

---

## 7. Anticipation — Assignation future

Une fois la Phase F faite (catégories de session créées), les assignations se feront ainsi :

```sql
-- Exemple : assigner un utilisateur comme responsable_pedago de la session "Casablanca-A"
-- 1. Trouver le contextid de la catégorie cible
SELECT c.id AS contextid
FROM mdl_context c
JOIN mdl_course_categories cc ON cc.id = c.instanceid AND c.contextlevel=40
WHERE cc.name = 'Session 2026-01 — Cohorte A — Casablanca';

-- 2. Assigner
INSERT INTO mdl_role_assignments (roleid, contextid, userid, timemodified, modifierid)
VALUES (@rid_resp, <contextid>, <userid>, UNIX_TIMESTAMP(), 2);
```

(Ou plus simple : via UI dans la catégorie → Permissions → Assigner des rôles.)

---

*Document rédigé le 22 mai 2026 — Suivi Sprint 1 (Phase E rôles custom)*
