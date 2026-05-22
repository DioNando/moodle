# Suivi — Mise en place du cron Moodle

> **Date :** 22 mai 2026
> **Contexte :** Sprint 1 — configuration globale (suite installation)
> **Statut :** ✅ Cron opérationnel, tâches planifiées exécutées toutes les minutes

---

## 1. Pourquoi un cron pour Moodle

Moodle s'appuie sur un script `admin/cli/cron.php` pour de nombreuses opérations différées et planifiées :

- Envoi des **notifications email** (forums, devoirs, messages)
- Suivi des **achèvements d'activités** et des cours
- **Rapports planifiés** (Configurable Reports, ReportBuilder)
- **Sauvegardes automatiques** des cours
- Nettoyage : sessions expirées, fichiers temporaires, logs anciens
- Traitement des **tâches adhoc** (ex. import CSV utilisateurs en arrière-plan)

→ Recommandation officielle Moodle 4.x : **exécution chaque minute**.

---

## 2. Test manuel préalable

Avant de planifier, vérification que le script s'exécute sans erreur :

```bash
/opt/alt/php83/usr/bin/php /home/lmsmoodle/public_html/moodle/admin/cli/cron.php
```

Résultat :
```
Cron run completed correctly
Cron completed at 12:03:08 in 0.003248 seconds. Memory used: 67.9 Mo.
```
✅ OK

---

## 3. Création du cron via cPanel

cPanel → **Cron Jobs**

| Champ | Valeur |
|---|---|
| Common Settings | `Once Per Minute (* * * * *)` |
| Minute / Hour / Day / Month / Weekday | `*` / `*` / `*` / `*` / `*` |
| Command | `/opt/alt/php83/usr/bin/php /home/lmsmoodle/public_html/moodle/admin/cli/cron.php >/dev/null 2>&1` |
| Email | (vide pour éviter le spam — Moodle log en BDD) |

> Choix du binaire `/opt/alt/php83/usr/bin/php` (alt-php83) pour rester aligné avec le PHP qui sert le site web (mêmes extensions, notamment `sodium`).

> Redirection `>/dev/null 2>&1` : supprime stdout + stderr afin d'éviter de recevoir un mail à chaque minute. Les erreurs éventuelles restent visibles dans `mdl_task_log` côté Moodle.

---

## 4. Vérification de fonctionnement

### 4.1 Logs Moodle (table `mdl_task_log`)

```sql
SELECT FROM_UNIXTIME(timestart) AS start, classname, result
FROM mdl_task_log
ORDER BY id DESC
LIMIT 10;
```

Extrait observé après ~5 minutes :

| start | classname | result |
|---|---|---|
| 2026-05-22 12:09:01 | `workshopallocation_scheduled\task\cron_task` | 0 |
| 2026-05-22 12:09:01 | `tool_monitor\task\clean_events` | 0 |
| 2026-05-22 12:09:01 | `tool_messageinbound\task\pickup_task` | 0 |
| 2026-05-22 12:09:01 | `mod_workshop\task\cron_task` | 0 |
| 2026-05-22 12:09:01 | `mod_quiz\task\update_overdue_attempts` | 0 |
| 2026-05-22 12:09:01 | `mod_forum\task\cron_task` | 0 |
| 2026-05-22 12:09:01 | `mod_assign\task\cron_task` | 0 |
| 2026-05-22 12:09:01 | `core\task\automated_backup_report_task` | 0 |
| 2026-05-22 12:09:01 | `core_reportbuilder\task\send_schedules` | 0 |
| 2026-05-22 12:09:01 | `core\task\question_preview_cleanup_task` | 0 |

- `result = 0` : succès
- Total logs après quelques minutes : **207 lignes**

### 4.2 Config Moodle

```sql
SELECT name, value FROM mdl_config WHERE name = 'cron_enabled';
-- → cron_enabled = 1 ✅
```

---

## 5. Points d'attention pour la suite

- ⚠️ En préprod cPanel mutualisé, le cron peut être **soft-throttlé** par CloudLinux (LVE) si trop de tâches simultanées. À surveiller en cas de blocage ou de runs > 60 s.
- 🔄 Le warning *"Cron has not been run in the last hour"* doit avoir disparu de l'interface admin Moodle (Site administration → Notifications). À vérifier après le prochain login.
- 📌 En **production** (VM Ubuntu 22.04 dédiée) : on basculera sur un cron système (`/etc/cron.d/moodle`) exécuté en `www-data`, identique en logique.

---

## 6. Étapes suivantes

- [ ] Login admin web (`http://lms-moodle.preprod.io/moodle/login/`) → changer le mot de passe initial
- [ ] Configurer HTTPS (Let's Encrypt cPanel) + mettre à jour `$CFG->wwwroot`
- [ ] Phase B — configuration globale Moodle (pays MA, fuseau Africa/Casablanca, politique mdp, désactiver auto-inscription)
- [ ] Phase D — champs profil custom (`validation_module_1..5`, `etablissement`, `ville`, `certification_obtenue`)
- [ ] Phase E — rôles custom (`responsable_pedago`, `jury_member`)
- [ ] Phase F — arborescence catégories (Formation Enseignants > Modèles | Sessions)
- [ ] Phase C — installation des plugins requis

---

*Document rédigé le 22 mai 2026 — Suivi Sprint 1 (config globale, cron)*
