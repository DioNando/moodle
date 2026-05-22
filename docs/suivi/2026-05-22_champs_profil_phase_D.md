# Suivi — Champs profil utilisateur custom (Phase D)

> **Date :** 22 mai 2026
> **Contexte :** Sprint 1 — Phase D du `MOODLE_SETUP.md`
> **Statut :** ✅ Catégories et champs créés via SQL ; liste `etablissement` à compléter par le client

---

## 1. Pourquoi ces champs

Les champs profil custom sont la pierre angulaire de deux mécanismes du projet :

1. **Séquencement des modules** : les checkbox `validation_module_1..5` sont lues par les *Restrictions d'accès* du Module N+1 → tant qu'un responsable pédagogique ne coche pas la case, l'apprenant ne peut pas accéder au module suivant.
2. **Pilotage institutionnel** : les champs `etablissement` / `ville` alimentent les rapports KPI (taux d'activation par établissement, par ville, etc.).
3. **Traçabilité de la certification** : `certification_obtenue` matérialise la délivrance du certificat final après le jury Module 5.

---

## 2. Catégories créées

| ID | Nom | Ordre | Champs |
|----|---|---|---|
| 1 | **Identité** | 1 | etablissement, ville |
| 2 | **Progression** | 2 | validation_module_1..5 |
| 3 | **Certification** | 3 | certification_obtenue |

---

## 3. Champs créés

### 3.1 Catégorie « Progression » (5 checkbox)

| Nom court | Nom affiché | Type | Visible | Verrouillé | Requis |
|---|---|---|---|---|---|
| `validation_module_1` | Module 1 validé par responsable pédago | checkbox | Non | Oui | Non |
| `validation_module_2` | Module 2 validé par responsable pédago | checkbox | Non | Oui | Non |
| `validation_module_3` | Module 3 validé par responsable pédago | checkbox | Non | Oui | Non |
| `validation_module_4` | Module 4 validé par responsable pédago | checkbox | Non | Oui | Non |
| `validation_module_5` | Module 5 validé par responsable pédago | checkbox | Non | Oui | Non |

> `visible = 0` (invisible utilisateur) + `locked = 1` (non modifiable par l'utilisateur) : ces cases ne seront cochées que par le **responsable pédagogique** via le futur plugin `local_modulvalidation`. Valeur par défaut : `0` (décoché).

### 3.2 Catégorie « Certification » (1 checkbox)

| Nom court | Nom affiché | Type | Visible | Verrouillé |
|---|---|---|---|---|
| `certification_obtenue` | Certificat final délivré | checkbox | Non | Oui |

### 3.3 Catégorie « Identité » (2 menus déroulants)

| Nom court | Nom affiché | Type | Visible | Requis | Verrouillé |
|---|---|---|---|---|---|
| `etablissement` | Établissement d'origine | menu | Tous | Oui | Oui |
| `ville` | Ville de l'établissement | menu | Tous | Oui | Oui |

> `visible = 2` (visible par tous) + `required = 1` + `locked = 1` : l'utilisateur voit l'info dans son profil mais ne peut pas la modifier. La saisie se fait par l'administrateur (manuellement ou via import CSV).

#### Options provisoires
- **`ville`** (pré-rempli, 18 grandes villes marocaines, modifiable client) :
  ```
  Casablanca, Rabat, Marrakech, Fès, Tanger, Agadir, Meknès, Oujda,
  Kénitra, Tétouan, Safi, El Jadida, Béni Mellal, Nador, Taza,
  Settat, Larache, Khouribga
  ```
- **`etablissement`** : valeur placeholder *« — À compléter par le client — »* (liste des établissements partenaires à fournir).

---

## 4. Commandes SQL utilisées

```sql
-- 4.1 Catégories
INSERT INTO mdl_user_info_category (name, sortorder) VALUES
  ('Identité', 1),
  ('Progression', 2),
  ('Certification', 3);

-- 4.2 Récupération des IDs
SET @cat_identite      := (SELECT id FROM mdl_user_info_category WHERE name='Identité');
SET @cat_progression   := (SELECT id FROM mdl_user_info_category WHERE name='Progression');
SET @cat_certification := (SELECT id FROM mdl_user_info_category WHERE name='Certification');

-- 4.3 Checkbox de validation (visible=0, locked=1)
INSERT INTO mdl_user_info_field
  (shortname, name, datatype, descriptionformat, categoryid, sortorder,
   required, locked, visible, forceunique, signup, defaultdata, defaultdataformat)
VALUES
  ('validation_module_1', 'Module 1 validé par responsable pédago', 'checkbox', 1, @cat_progression, 1, 0, 1, 0, 0, 0, '0', 0),
  -- ... etc pour modules 2..5 ...
  ('certification_obtenue', 'Certificat final délivré', 'checkbox', 1, @cat_certification, 1, 0, 1, 0, 0, 0, '0', 0);

-- 4.4 Menus déroulants (visible=2, required=1, locked=1)
INSERT INTO mdl_user_info_field (..., param1)
VALUES
  ('etablissement', 'Établissement d''origine', 'menu', ..., '— À compléter par le client —'),
  ('ville', 'Ville de l''établissement', 'menu', ..., 'Casablanca\nRabat\n...');
```

Puis :
```bash
/opt/alt/php83/usr/bin/php admin/cli/purge_caches.php
```

> ⚠️ **Astuce MySQL #1 — Retours à la ligne** : les `\n` littéraux ne sont pas interprétés en mode texte. Pour les listes de menu (`param1`), il faut utiliser **de vrais retours à la ligne** dans le heredoc ou `CHAR(10)`, sinon Moodle affiche une seule entrée géante.
>
> ⚠️ **Astuce MySQL #2 — Charset UTF-8** : le client `mysql` en CLI utilise par défaut le charset système (souvent `latin1`). Sans précaution, les chaînes UTF-8 contenant des accents (é, è, à…) sont **double-encodées** : « Module validé » devient « Module validÃ© ». Pour toutes les écritures contenant des accents, **toujours utiliser** :
> ```bash
> mysql --default-character-set=utf8mb4 -u ... lmsmoodle_moodle
> ```
> ou, en début de session SQL :
> ```sql
> SET NAMES utf8mb4;
> ```
> **Diagnostic d'un double-encodage** :
> ```sql
> SELECT shortname, HEX(name) FROM mdl_user_info_field WHERE shortname='validation_module_1';
> -- Si on voit "C383C2A9" au lieu de "C3A9" pour "é" → double-encodage confirmé
> ```
> **Réparation** : ré-injecter les valeurs avec la bonne session encoding (a été fait pour les 8 champs + 1 catégorie le 22/05/2026).

---

## 5. Vérification

| Vérif | Commande | Résultat |
|---|---|---|
| Catégories en place | `SELECT * FROM mdl_user_info_category` | 3 lignes ✅ |
| Champs en place | `SELECT shortname FROM mdl_user_info_field` | 8 lignes ✅ |
| Options ville | `SELECT param1 FROM mdl_user_info_field WHERE shortname='ville'` | 18 villes séparées par `\n` ✅ |
| Préfix correct | aucun nom préfixé par `profile_field_*` | Les checkbox Moodle s'appellent `profile_field_validation_module_N` dans les conditions d'accès — c'est correct, le préfixe est automatique côté lecture |

### Vérif visuelle UI (recommandée après login admin)
1. `https://lms-moodle.preprod.io/moodle/user/profile/index.php`
2. Doit afficher 3 catégories avec les 8 champs
3. Créer un utilisateur test → l'écran doit présenter :
   - section *Identité* : 2 menus déroulants (ville et établissement)
   - section *Progression* : pas visible si on est connecté en tant que l'utilisateur lui-même (visible=0)
   - section *Certification* : idem

---

## 6. Reste à faire

- [ ] **[client]** Liste finale des établissements à mettre dans `etablissement` (param1)
- [ ] **[client]** Vérifier que la liste des 18 villes est exhaustive vs. besoins métier
- [ ] **[Phase E]** Créer les rôles `responsable_pedago` et `jury_member`
- [ ] **[Phase F]** Créer l'arborescence des catégories de cours
- [ ] **[Sprint 4]** Développer le plugin `local_modulvalidation` qui pilotera ces checkbox

---

## 7. Note pour le plugin custom (anticipation)

Le futur plugin `local_modulvalidation` devra :
1. Lister les apprenants d'une session
2. Pour chaque apprenant + chaque module : afficher l'état des checkbox `validation_module_N`
3. Permettre au responsable pédagogique de cocher/décocher → écriture dans `mdl_user_info_data` (table de stockage des valeurs par utilisateur)

Schéma cible (pour info dev) :
```sql
-- Récupérer/écrire la valeur d'un champ pour un user
SELECT d.data
FROM mdl_user_info_data d
JOIN mdl_user_info_field f ON f.id = d.fieldid
WHERE d.userid = ? AND f.shortname = 'validation_module_3';
```

---

*Document rédigé le 22 mai 2026 — Suivi Sprint 1 (Phase D)*
