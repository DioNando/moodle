# Suivi — Configuration globale Moodle (Phase B)

> **Date :** 22 mai 2026
> **Contexte :** Sprint 1 — Phase B du `MOODLE_SETUP.md`
> **Statut :** ✅ Partie scriptable appliquée en BDD ; reste les éléments nécessitant fourniture client (charte, politique CNDP, SMTP)

---

## 1. Approche

La majorité des paramètres globaux Moodle sont stockés dans la table `mdl_config`. Plutôt que de tout cliquer dans l'UI, j'ai appliqué les réglages en **SQL direct** puis purgé les caches. Cette approche est :
- ✅ Plus rapide et reproductible
- ✅ Auditable (les changements sont tracés dans ce document)
- ✅ Réutilisable pour la production (mêmes commandes à rejouer)

---

## 2. Réglages appliqués

### 2.1 Langue & localisation (§ 5.1)

| Paramètre | Avant | Après | Action |
|---|---|---|---|
| `lang` | `fr` | `fr` | ✅ déjà OK (install CLI) |
| `country` | `MA` | `MA` | ✅ déjà OK |
| `timezone` | `Africa/Casablanca` | `Africa/Casablanca` | ✅ déjà OK |
| `langmenu` | `1` | **`0`** | Masquer le menu de langue (V1 mono-langue) |

### 2.2 Fonctionnalités avancées (§ 5.2)

| Paramètre | Avant | Après | Notes |
|---|---|---|---|
| `enablecompletion` | `1` | `1` | ✅ Suivi d'achèvement actif (requis pour séquencement modules) |
| `enableavailability` | `1` | `1` | ✅ Restrictions d'accès actives (requises pour verrouillage Module N+1) |
| `enablebadges` | `1` | **`0`** | Désactivé en V1 (cf. spec) |

### 2.3 Politique de mots de passe (§ 5.3)

| Paramètre | Avant | Après |
|---|---|---|
| `passwordpolicy` | `1` | `1` ✅ |
| `minpasswordlength` | `8` | **`10`** |
| `minpassworddigits` | `1` | `1` ✅ |
| `minpasswordupper` | `1` | `1` ✅ |
| `minpasswordlower` | `1` | `1` ✅ |
| `minpasswordnonalphanum` | `1` | `1` ✅ |
| `passwordreuselimit` | `0` | **`5`** (refus 5 derniers mots de passe) |
| `lockoutthreshold` | `0` (désactivé) | **`5`** tentatives avant verrouillage |
| `lockoutwindow` | `1800` | `1800` ✅ (30 min) |
| `lockoutduration` | `1800` | `1800` ✅ (30 min) |

### 2.4 Authentification (§ 5.4)

| Paramètre | Avant | Après |
|---|---|---|
| `auth` (plugins activés) | `email` | **`manual`** |
| `registerauth` | (vide) | (vide) ✅ — pas d'auto-inscription |
| `authpreventaccountcreation` | `0` | **`1`** — empêcher création de comptes |

> Conséquence : seuls les administrateurs peuvent créer les comptes utilisateurs. Conforme au cahier des charges (inscription par cohorte uniquement).

### 2.5 Inscriptions par défaut sur les cours (§ 5.5)

| Paramètre | Avant | Après |
|---|---|---|
| `enrol_plugins_enabled` | `manual,guest,self,cohort` | **`manual,cohort`** |

> `enrol_self` et `enrol_guest` retirés. L'inscription se fera exclusivement par synchronisation de cohorte (`enrol_cohort`) ou inscription manuelle ponctuelle (`enrol_manual`).

### 2.6 Identité du site (§ 5.10 partiel)

| Paramètre | Valeur |
|---|---|
| `fullname` (mdl_course id=1) | `Plateforme Nationale Formation Enseignants` ✅ |
| `shortname` | `PNFE` ✅ |
| `summary` | `Programme de formation des enseignants — Maroc` ✅ |

---

## 3. Commandes SQL utilisées

```sql
-- Langue : masquer le menu langue
UPDATE mdl_config SET value='0' WHERE name='langmenu';

-- Fonctionnalités avancées
UPDATE mdl_config SET value='0' WHERE name='enablebadges';

-- Politique mots de passe
UPDATE mdl_config SET value='10'   WHERE name='minpasswordlength';
UPDATE mdl_config SET value='5'    WHERE name='passwordreuselimit';
UPDATE mdl_config SET value='5'    WHERE name='lockoutthreshold';
UPDATE mdl_config SET value='1800' WHERE name='lockoutduration';
UPDATE mdl_config SET value='1800' WHERE name='lockoutwindow';

-- Authentification
UPDATE mdl_config SET value='manual' WHERE name='auth';
UPDATE mdl_config SET value='1'      WHERE name='authpreventaccountcreation';
UPDATE mdl_config SET value=''       WHERE name='registerauth';

-- Inscriptions
UPDATE mdl_config SET value='manual,cohort' WHERE name='enrol_plugins_enabled';
```

Suivi de :
```bash
/opt/alt/php83/usr/bin/php /home/lmsmoodle/public_html/moodle/admin/cli/purge_caches.php
```

---

## 4. ⚠️ Reste à faire (nécessite UI ou fourniture externe)

### 4.1 Politique de confidentialité (§ 5.6) — CNDP loi 09-08
**Pourquoi pas fait :** texte juridique à fournir par le client.
**Action attendue :**
- Récupérer le texte officiel de la politique de confidentialité (validé juridiquement)
- Moodle : `Administration du site > Utilisateurs > Confidentialité et politiques > Politiques du site > Nouvelle politique`
- Type : *Politique de confidentialité* | Public : *Tous* | Statut : *Actif*
- Acceptation obligatoire à la première connexion : Oui

### 4.2 SMTP (§ 5.7) — envoi des emails
**Pourquoi pas fait :** identifiants SMTP de production à fournir (provider transactionnel : Mailgun, SES, Brevo, OVH…).
**En attendant** : Moodle utilise le `mail()` PHP système — fonctionnel pour des tests internes mais risque de finir en spam pour les envois externes.
**Action attendue :**
- Configurer dans `Administration du site > Serveur > Email > Configuration de la messagerie sortante`
- Variables à renseigner : hôte SMTP, port, sécurité (TLS), user, mdp, adresse expéditeur

### 4.3 Rétention des logs (§ 5.8)
**Vérification automatique** : Moodle 4.x gère via tâche planifiée `core\task\delete_unconfirmed_users_task` et stores configurables.
**Action recommandée** :
- `Administration du site > Plugins > Stores de log > Standard log` : vérifier que la durée de rétention est ≥ 18 mois (idéalement 36 mois pour la certif).

### 4.4 Apparence — charte client (§ 5.9)
**Pourquoi pas fait :** charte graphique non encore fournie (logo + couleurs + police).
**Variables TBD** dans le `MOODLE_SETUP.md` :
- Logo client (PNG/SVG)
- Couleur primaire (hex)
- Couleur secondaire (hex)
- Police principale

**Action attendue** :
- `Administration du site > Apparence > Logos` : upload logo + favicon
- `Administration du site > Apparence > Thèmes > Boost` : couleurs + police + page d'accueil

### 4.5 Frontpage (§ 5.10) — choix de layout
**Pourquoi pas fait** : choix de mise en page à arbitrer (texte d'accueil à rédiger avec le client).
**Action attendue** :
- `Administration du site > Page d'accueil > Réglages` : sélectionner items + ordonner

### 4.6 Mot de passe administrateur initial
**À FAIRE EN PRIORITÉ après login** :
- `https://lms-moodle.preprod.io/moodle/login/`
- Connexion `admin / Admin@2026!`
- `Mon profil > Préférences > Changer le mot de passe`
- Choisir un mdp respectant la nouvelle politique (10 car., 1 chiffre, 1 majuscule, 1 minuscule, 1 spécial)

---

## 5. Vérifications post-application

```bash
mysql -u lmsmoodle_moodle -p... lmsmoodle_moodle -e "
  SELECT name, value FROM mdl_config WHERE name IN (
    'lang','country','timezone','langmenu',
    'enablecompletion','enableavailability','enablebadges',
    'passwordpolicy','minpasswordlength','passwordreuselimit',
    'lockoutthreshold','lockoutwindow','lockoutduration',
    'auth','authpreventaccountcreation','registerauth',
    'enrol_plugins_enabled'
  ) ORDER BY name;
"
```

Sortie attendue (=état actuel vérifié) :

| name | value |
|---|---|
| auth | manual |
| authpreventaccountcreation | 1 |
| country | MA |
| enableavailability | 1 |
| enablebadges | 0 |
| enablecompletion | 1 |
| enrol_plugins_enabled | manual,cohort |
| lang | fr |
| langmenu | 0 |
| lockoutduration | 1800 |
| lockoutthreshold | 5 |
| lockoutwindow | 1800 |
| minpasswordlength | 10 |
| passwordpolicy | 1 |
| passwordreuselimit | 5 |
| timezone | Africa/Casablanca |

---

## 6. Étapes suivantes

- [ ] Changer le mdp admin via UI (cf. § 4.6)
- [ ] Récupérer auprès du client : politique CNDP, charte graphique, SMTP prod
- [ ] **Phase D** — champs profil custom (`validation_module_1..5`, `etablissement`, `ville`, `certification_obtenue`)
- [ ] **Phase E** — rôles custom (`responsable_pedago`, `jury_member`)
- [ ] **Phase F** — arborescence catégories (Formation Enseignants > Modèles | Sessions)
- [ ] **Phase C** — installation des plugins requis

---

*Document rédigé le 22 mai 2026 — Suivi Sprint 1 (Phase B configuration globale)*
