# Suivi — Sprint 2 : Template SP + séquencement (Phases H + I)

> **Date :** 22 mai 2026
> **Contexte :** Sprint 2 du brief technique — construction de la structure pédagogique de base
> **Statut :** ✅ Template SP créé dans M1..M5, completion + restrictions actives

---

## 1. Objectif Sprint 2

Mettre en place :
1. Une **SP standard** (Situation Professionnelle) avec ses 6 types d'activités, dans le module modèle M1
2. La **réplication** de cette SP dans les modules M2..M5 (chaque module aura sa SP X.1 de démo)
3. L'**activity completion** sur chaque activité
4. La **course completion** sur chaque module (= terminer l'évaluation)
5. Les **restrictions d'accès** : un module N+1 ne devient accessible que si `validation_module_N = 1`

---

## 2. Structure d'une SP créée

Dans chaque cours `[MODÈLE] Module X`, **section 1** renommée `SP X.1 — Démo`, contenant :

| # | Type | Nom | Completion | Notes |
|---|---|---|---|---|
| 1 | Page | Introduction & Objectifs | Auto (vue requise) | HTML structuré avec sections Introduction / Objectifs / Volume horaire |
| 2 | Dossier | Supports documentaires | Manuel | Display inline, sous-dossiers visibles |
| 3 | URL | Vidéo de la SP | Auto (vue requise) | Placeholder YouTube — à remplacer |
| 4 | Forum | Échanges & questions | Manuel | Type "général", tracking activé |
| 5 | Devoir | Évaluation SP X.1 | Auto (note ≥ 10/20) | Texte en ligne + fichiers (3 × 20 Mo max) |

> **Note :** H5P et Quiz omis pour Sprint 2 :
> - H5P nécessite des contenus dans la Content Bank (à faire avec le client)
> - Quiz nécessite des questions configurées (à faire par le formateur)
> - Les deux sont prévus dans la spec « optionnelle » — ajoutables en Sprint 3 / formation formateur

---

## 3. Activity completion (Phase I.1)

Configurée à la création de chaque activité :

| Activité | `completion` | `completionview` | `completionusegrade` | `completionpassgrade` |
|---|---|---|---|---|
| Page | 2 (auto) | 1 (vue requise) | — | — |
| Dossier | 1 (manuel) | — | — | — |
| URL | 2 (auto) | 1 (vue requise) | — | — |
| Forum | 1 (manuel) | — | — | — |
| Devoir | 2 (auto) | — | 1 (utiliser la note) | 1 (note de passage requise) |

---

## 4. Course completion (Phase I.2)

Pour chaque cours modèle, **critère unique** : achever le devoir d'évaluation de la SP.

```sql
-- Représentation BDD
INSERT INTO mdl_course_completion_criteria
  (course, criteriatype, moduleinstance) VALUES
  (<courseid>, 4, <cmid_devoir>);  -- criteriatype 4 = ACTIVITY
```

> Méthode d'agrégation : `COMPLETION_AGGREGATION_ALL` (« toutes les conditions doivent être remplies ») — par sécurité même si on n'a qu'un critère pour l'instant. Quand on ajoutera d'autres SP par module, on ajoutera leurs devoirs comme nouveaux critères.

---

## 5. Restrictions d'accès (Phase I.3)

### 5.1 Choix d'implémentation
Le brief mentionne « restreindre l'accès au cours Module N+1 ». Or **les cours Moodle 4.5 ne possèdent pas de colonne `availability` dédiée** (vérifié sur `mdl_course`). Les restrictions JSON existent uniquement sur :
- `mdl_course_modules.availability` (activités)
- `mdl_course_sections.availability` (sections)

→ **Décision :** appliquer la restriction sur **toutes les sections** > 0 de chaque module N+1. Effet visible pour l'apprenant : le cours apparaît dans son tableau de bord mais toutes les sections sont grisées avec le message « Non disponible (champ validation_module_X requis = 1) ».

### 5.2 JSON d'availability appliqué

Pour MODELE_M2 (et identique pour M3/M4/M5 en changeant le N) :
```json
{
  "op": "&",
  "c": [
    {"type":"profile","sf":"validation_module_1","op":"isequalto","v":"1"}
  ],
  "showc": [true]
}
```

Appliqué via :
```sql
UPDATE mdl_course_sections SET availability = '<JSON>'
WHERE course IN (3,4,5,6) AND section > 0;
```

### 5.3 Vérification
| Cours | Restriction (sections 1..5) |
|---|---|
| MODELE_M2 | `validation_module_1 = 1` |
| MODELE_M3 | `validation_module_2 = 1` |
| MODELE_M4 | `validation_module_3 = 1` |
| MODELE_M5 | `validation_module_4 = 1` |

> ⚠️ La condition « completion du cours précédent » n'est pas implémentée ici (complexité du cross-course en JSON natif Moodle). Elle est implicitement portée par le **responsable pédagogique** qui ne coche `validation_module_N=1` que si l'apprenant a effectivement complété le module N. C'est la logique-métier du brief (validation manuelle pilotée).

---

## 6. Approche technique — script CLI

Tout réalisé via un script ad-hoc `admin/cli/sprint2_template_sp.php` (supprimé après usage).

### Helpers utilisés
- `add_moduleinfo($moduleinfo, $course)` — création générique d'un cm
- `core_course_category` + `core_course` — gestion catégories/cours
- `completion_criteria_activity::update_config()` — pose des critères de completion
- `rebuild_course_cache()` — invalide les caches après mise à jour des sections

### Pièges rencontrés

#### 6.1 `mod_assign` — colonnes NOT NULL obligatoires
La création d'un devoir via `add_moduleinfo()` casse si on omet certaines colonnes NOT NULL sans default :
```
Column 'requiresubmissionstatement' cannot be null
```
**Correctif** : toujours passer explicitement ces colonnes à la création :
- `requiresubmissionstatement => 0`
- `completionsubmit => 0`
- `sendstudentnotifications => 1`

#### 6.2 Erreurs DML masquées par défaut
`Exception : Erreur d'écriture vers la base de données` ne révèle pas la cause. Pour voir la vraie erreur MySQL, en mode debug Moodle :
```php
$CFG->debug = 32767;
$CFG->debugdisplay = 1;
$CFG->debugdeveloper = true;
```

#### 6.3 Champ `availability` absent sur `mdl_course`
Idée fausse initiale : poser la restriction directement sur le cours. **Correctif** : poser sur toutes les sections enfants.

#### 6.4 Reprise après échec partiel
Lors des essais, le script avait créé des modules en partie puis échoué — laissant des modules orphelins. Pattern de nettoyage utilisé :
```sql
DELETE FROM mdl_course_modules WHERE course IN (...) AND id > 1;
DELETE FROM mdl_page WHERE id > 0;
DELETE FROM mdl_folder WHERE id > 0;
DELETE FROM mdl_url WHERE id > 0;
DELETE FROM mdl_assign WHERE id > 0;
DELETE FROM mdl_forum WHERE id NOT IN (
  SELECT instance FROM mdl_course_modules cm
  JOIN mdl_modules m ON m.id=cm.module WHERE m.name='forum');
DELETE FROM mdl_course_completion_criteria WHERE course IN (...);
UPDATE mdl_course_sections SET name=NULL, sequence='' WHERE course IN (...) AND section=1;
```

> 💡 Pour l'idempotence : le script vérifie l'existence de `Évaluation SP X.1` avant de recréer la SP → permet de relancer sans dupliquer.

---

## 7. Pourquoi pas de backup .mbz / restore (comme dans le brief) ?

Le brief propose la voie « créer la SP dans M1, l'exporter en `.mbz`, la restaurer dans M2..M5 ». C'est la voie UI manuelle. En **scripté** (CLI), la duplication directe par boucle est plus rapide, plus déterministe, et évite le passage par un fichier intermédiaire. La structure obtenue est strictement équivalente.

> Pour la **production**, il sera tout de même pertinent d'exporter la SP de M1 en `.mbz` une fois enrichie (avec H5P, vidéos réelles, etc.) → archivé dans `/templates/SP_TEMPLATE.mbz` (livrable L9 du brief). Cette étape est à faire après ajout des contenus réels.

---

## 8. Reste à faire

### Bloqué client
- [ ] Vidéos pédagogiques réelles → remplacer les URLs placeholder
- [ ] Contenus H5P à ajouter à la Content Bank (si voulu)
- [ ] Questions Quiz à définir avec les formateurs

### Phase L (Sprint 4)
- [ ] Développer plugin `local_modulvalidation` qui pilotera les checkbox `validation_module_N` côté responsable_pedago. Tant que ça n'existe pas, la validation se fait manuellement (édition de profil utilisateur par l'admin).

### Phase H — Bloc « Completion Progress »
- [ ] Ajouter le bloc `block_completion_progress` au tableau de bord par défaut → barre de progression visible par l'apprenant
- [ ] Faisable via UI ou via script (config `Administration du site > Apparence > Pages par défaut > Tableau de bord`)

### Test du workflow complet (recommandé)
1. Se connecter en tant que `apprenant1` (inscrit dans MODELE_M1) :
   - Doit voir SP 1.1 et ses 5 activités
   - Doit soumettre le devoir, attendre la note ≥ 10/20
2. Vérifier MODELE_M2 : sections grisées (validation_module_1=0)
3. Se connecter en tant qu'admin → éditer profil apprenant1 → cocher `validation_module_1`
4. Reconnexion apprenant1 → MODELE_M2 doit devenir accessible

---

## 9. État Sprint 2

| Phase | Statut |
|---|---|
| G — 5 cours modèles vides | ✅ (fait en clôture Sprint 1) |
| H — Template SP (Page/Folder/URL/Forum/Assign) dans M1..M5 | ✅ |
| H — Backup .mbz / livrable L9 | ⏳ Après enrichissement contenus |
| I — Activity completion | ✅ |
| I — Course completion | ✅ |
| I — Restrict access modules N+1 | ✅ |
| I — Bloc Completion Progress sur dashboard | ⏳ À faire |

---

*Document rédigé le 22 mai 2026 — Sprint 2 (Phases H + I)*
