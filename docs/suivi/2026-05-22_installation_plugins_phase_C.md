# Suivi — Installation des plugins (Phase C)

> **Date :** 22 mai 2026
> **Contexte :** Sprint 1 — Phase C du `MOODLE_SETUP.md` (initialement zappée, reprise après Phase E)
> **Statut :** ✅ 8 plugins installés et opérationnels (7 requis + 1 optionnel)

---

## 1. Plugins installés

| Plugin | Type | Version installée | Statut | Usage métier |
|---|---|---|---|---|
| `mod_attendance` | Activity | 2024082402 | ✅ Requis V1 | Gestion des présences présentiel/distanciel |
| `mod_customcert` | Activity | 2024042217 | ✅ Requis V1 | Certificats PDF + QR code |
| `mod_scheduler` | Activity | 2023052300 | ✅ Requis V1 | Créneaux de soutenance Module 5 |
| `mod_checklist` | Activity | 2026042400 | ✅ Requis V1 | Listes de contrôle pédagogiques |
| `mod_choicegroup` | Activity | 2026013100 | ✅ Optionnel | Auto-inscription aux groupes |
| `block_configurable_reports` | Block | 2027050401 | ✅ Requis V1 | Rapports SQL custom (KPI) |
| `block_completion_progress` | Block | 2026042700 | ✅ Requis V1 | Barre de progression apprenant |
| `filter_poodll` | Filter | 2026042000 | ✅ Optionnel | Feedback audio/vidéo sur devoirs |

### À noter — déjà dans le core Moodle 4.5
| Composant | Notes |
|---|---|
| `mod_bigbluebuttonbn` | Module intégré au core depuis Moodle 4.0 — **pas besoin d'installer** |
| H5P (Content Bank) | Natif Moodle 4.x |
| MFA (`tool_mfa`) | Natif Moodle 4.5 |
| `block_myoverview` | Natif |

---

## 2. Méthode d'installation utilisée

### 2.1 Upload des ZIPs
Le client a déposé tous les ZIPs (téléchargés depuis moodle.org) dans :
```
/home/lmsmoodle/plugins_zips/
```

### 2.2 Vérification de compatibilité préalable
Moodle 4.5 = version DB `2024100700`. Chaque plugin a été inspecté avant extraction pour vérifier que `$plugin->requires` ≤ 2024100700 :

```bash
for z in *.zip; do
  unzip -p "$z" $(unzip -Z1 "$z" | grep version.php$ | head -1) \
    | grep -E "plugin->(component|requires|release)"
done
```

Résultats (tous compatibles) :

| Plugin | `requires` (Moodle min) |
|---|---|
| block_completion_progress | 2024100700 (4.5) |
| block_configurable_reports | 2022041900 (4.0) |
| filter_poodll | 2023100900 (4.3) |
| mod_attendance | 2024092700 (4.5) |
| mod_checklist | 2022112800 (4.1) |
| mod_choicegroup | 2023100900 (4.3) |
| mod_customcert | 2024042200 (4.4) |
| mod_scheduler | 2022041900 (4.0) |

> ⚠️ Le suffixe `moodleXX` dans le nom de fichier (`mod_checklist_moodle52_*.zip`) indique la version **cible/testée** par le mainteneur, **pas** la version minimum. Toujours vérifier `$plugin->requires` dans `version.php`.

### 2.3 Extraction dans les bons dossiers

Mapping appliqué :

| Préfixe ZIP | Destination Moodle |
|---|---|
| `mod_*` | `moodle/mod/<nom_court>/` |
| `block_*` | `moodle/blocks/<nom_court>/` |
| `filter_*` | `moodle/filter/<nom_court>/` |

Le ZIP contient déjà un dossier top-level nommé (`<nom_court>/`), donc `unzip -d <destination>` suffit.

```bash
unzip -q -o mod_attendance_*.zip -d /home/lmsmoodle/public_html/moodle/mod
# → crée moodle/mod/attendance/
```

### 2.4 Upgrade BDD

```bash
/opt/alt/php83/usr/bin/php \
  -d max_input_vars=5000 -d memory_limit=512M -d max_execution_time=600 \
  /home/lmsmoodle/public_html/moodle/admin/cli/upgrade.php --non-interactive
```

Sortie finale :
> Mise à jour en ligne de commande de la version 4.5.10 (Build: 20260216) (2024100710) à la version 4.5.10 (Build: 20260216) (2024100710) terminée avec succès.

→ Création des tables propres à chaque plugin + initialisation des réglages par défaut.

### 2.5 Purge des caches
```bash
/opt/alt/php83/usr/bin/php /home/lmsmoodle/public_html/moodle/admin/cli/purge_caches.php
```

---

## 3. Vérifications post-install

### 3.1 Versions enregistrées en BDD
```sql
SELECT plugin, value AS version FROM mdl_config_plugins
WHERE plugin IN ('mod_attendance','mod_customcert','mod_scheduler','mod_checklist','mod_choicegroup',
                 'block_configurable_reports','block_completion_progress','filter_poodll')
  AND name='version'
ORDER BY plugin;
```

Toutes les versions correspondent aux ZIPs (cf. tableau § 1). ✅

### 3.2 Vérif visuelle (recommandée)
`Administration du site > Plugins > Vue d'ensemble des plugins`
→ les 8 plugins doivent apparaître en vert (à jour, sans alerte).

---

## 4. Impact sur les rôles déjà créés (Phase E)

Maintenant que `mod_attendance` est installé, on peut compléter le rôle `responsable_pedago` avec :
- `mod/attendance:view`
- `mod/attendance:viewreports`

À faire :
```sql
INSERT INTO mdl_role_capabilities (contextid, roleid, capability, permission, timemodified, modifierid)
SELECT 1,
       (SELECT id FROM mdl_role WHERE shortname='responsable_pedago'),
       c.name, 1, UNIX_TIMESTAMP(), 2
FROM (SELECT 'mod/attendance:view' AS name UNION ALL SELECT 'mod/attendance:viewreports') c
WHERE EXISTS (SELECT 1 FROM mdl_capabilities cap WHERE cap.name = c.name);
```

(à faire dans un suivi dédié si on y procède maintenant)

---

## 5. Reste à faire

- [ ] **Vérif UI** : Site administration → Plugins → Plugin overview (statuts verts)
- [ ] **Ajustement rôle `responsable_pedago`** : ajouter capabilities `mod/attendance:*` maintenant que le plugin est installé
- [ ] **Phase F** — arborescence catégories
- [ ] **Sprint 3** — paramétrage BBB (URL + secret), configuration certificat type
- [ ] **Sprint 5** — création des 8 rapports Configurable Reports KPI

---

*Document rédigé le 22 mai 2026 — Suivi Sprint 1 (Phase C plugins)*
