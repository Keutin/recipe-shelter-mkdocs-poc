# 04 — Sécurité et RGPD

Le cahier des charges insiste : « **garantir la sécurité, la fiabilité et l'efficacité opérationnelle** ». Le jury teste la conscience des menaces classiques (OWASP) et la connaissance du RGPD.

---

## Q1. Quelles failles de sécurité avez-vous prises en compte ?

**Intention jury** — Connaissance de l'OWASP Top 10. Probablement la question d'ouverture sur la sécurité.

**Ossature de réponse** — Annoncer **par catégorie** :

| Faille OWASP | Comment Recipe Shelter s'en protège |
|---|---|
| **Injection SQL** | Requêtes paramétrées (`?` placeholders) avec mysql2, jamais de concaténation |
| **Auth cassée** | bcrypt pour les mots de passe, JWT signé, cookie HttpOnly, expiration |
| **Exposition données sensibles** | Mot de passe jamais retourné par l'API, HTTPS en prod, secrets en variables d'env |
| **XSS** | Cookie HttpOnly (le JWT n'est pas lisible par JS), Angular échappe par défaut le HTML |
| **CSRF** | Cookie `SameSite=Strict` (ou `Lax`), CORS configuré strictement |
| **Contrôle d'accès cassé** | Middleware d'autorisation par rôle, vérification ownership côté service |
| **Mauvaise config** | CORS whitelist, headers de sécurité, secrets non versionnés |
| **Désérialisation** | Validation via DTO, pas d'eval, JSON.parse seulement |
| **Composants vulnérables** | `npm audit` régulier (limite : pas automatisé en CI) |
| **Logging insuffisant** | Logs d'erreur et de modération (limite : pas de centralisation) |

**Pièges**
- ❌ Ne nommer que 2-3 failles → manque de profondeur.
- ❌ Réciter sans relier au code → le jury demandera un exemple.

---

## Q2. Comment vous protégez-vous contre le XSS ?

**Intention jury** — Cross-Site Scripting — top 3 des failles classiques.

**Ossature de réponse**

1. **Cookie HttpOnly** : même si du JS malveillant s'exécute (script injecté via un commentaire mal échappé), il **ne peut pas lire le cookie** d'auth → l'attaquant ne peut pas voler la session.
2. **Échappement automatique côté Angular** : `{{ recipe.description }}` est échappé par défaut, on n'utilise jamais `[innerHTML]` sur du contenu user-generated. Si on doit accepter du markdown, on le rend via une lib qui sanitize.
3. **CSP (Content Security Policy)** — header HTTP qui restreint les origines de scripts autorisées. *(Limite assumée : pas configuré dans le projet, à ajouter au reverse proxy en prod.)*
4. **Validation côté back** des champs textes : longueurs maximales, refus de caractères suspects sur des champs sensibles.

**Pièges**
- ❌ Dire « Angular protège tout seul » sans nuancer (`bypassSecurityTrustHtml` casse cette protection).

---

## Q3. Comment vous protégez-vous contre le CSRF ?

**Intention jury** — Cross-Site Request Forgery.

**Ossature de réponse**

1. **Mécanisme** : un site malveillant force le navigateur d'un utilisateur connecté à envoyer une requête à notre API ; le navigateur joint le cookie d'auth automatiquement.
2. **Protection 1 — `SameSite` sur le cookie** : `SameSite=Strict` empêche le navigateur d'envoyer le cookie sur des requêtes initiées par un autre site. **C'est la défense principale chez moi.**
3. **Protection 2 — CORS strict** : seul l'origine du front est en whitelist côté back ; une requête depuis un autre origine est bloquée par le navigateur (avant même d'atteindre le serveur).
4. **Protection 3 — Token CSRF** (double-submit cookie ou synchronizer token) — utile en complément si on a des opérations très sensibles. *(Pas implémenté ici, considéré comme redondant avec SameSite=Strict.)*

**Pièges**
- ❌ Confondre XSS et CSRF — ce sont deux familles distinctes.

---

## Q4. Vos mots de passe sont-ils bien protégés ?

**Intention jury** — Variante de la question bcrypt.

**Ossature de réponse**

1. **Stockage** : jamais en clair. Hashé avec **bcrypt** + sel aléatoire intégré (cost factor ≈ 10).
2. **Transport** : seulement sur HTTPS (assuré par l'hébergeur / reverse proxy en prod).
3. **Politique de mot de passe** côté inscription : longueur min, mix de caractères. *(Confirmer ce qui est imposé dans le DTO.)*
4. **Récupération** : par lien magique avec token unique expirant, jamais d'envoi du mot de passe en clair par email.
5. **Pas de log du mot de passe** : on ne le passe jamais à `console.log` ou dans une trace d'erreur.

**Pièges**
- ❌ Confondre hash et chiffrement.

---

## Q5. Où stockez-vous le JWT ? Pourquoi pas le localStorage ?

**Intention jury** — Question de débat classique. Le jury veut voir qu'Arthur connaît les deux options.

**Ossature de réponse**

1. **Cookie HttpOnly + Secure + SameSite** chez moi.
2. **Pourquoi pas localStorage** :
   - Accessible par JavaScript → tout XSS qui s'exécute peut voler le token.
   - Pas de protection automatique contre l'envoi cross-origin (mais ça peut se gérer manuellement).
3. **Pourquoi cookie HttpOnly** :
   - Le navigateur joint automatiquement le cookie, donc pas de code à écrire côté front pour ajouter le header `Authorization`.
   - Pas accessible par JS → résistant au XSS.
   - `SameSite` protège contre CSRF.
4. **Compromis** : moins facile à transporter dans un client non-navigateur (mobile native, API publique). Pour ce projet web ciblé navigateur, c'est le meilleur choix.

**Ancres** — [`_draft_adr/adr-003-jwt-cookie-httponly.md`](../soutenance/backend/adr/adr-003-jwt-cookie-httponly.md).

**Pièges**
- ❌ « localStorage c'est mal » → trop catégorique ; mieux : « localStorage est viable si l'app est mono-domaine et n'a aucun risque XSS connu, mais HttpOnly est plus défensif ».

---

## Q6. Qu'est-ce que HTTPS et est-il en place ?

**Intention jury** — Bases.

**Ossature de réponse**

1. **HTTPS = HTTP sur TLS** : chiffre tout l'échange (URL, headers, body, cookies) entre navigateur et serveur.
2. **Indispensable** pour transporter le cookie HttpOnly de manière sûre — sans HTTPS, le cookie circule en clair sur le réseau.
3. **Mise en place** : géré par l'hébergeur (Let's Encrypt, Cloudflare, reverse proxy nginx, …) → certif gratuit, renouvelé automatiquement.
4. **Headers complémentaires** : `Strict-Transport-Security` pour forcer HTTPS sur les visites suivantes.

**Ancres** — l'URL de prod (à confirmer) doit être en `https://`.

**Pièges**
- ❌ Confondre HTTPS et le hashage de mots de passe.

---

## Q7. Que se passe-t-il si quelqu'un essaie de tester plein de mots de passe ?

**Intention jury** — Brute force / rate limiting.

**Ossature de réponse**

1. **Actuel** : pas de rate limiting strict implémenté (limite assumée).
2. **bcrypt à cost 10** rend déjà chaque tentative coûteuse côté serveur (~100 ms) — limite naturelle à ~10 essais/sec/serveur.
3. **Ce que je rajouterais en prod** :
   - Middleware `express-rate-limit` (lib externe — j'accepterais l'entorse au from-scratch pour la sécurité, ou j'écrirais un middleware custom).
   - Lock temporaire du compte après N échecs.
   - CAPTCHA après plusieurs échecs (reCAPTCHA, hCaptcha).
   - Log des tentatives suspectes pour analyse.

**Pièges**
- ❌ Prétendre qu'il y a un rate limit s'il n'y en a pas.

---

## Q8. Comment l'utilisateur peut-il supprimer son compte ?

**Intention jury** — RGPD, droit à l'effacement (article 17 du RGPD).

**Ossature de réponse**

1. **Endpoint `DELETE /users/me`** *(à confirmer dans le code)* : permet à l'utilisateur de demander la suppression.
2. **Stratégie** :
   - **Anonymisation** des contributions (commentaires, recettes) — l'auteur devient `[supprimé]` mais le contenu reste utile à la communauté.
   - **Suppression dure** des données personnelles (email, nom, mot de passe hashé).
   - **Conservation** éventuelle d'un identifiant anonymisé pour intégrité référentielle.
3. **Si non implémenté** : « Pas encore implémenté, mais c'est une obligation RGPD. Mon approche serait l'anonymisation côté contributions + suppression dure côté données perso. »

**Pièges**
- ❌ « On garde tout pour des raisons d'audit » sans nuancer → non-conformité RGPD.

---

## Q9. Quelles données personnelles collectez-vous ?

**Intention jury** — Conscience RGPD, minimisation des données.

**Ossature de réponse**

1. **Données collectées** :
   - Email (pour la connexion et la récupération de mot de passe).
   - Pseudo (pour l'affichage public).
   - Mot de passe (hashé, pas en clair).
   - Contributions (recettes, commentaires) — données publiques par nature.
   - Date de création du compte (utile pour le support et la sécurité).
2. **Données non collectées** : pas de nom réel, pas d'adresse postale, pas de téléphone, pas de date de naissance, pas de tracking publicitaire.
3. **Principe de minimisation** : on ne demande que ce qui est nécessaire au fonctionnement.

**Pièges**
- ❌ Collecter des données qu'on n'utilise pas → non-conformité.

---

## Q10. Avez-vous une politique de confidentialité ? Un cookie banner ?

**Intention jury** — Conformité RGPD pratique.

**Ossature de réponse**

- **Politique de confidentialité** : *(à confirmer)* — devrait être présente comme page statique accessible depuis le footer.
- **Mentions légales** : idem.
- **Cookie banner** : si le site n'utilise que des **cookies essentiels** (cookie d'auth HttpOnly), un banner n'est pas obligatoire au sens strict. Si on rajoute des cookies analytics (Google Analytics, Matomo non anonymisé), il faut un consentement.
- **Position défendable** : « Le site n'utilise qu'un cookie d'authentification strictement nécessaire au fonctionnement — exempté de consentement selon la CNIL. Aucun tracker tiers. »

**Pièges**
- ❌ « Pas de RGPD parce que c'est un projet de cert » → faux, dès qu'il y a des utilisateurs réels, ça s'applique.

---

## Q11. Le soft-delete que vous utilisez est-il compatible avec le RGPD ?

**Intention jury** — Question fine. Le soft-delete (marquer `deleted_at`) conserve la donnée — donc en tension avec le droit à l'effacement.

**Ossature de réponse**

1. **Distinguer deux suppressions** :
   - **Suppression de contenu** par l'utilisateur (ex : « je supprime ma recette ») → soft-delete OK, le contenu est juste masqué, on peut le restaurer.
   - **Suppression de compte** (RGPD) → là, il faut **anonymiser les contributions** et **supprimer dur les données personnelles**, le soft-delete ne suffit pas.
2. **Le soft-delete par modération** (admin supprime une recette pour cause de spam) garde la donnée pour traçabilité légitime (audit).
3. **Limite assumée** : la distinction entre les deux n'est peut-être pas encore implémentée. Dans ce cas, le dire clairement et expliquer comment on traiterait.

**Ancres** — [`_draft_adr/adr-004-soft-delete-et-log-moderation.md`](../soutenance/backend/adr/adr-004-soft-delete-et-log-moderation.md).

**Pièges**
- ❌ Dire que le soft-delete suffit pour le RGPD → faux.

---

## Q12. Avez-vous prévu une gestion des sauvegardes ?

**Intention jury** — Continuité de service.

**Ossature de réponse**

1. **En dev local** : pas de backup automatique (limite).
2. **En prod** :
   - Backups quotidiens de la base MySQL (dump SQL) — souvent intégré au plan d'hébergement.
   - Conservation sur 7-30 jours.
   - Test de restauration périodique (sinon le backup ne vaut rien).
3. **Code** : la sauvegarde, c'est Git + le remote GitHub.
4. **Données utilisateurs uploadées** (images de recettes) : à backuper aussi, parfois oublié.

**Pièges**
- ❌ Inventer un système de backup.
