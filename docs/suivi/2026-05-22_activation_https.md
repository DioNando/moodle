# Suivi — Activation HTTPS (Let's Encrypt)

> **Date :** 22 mai 2026
> **Contexte :** Sprint 1 — sécurisation avant configuration UI
> **Statut :** ✅ HTTPS opérationnel, certificat Let's Encrypt valide, redirection forcée HTTP → HTTPS

---

## 1. Contexte

Avant d'attaquer la configuration globale Moodle (Phase B) et toute saisie d'identifiants admin via l'UI, il est indispensable de passer la plateforme en HTTPS. Cela évite aussi de devoir modifier `$CFG->wwwroot` deux fois.

État initial : cPanel servait un certificat **auto-signé** (subject = issuer = `lms-moodle.preprod.io`) — les navigateurs auraient affiché une alerte de sécurité.

---

## 2. Activation AutoSSL (Let's Encrypt) côté cPanel

Action utilisateur :

1. cPanel → **SSL/TLS Status** (ou **AutoSSL**)
2. Cocher `lms-moodle.preprod.io` si non inclus
3. Cliquer **Run AutoSSL**
4. Attente : 1–3 minutes pour la validation HTTP-01 + installation

Vérification du certificat :
```bash
echo | openssl s_client -servername lms-moodle.preprod.io \
  -connect lms-moodle.preprod.io:443 2>/dev/null \
  | openssl x509 -noout -subject -issuer -dates
```

Résultat :
```
subject=CN=lms-moodle.preprod.io
issuer=C=US, O=Let's Encrypt, CN=R12
notBefore=May 22 10:14:50 2026 GMT
notAfter=Aug 20 10:14:49 2026 GMT
```
✅ Certificat valide ~90 jours. AutoSSL renouvelle automatiquement.

---

## 3. Mise à jour de Moodle

### 3.1 `config.php`
```diff
- $CFG->wwwroot   = 'http://lms-moodle.preprod.io/moodle';
+ $CFG->wwwroot   = 'https://lms-moodle.preprod.io/moodle';
+ $CFG->sslproxy  = false;
```

> `sslproxy = false` car Nginx termine la connexion TLS et passe directement la requête au backend PHP (pas de reverse proxy SSL devant).

### 3.2 Redirection HTTP → HTTPS dans `.htaccess`
Bloc ajouté en tête (avant la redirection racine → `/moodle/`) :

```apache
# Forcer HTTPS sur toutes les URLs
<IfModule mod_rewrite.c>
  RewriteEngine On
  RewriteCond %{HTTPS} !=on
  RewriteCond %{HTTP:X-Forwarded-Proto} !=https
  RewriteRule ^(.*)$ https://%{HTTP_HOST}%{REQUEST_URI} [R=301,L]
</IfModule>
```

- Test sur `%{HTTPS}` ET `X-Forwarded-Proto` : robuste si un proxy/CDN est ajouté plus tard
- Redirection **301** (permanente) : indexée correctement par les moteurs et cachée par les navigateurs

### 3.3 Purge des caches Moodle
```bash
/opt/alt/php83/usr/bin/php /home/lmsmoodle/public_html/moodle/admin/cli/purge_caches.php
```

---

## 4. Vérifications

| Test | Commande | Résultat attendu | Résultat obtenu |
|---|---|---|---|
| HTTP racine → HTTPS | `curl -sI http://lms-moodle.preprod.io/moodle/` | 301 + Location https | ✅ 301 → https://.../moodle/ |
| HTTPS racine → /moodle/ | `curl -sIk https://lms-moodle.preprod.io/` | 302 + Location /moodle/ | ✅ 302 → https://.../moodle/ |
| Page login | `curl -sIk https://lms-moodle.preprod.io/moodle/login/index.php` | 200 OK | ✅ 200 OK |
| Certificat | `openssl s_client` | Let's Encrypt valide | ✅ R12, exp. 20/08/2026 |
| PHP servant | `_c.php` (test temporaire) | alt-php83 + sodium | ✅ /opt/alt/php83/usr/bin/php-cgi, sodium OK |

---

## 5. Points d'attention

- ⚠️ **Renouvellement** : AutoSSL renouvelle automatiquement le certificat ~30 j avant expiration. À surveiller dans cPanel → SSL/TLS Status une fois par trimestre.
- 📌 **Contenu mixte** : si après login on observe des warnings "Mixed Content" dans la console navigateur, il faudra `replace` les URLs `http://` en BDD dans les contenus existants. Comme le site est tout neuf, peu de risques.
- 🔒 **HSTS** : non activé pour l'instant (préprod). À ajouter en prod via header `Strict-Transport-Security: max-age=31536000; includeSubDomains` (cPanel → "Indexes & Redirects" ou .htaccess).
- 🔑 **Handler PHP** : la directive `AddHandler ea-php84` dans `.htaccess` est cosmétique (gérée par cPanel) — le PHP Selector CloudLinux route effectivement vers `alt-php83`. Vérification refaite : `/opt/alt/php83/usr/bin/php-cgi` sert le site, sodium OK.

---

## 6. Étapes suivantes

- [ ] Login admin (`https://lms-moodle.preprod.io/moodle/login/`) → changer le mot de passe initial `Admin@2026!`
- [ ] Phase B — configuration globale Moodle (pays MA, fuseau Africa/Casablanca, politique mdp, désactiver auto-inscription, langue par défaut FR)
- [ ] Phase D — champs profil custom (`validation_module_1..5`, `etablissement`, `ville`, `certification_obtenue`)
- [ ] Phase E — rôles custom (`responsable_pedago`, `jury_member`)
- [ ] Phase F — arborescence catégories (Formation Enseignants > Modèles | Sessions)
- [ ] Phase C — installation des plugins requis (BBB, Attendance, Customcert, etc.)

---

*Document rédigé le 22 mai 2026 — Suivi Sprint 1 (sécurisation HTTPS)*
