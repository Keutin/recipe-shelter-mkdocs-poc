# Sécurité — Recipe Shelter (Backend)

> Document de référence pour la soutenance RNCP.
> Toutes les affirmations sont sourcées sur le code (`fichier:ligne`).
> Section finale : "Améliorations identifiées" — honnêteté pédagogique sur ce qui n'a PAS été implémenté.

---

## 1. Vue d'ensemble

### 1.1 Périmètre de l'application
Recipe Shelter est une application web composée d'une API REST (Express + TypeScript), d'un front Angular et d'une base de données MySQL. Le backend expose `/api/v1/*` (voir `backend/src/app.ts:133-147`). L'authentification est portée par un cookie de session (JWT signé) déposé par le serveur.

### 1.2 Surface d'attaque couverte
- Authentification (inscription, login, logout, reset password, email validation)
- Autorisation (utilisateur authentifié / administrateur)
- Persistance des mots de passe (hash bcrypt)
- Communication HTTP (CORS strict, cookie HttpOnly)
- Brute-force credentials (rate limiting sur les routes sensibles)
- Injection SQL (mysql2 + placeholders paramétrés)
- Tokens à usage unique (reset mot de passe, validation email)
- Validation des entrées au boundary HTTP (DTOs explicites)
- Gestion d'erreurs maîtrisée (pas de stack trace renvoyée au client)

### 1.3 Périmètre EXPLICITEMENT hors scope (assumé)
Choix pédagogique : le projet est "from scratch" et vise à démontrer la maîtrise des fondamentaux. Les éléments ci-dessous, bien que recommandés en production, ne font pas partie du scope :

- **WAF** (Web Application Firewall) → mutualisé en infra (Cloudflare, AWS WAF…), pas géré applicativement.
- **Protection DDoS** → mutualisée au niveau infra/CDN.
- **IDS/IPS** → relève de la couche système/réseau.
- **Audit logs centralisés** → un logger applicatif existe (`backend/src/utils/logger.ts`) mais pas d'envoi vers SIEM.
- **Rotation automatique des secrets** → faite manuellement via variables d'environnement.
- **Scan de dépendances en CI** → audit manuel via `npm audit`.

---

## 2. Authentification

### 2.1 Hash des mots de passe
Algorithme **bcrypt**, coût par défaut **12** (configurable via `BCRYPT_COST`).

- Configuration : `backend/src/utils/env.ts:98` → `bcryptCost: readNumber(process.env.BCRYPT_COST, 12)`
- Hash au register : `backend/src/services/auth/auth.service.ts:63` → `bcrypt.hash(password, env.auth.bcryptCost)`
- Vérification au login : `backend/src/services/auth/auth.service.ts:81` → `bcrypt.compare(password, authUser.passwordHash)`
- Hash lors du reset : `backend/src/services/auth/password-reset.service.ts:64` → `bcrypt.hash(newPassword, env.auth.bcryptCost)`

**Pourquoi bcrypt** : algorithme à fonction de hachage lent (résistant aux attaques GPU), salt intégré au hash, coût configurable. Coût 12 ≈ 250 ms/hash sur matériel courant — bon compromis sécurité/latence.

**Note** : `backend/src/services/users/users.service.ts:112` utilise une valeur littérale `12` au lieu de `env.auth.bcryptCost` lors de l'update de mot de passe. À aligner sur l'env (voir section "Améliorations").

### 2.2 Politique de mot de passe
Fichier : `backend/src/services/auth/password-policy.ts`

```
- Chaîne non vide (ligne 2-3)
- Longueur minimale : 8 caractères (ligne 5-6)
- Longueur maximale : 128 caractères (ligne 8-9)
```

**Justification du choix** : la politique suit les recommandations NIST SP 800-63B (longueur > complexité). Pas d'obligation de caractères spéciaux/majuscules qui pousse les utilisateurs à des mots de passe prévisibles. La longueur max protège contre les attaques DoS par bcrypt (bcrypt tronque à 72 octets, mais la borne 128 évite le coût de hash sur des entrées géantes).

Application :
- Register : `auth.service.ts:47-49` (jette `AUTH_WEAK_PASSWORD`)
- Reset : `password-reset.service.ts:54-56`
- Update : `users.service.ts:107-110`

### 2.3 JWT
Bibliothèque : `jsonwebtoken@^9.0.3` (`backend/package.json:46`).

- **Algorithme de signature** : `HS256` (algo par défaut de `jwt.sign` quand un secret string est fourni).
- **Secret** : `JWT_SECRET` exigé via env (`env.ts:90` lève une erreur si absent — pas de défaut secret).
- **Claims** (`auth.service.ts:14-19`) :
  - `sub` (user id, number)
  - `username` (string)
  - `roleId` (number)
  - `status` (`'inactive' | 'active' | 'banned'`)
- **Durée de vie** : `JWT_EXPIRES_IN`, défaut **7 jours** (`env.ts:68`).
- **Vérification** : `require-auth.ts:76` → `jwt.verify(token, env.auth.jwtSecret)` (catch global lève `unauthorized`).
- **Garde-fou statut** : `require-auth.ts:57-67` — après vérification JWT, on recharge le user en base et on rejette si `status !== 'active'`. Empêche un token volé d'un compte banni de rester valide.

---

## 3. Session / Cookie

### 3.1 Nom et flags
Fichier : `backend/src/utils/session-cookie.ts`

| Flag       | Valeur                                                    | Source                       |
|------------|-----------------------------------------------------------|------------------------------|
| Nom        | `rs_session` (configurable `AUTH_SESSION_COOKIE_NAME`)    | `env.ts:92`                  |
| `httpOnly` | `true`                                                    | `session-cookie.ts:10`       |
| `path`     | `/`                                                       | `session-cookie.ts:11`       |
| `sameSite` | `'lax'` par défaut (configurable)                         | `env.ts:94`, `session-cookie.ts:12` |
| `secure`   | `true` en production, `false` en dev                      | `env.ts:95`                  |
| `domain`   | optionnel, non posé par défaut                            | `session-cookie.ts:15-16`    |
| `maxAge`   | 7 jours par défaut (aligné sur JWT_EXPIRES_IN)            | `env.ts:96`, `session-cookie.ts:24` |

### 3.2 Pourquoi un cookie HttpOnly plutôt que localStorage
- **Anti-XSS** : le flag `HttpOnly` empêche JavaScript d'accéder au cookie. Une faille XSS sur le front ne permet pas d'exfiltrer le token.
- **`localStorage`** est lisible par tout script s'exécutant dans la page (extensions, librairies tierces compromises, XSS) — vecteur de vol de session trivial.
- **`SameSite=Lax`** réduit fortement la surface CSRF (voir §6) tout en restant compatible avec la navigation classique (lien externe vers le site).
- **`Secure`** en production garantit que le cookie n'est jamais envoyé en clair sur HTTP.

### 3.3 Déconnexion
`clearSessionCookie` (`session-cookie.ts:36-38`) utilise les mêmes options (sans `maxAge`) que la pose, condition `clearCookie` d'Express pour effacer correctement le cookie côté navigateur.

---

## 4. Autorisation

### 4.1 `requireAuth`
Fichier : `backend/src/middlewares/require-auth.ts`

Pipeline :
1. Lit le cookie de session (`getSessionToken`, ligne 30-34).
2. Si absent → `401 AUTH_NO_TOKEN`.
3. Vérifie la signature JWT (`jwt.verify`, ligne 76).
4. Parse les claims (`parseAuthPayload`, ligne 36-55) : rejette si `sub`, `roleId` ou `username` invalides.
5. Recharge l'utilisateur en base (`resolveActiveAuth`, ligne 57-67) : rejette si user introuvable ou `status !== 'active'`.
6. Attache `req.auth = { userId, username, roleId, status }`.

**Variante** : `optionalAuth` (ligne 94-117) — même logique mais ne bloque jamais ; utilisée pour les routes publiques qui adaptent leur réponse selon que le visiteur est connecté ou non (ex. flag "is owned" sur les recettes).

### 4.2 `requireAdmin`
Fichier : `backend/src/middlewares/require-admin.ts` (10 lignes).

```ts
if (req.auth?.roleId !== 1)
    return next(forbidden('Admin access required', 'ADMIN_ACCESS_REQUIRED'));
```

**Conventions** :
- `roleId === 1` = administrateur (rôle seedé en base, voir `backend/database/`).
- DOIT être chaîné APRÈS `requireAuth` — `req.auth` n'existe que si `requireAuth` est passé.

### 4.3 Ownership checks
La vérification "le user agit sur SES propres ressources" se fait dans les services métier (ex. `comments.service.ts:38-47` `deleteComment(id, userId)` — le repository ne supprime que si `UserId` matche). Pattern à généraliser (voir Améliorations).

### 4.4 Ordre dans le pipeline
Voir document `_draft_backend_docs/diagrams/g2-pipeline-middleware.md` (à produire). Ordre Express type :

```
cors → cookieParser → express.json → router → [requireAuth → requireAdmin] → controller → errorHandler
```

Sources : `app.ts:80-82` (middlewares globaux), `auth.routes.ts:24` (rate limiter spécifique).

---

## 5. CORS

Fichier : `backend/src/app.ts:72-80`

```ts
const origins = env.http.corsAllowedOrigins
  .split(',').map((o) => o.trim()).filter(Boolean);

if (origins.includes('*'))
  throw new Error('CORS_ALLOWED_ORIGINS must list explicit origins when credentials are enabled');

app.use(cors({ credentials: true, origin: origins }));
```

**Points clés** :
- **Origines explicites** : liste blanche d'origines (`CORS_ALLOWED_ORIGINS`, défaut `http://localhost:4200,http://127.0.0.1:4200` en `env.ts:76`).
- **`credentials: true`** : autorise l'envoi du cookie de session cross-origin pour le front Angular.
- **Garde-fou wildcard** (`app.ts:77-78`) : refuse explicitement `*` dans la liste. C'est INTERDIT par la spec CORS quand `credentials: true` est posé — la lib `cors` n'enforce pas, nous le faisons au démarrage de l'app. **Cette protection est CRITIQUE** : un `*` avec `credentials: true` permettrait à n'importe quelle origine de lancer des requêtes authentifiées.

---

## 6. CSRF

### 6.1 Défense en profondeur
Pas de token CSRF explicite. La protection repose sur :

1. **`SameSite=Lax`** sur le cookie (`session-cookie.ts:12`) : le navigateur n'envoie PAS le cookie sur les requêtes cross-site déclenchées par des balises form ou XHR depuis un autre site (sauf navigation top-level GET).
2. **`HttpOnly`** : un attaquant ne peut pas lire le cookie via XSS pour le rejouer.
3. **CORS strict** (`credentials: true` + liste blanche d'origines, §5) : un XHR cross-site depuis une origine non listée est bloqué par le navigateur AVANT que la requête atteigne le code applicatif.

### 6.2 Pourquoi pas de token CSRF ?
- Avec `SameSite=Lax`, l'attaque CSRF classique (form POST cross-site) est neutralisée par le navigateur.
- Le coût d'implémentation d'un double-submit token apporte une protection marginale dans ce contexte (pas de support de très vieux navigateurs).
- En revanche, **un compromis XSS du frontend resterait fatal** — peu importe le token CSRF puisque l'attaquant pourrait le récupérer. La priorité est donc anti-XSS (CSP côté front, échappement Angular natif, validation server-side).

### 6.3 Limite assumée
Si la politique passait à `SameSite=None` (ex. front sur domaine distinct sans proxy), il faudrait introduire un token CSRF (cf. Améliorations).

---

## 7. Validation des entrées

### 7.1 Helpers de DTO
Fichier : `backend/src/api/http/dto.helpers.ts`

Helpers stricts, lève `badRequest` (400) en cas d'invalidité :

| Helper                              | Garantie                                                  |
|-------------------------------------|-----------------------------------------------------------|
| `getRequiredString` (ligne 7-14)    | string non vide après `trim`                              |
| `getOptionalString` (ligne 16-24)   | undefined OU string                                       |
| `getOptionalNullableString` (26-37) | undefined / null / string                                 |
| `getOptionalNumber` (39-47)         | undefined / null / number fini                            |
| `getRequiredNumber` (62-67)         | number fini                                               |
| `getRequiredPositiveInteger` (69-74)| integer > 0                                               |
| `getOptionalArray` (76-84)          | undefined / null / tableau (parseur par item)             |

### 7.2 Rejets au boundary
Toute violation se traduit par une erreur applicative `badRequest` (code 400), levée AVANT que la donnée atteigne le service ou le repository. Pattern systématique dans les controllers.

### 7.3 Validation email spécifique
`users.service.ts:65-68` : regex simple `/^[^\s@]+@[^\s@]+\.[^\s@]+$/` pour l'update email. Volontairement minimale (la validation finale = envoi d'un mail de confirmation, voir §10).

### 7.4 Normalisation
- Email systématiquement normalisé (lowercase + trim) via `normalizeEmail` (`utils/string.ts`). Évite le bypass d'unicité (`Alice@x.com` vs `alice@x.com`).
- Username `trim()` + longueur min 3 (`auth.service.ts:54-55`).

---

## 8. Rate limiting

### 8.1 Implémentation
Fichier : `backend/src/middlewares/rate-limiter.ts`

- Store en mémoire (`Map<string, { count, resetAt }>`, ligne 3).
- Clé : `IP + method + baseUrl + path` (ligne 5-12).
- Renvoie `429 RATE_LIMIT` + headers `RateLimit-*` + `Retry-After` quand `count >= max`.
- Fenêtre glissante par "bucket" : à expiration, le compteur est réinitialisé.

### 8.2 Application
Routes protégées :

| Route                                | Limite             | Fenêtre        | Source                  |
|--------------------------------------|--------------------|----------------|-------------------------|
| `POST /auth/register`                | 5 (configurable)   | 15 min         | `auth.routes.ts:24`     |
| `POST /auth/login`                   | 5                  | 15 min         | `auth.routes.ts:25`     |
| `POST /auth/validate-email`          | 5                  | 15 min         | `auth.routes.ts:28`     |
| `POST /auth/resend-validation-email` | 5                  | 15 min         | `auth.routes.ts:29`     |
| `POST /auth/forgot-password`         | 5                  | 15 min         | `auth.routes.ts:30`     |
| `POST /contact`                      | 5                  | 1 h            | `contact.router.ts:11-16` |

Configuration : `AUTH_RATE_LIMIT_MAX_ATTEMPTS` (défaut 5) et `AUTH_RATE_LIMIT_WINDOW_MS` (défaut 900000 = 15 min), `env.ts:99-100`.

### 8.3 Limites assumées
- **Store mémoire** : non partagé entre instances ⇒ ne tient pas l'horizontal scaling. À porter sur Redis pour une vraie prod (voir Améliorations).
- **Clé IP** : peut être contournée par rotation d'IP (botnets). Combinaison avec une politique de mot de passe forte et bcrypt coût 12 reste défensive.
- **`POST /auth/reset-password`** n'est PAS rate-limité (`auth.routes.ts:31`). Argument : la connaissance du token (32 octets aléatoires) suffit comme protection ; mais ajouter un rate limit est trivial et conseillé (cf. Améliorations).

---

## 9. Headers de sécurité HTTP

**État actuel** : aucun middleware `helmet` n'est utilisé (`backend/package.json` n'inclut PAS `helmet`). Aucun header CSP, HSTS, X-Frame-Options, X-Content-Type-Options n'est posé par l'application.

**Conséquence** : les protections classiques contre clickjacking, MIME-sniffing, mixed content ne sont pas actives au niveau applicatif. En production, ces headers DOIVENT être posés par le reverse proxy (Nginx, Cloudflare) OU par l'ajout de `helmet` à Express.

Voir Améliorations §14.

---

## 10. Emails & tokens à usage unique

### 10.1 Génération de token
Fichier : `backend/src/utils/security/password-reset-token.ts`

```ts
generateResetToken() → crypto.randomBytes(32).toString('hex')   // 256 bits
hashResetToken(t)    → crypto.createHash('sha256').update(t).digest('hex')
```

**Propriétés** :
- 32 octets aléatoires (CSPRNG `crypto.randomBytes`) → entropie 256 bits, impraticable à brute-forcer.
- **Le token brut n'est JAMAIS stocké en base** : seul son SHA-256 l'est. Une fuite de la table `PasswordResets` ne révèle pas les tokens utilisables (le hash SHA-256 ne se "renverse" pas en pratique pour 256 bits aléatoires).
- Le hash SHA-256 est suffisant car le token est à très haute entropie (pas besoin de bcrypt sur du random 256 bits, où l'attaque par dictionnaire n'a aucun sens).

### 10.2 Reset password
Fichier : `backend/src/services/auth/password-reset.service.ts`

- **TTL** : 30 minutes (`PASSWORD_RESET_TTL_MINUTES`, ligne 22).
- **Invalidation préalable** : `resets.invalidateAllForUser(user.id)` (ligne 37) — chaque demande invalide les tokens précédents.
- **Énumération utilisateurs neutralisée** : `requestReset` renvoie silencieusement si l'email n'existe pas (ligne 32-34) ⇒ pas de leak "ce compte existe".
- **Usage unique** : `markUsed(reset.Id)` (ligne 67) après update du password.
- **Notification** : email "password changed" envoyé après changement (ligne 71) — alerte l'utilisateur en cas d'usage malveillant.

### 10.3 Validation email
Fichier : `backend/src/services/auth/email-validation.service.ts`

- Même primitive de token (réutilise `generateResetToken` / `hashResetToken`).
- **TTL** : 24 heures (`EMAIL_VALIDATION_TTL_MINUTES = 24 * 60`, ligne 15).
- **Compte inactif jusqu'à validation** : `register` crée le compte avec `status: 'inactive'` (`auth.service.ts:64`), et `login` refuse `EMAIL_NOT_VALIDATED` tant que non validé (`auth.service.ts:85-86`).
- **Resend** : possible seulement si `status === 'inactive'` (`email-validation.service.ts:35-36`), évite l'abus pour utilisateurs déjà actifs.
- **Pas de leak d'énumération** : `resendValidationEmail` renvoie silencieusement si email inconnu (ligne 32-33).

---

## 11. Persistance — Anti SQL Injection

### 11.1 Pattern systématique
mysql2 avec **placeholders `?`** sur toutes les requêtes paramétrées. Exemple type :

```ts
await this.db.execute(
  `UPDATE Recipes SET Slug = ?, ... WHERE Id = ?`,
  [slug, id]
);
```

Source : `backend/src/repositories/recipes/recipe.repository.mysql.ts:145-149`.

### 11.2 Cas d'interpolation contrôlée
Certains repos construisent dynamiquement des fragments SQL (clauses LIMIT/OFFSET, listes de placeholders, SET clause). Audit :

| Fichier:ligne                                          | Contenu interpolé                                  | Vecteur SQLi ? |
|--------------------------------------------------------|----------------------------------------------------|----------------|
| `recipe.repository.mysql.ts:110`                       | `${updateFields.join(', ')}` — `'Field = ?'`       | NON — strings construites en code, jamais de user input |
| `recipe.repository.mysql.ts:207,225,235,237,270`       | `${limitOffsetClause}` / `${where.clause}`         | NON — fragments générés en interne, valeurs en `?` |
| `recipe.repository.mysql.ts:320`                       | `\`%${filters.q}%\``                               | NON — c'est une VALEUR poussée dans `params[]`, jouée en `?` |
| `recipe.repository.mysql.ts:333,345`                   | `${placeholders}` (suite de `?,?,?`)               | NON — chaîne fixe de marqueurs `?` |
| `favorites.repository.mysql.ts:76`                     | `${limitOffsetClause}`                             | NON — interne |
| `admin.users.repository.mysql.ts:64`                   | `LIMIT ${USER_MODERATION_LOGS_LIMIT}`              | NON — constante littérale |

**Conclusion** : aucune interpolation de user input dans une chaîne SQL. Toutes les valeurs utilisateur passent par les placeholders `?` (paramètres préparés). **Protection SQLi : OK.**

### 11.3 Transaction et isolation
Le repository recettes utilise `connection.commit()` / `rollback()` (`recipe.repository.mysql.ts:128,137`) pour garantir l'atomicité des updates multi-tables (recipe + ingredients + steps + equipments + tags). Pool de connexions configuré via `DB_CONNECTION_LIMIT` (`env.ts:86`, défaut 10).

---

## 12. Mapping OWASP Top 10 (2021)

| ID  | Catégorie                              | Statut dans Recipe Shelter | Évidence                                                    |
|-----|----------------------------------------|----------------------------|-------------------------------------------------------------|
| A01 | Broken Access Control                  | **Couvert**                | `requireAuth` + `requireAdmin` (§4) ; ownership checks dans services |
| A02 | Cryptographic Failures                 | **Couvert**                | bcrypt coût 12, JWT HS256, cookie `Secure` en prod, tokens SHA-256 256 bits (§2, §10) |
| A03 | Injection                              | **Couvert**                | mysql2 placeholders `?`, DTO validation au boundary (§7, §11) |
| A04 | Insecure Design                        | **Couvert**                | Pattern Repository + couches (controller→service→repo), garde-fou statut user en plus du JWT, énumération email neutralisée |
| A05 | Security Misconfiguration              | **Partiel**                | `JWT_SECRET` requis (pas de défaut), CORS sans wildcard, error-handler ne fuit pas la stack ; MAIS pas de helmet (§9) |
| A06 | Vulnerable & Outdated Components       | **Manuel**                 | Audit `npm audit` ponctuel, pas de Dependabot/Renovate (cf. Améliorations) |
| A07 | Identification & Authentication Failures | **Couvert**              | bcrypt + politique mdp + rate limiting login/forgot + email validation obligatoire (§2, §8, §10) |
| A08 | Software & Data Integrity Failures     | **Couvert**                | JWT signé (HS256), pas d'`eval`, pas de désérialisation untrusted, package-lock.json présent |
| A09 | Security Logging & Monitoring Failures | **Partiel**                | Logger applicatif (`utils/logger.ts`) ; pas d'envoi SIEM, pas d'alerting (cf. Améliorations) |
| A10 | Server-Side Request Forgery (SSRF)     | **N/A**                    | L'application ne fait aucun appel HTTP sortant depuis un input utilisateur. Le seul flux sortant est SMTP (mail), destinataire = email du user destinataire d'un email transactionnel uniquement |

---

## 13. RGPD

### 13.1 Données personnelles collectées
| Donnée            | Finalité                                              | Base légale          |
|-------------------|-------------------------------------------------------|----------------------|
| Email             | Identification, communication transactionnelle        | Exécution du contrat |
| Username          | Identité publique sur la plateforme                   | Exécution du contrat |
| Hash mot de passe | Authentification                                      | Exécution du contrat |
| Recettes publiées | Contenu utilisateur (volontairement publié)           | Exécution du contrat |
| Commentaires      | Interactions sur les recettes                         | Exécution du contrat |
| Favoris           | Personnalisation                                      | Exécution du contrat |

**Pas de cookies traceurs**, pas de tracking publicitaire, pas de profilage.

### 13.2 Droits utilisateurs

| Droit                         | Implémenté ?     | Source                                                |
|-------------------------------|------------------|-------------------------------------------------------|
| Accès (lire ses données)      | Oui              | `GET /users/me` (`users.service.ts:28-42`)            |
| Rectification (email/username)| Oui              | `users.service.ts:59-91, 117-147`                     |
| Modification mot de passe     | Oui              | `users.service.ts:93-115`                             |
| **Suppression du compte**     | **Non implémenté** | Aucune méthode `deleteAccount` dans `UserService`. **Trou RGPD à combler** (cf. Améliorations) |
| Bannissement par admin        | Oui (modération) | `admin.users.service.ts:61-72`                        |
| Portabilité (export)          | Non              | Non implémenté                                        |

### 13.3 Conservation
- Mots de passe : conservés sous forme bcrypt (irréversible).
- Tokens de reset/validation : table dédiée, `markUsed` après usage, expiration automatique (30 min / 24 h).
- Pas de purge automatique des tokens expirés ⇒ croissance de la table. **À prévoir un cron de purge** (cf. Améliorations).

### 13.4 Consentement
À documenter côté frontend : checkbox "j'accepte les CGU" à l'inscription. La présence est à vérifier dans `frontend/src/app/.../signup` (hors scope de ce document backend).

---

## 14. Améliorations identifiées (transparence pédagogique)

Choix assumé : prioriser la maîtrise des fondamentaux plutôt que la couverture exhaustive. Voici ce qui MANQUE et serait ajouté en V2 :

### 14.1 Headers de sécurité
- **Ajouter `helmet`** dans `app.ts` (`app.use(helmet())`) pour poser : HSTS, X-Frame-Options, X-Content-Type-Options, Referrer-Policy.
- **Content-Security-Policy** stricte (script-src 'self', object-src 'none', etc.) — à coordonner avec le front Angular.

### 14.2 Authentification / Session
- **Rotation des JWT** : actuellement durée fixe 7 jours. Introduire un refresh token + access token court (15 min) pour limiter la fenêtre d'usage d'un token volé.
- **Révocation côté serveur** : pas de blacklist JWT (cohérent avec le design stateless). En cas de compromission, seul `status` (rechargé à chaque requête) permet d'invalider — OK mais pas instantané si on cachait l'utilisateur.
- **Rate limit sur `/auth/reset-password`** : actuellement non rate-limité.
- **Politique mot de passe** : envisager un check contre la liste HaveIBeenPwned (k-anonymity API).
- **2FA / TOTP** : non implémenté.
- **Aligner `users.service.ts:112`** sur `env.auth.bcryptCost` (actuellement valeur littérale `12`).

### 14.3 Rate limiter
- **Externaliser sur Redis** pour supporter l'horizontal scaling.
- **Combiner clé IP + clé email** pour bloquer aussi les attaques distribuées sur un même compte.

### 14.4 CSRF
- Si jamais le cookie devait passer en `SameSite=None` (front sur domaine distinct sans proxy), introduire un **token CSRF double-submit**.

### 14.5 Logs et monitoring
- **Audit log centralisé** des actions sensibles (login échec, ban, reset password, suppression de compte) avec corrélation `request-id`.
- **Envoi vers un SIEM** (ELK, Datadog) pour alerting.
- **Alerting sur seuils** (ex. > 100 logins échoués/min ⇒ alerte).

### 14.6 Dépendances
- **Renovate / Dependabot** sur le repo.
- **`npm audit` en CI** avec seuil `--audit-level=high`.

### 14.7 RGPD
- **Implémenter `DELETE /users/me`** avec choix anonymisation (recettes/commentaires conservés sous "Utilisateur supprimé") OU suppression complète.
- **Export des données** (JSON) à la demande.
- **Cron de purge** des tokens email/reset expirés.

### 14.8 Tests de sécurité
- **Tests automatisés** sur les chemins d'autorisation (un user A ne peut pas modifier la recette d'un user B → couvrir tous les endpoints).
- **Scan SAST** (Semgrep, CodeQL) en CI.
- **Scan DAST** (OWASP ZAP) en pre-prod.

### 14.9 Infrastructure (hors scope applicatif)
- WAF (Cloudflare, AWS WAF).
- Protection DDoS au CDN.
- HSTS preload list.
- Certificat TLS via Let's Encrypt avec renouvellement auto.

---

## 15. Récapitulatif défensif (pour le jury)

> "Ce projet implémente les fondamentaux de la sécurité applicative : authentification robuste (bcrypt 12, politique de mot de passe, JWT signé, cookie HttpOnly+Secure+SameSite=Lax), autorisation à deux niveaux (`requireAuth`/`requireAdmin`) avec garde-fou statut user, anti-SQL injection systématique via mysql2 paramétré, validation des entrées au boundary HTTP, rate limiting sur les routes sensibles, tokens à usage unique haute entropie pour reset/validation, CORS strict avec liste blanche d'origines et garde-fou anti-wildcard, défense CSRF par SameSite + CORS, et neutralisation de l'énumération utilisateurs. Les éléments non couverts (helmet, refresh token, suppression de compte, audit log centralisé) sont identifiés et documentés comme évolutions V2 — le scope a été délibérément concentré sur la démonstration de la maîtrise des principes, pas la couverture exhaustive."

---

## Annexes — Fichiers clés

| Domaine                    | Fichier                                                                |
|----------------------------|------------------------------------------------------------------------|
| Configuration              | `backend/src/utils/env.ts`                                             |
| Bootstrap app (CORS, etc.) | `backend/src/app.ts`                                                   |
| Auth service               | `backend/src/services/auth/auth.service.ts`                            |
| Politique mot de passe     | `backend/src/services/auth/password-policy.ts`                         |
| Reset mot de passe         | `backend/src/services/auth/password-reset.service.ts`                  |
| Validation email           | `backend/src/services/auth/email-validation.service.ts`                |
| Tokens (random + hash)     | `backend/src/utils/security/password-reset-token.ts`                   |
| Cookie session             | `backend/src/utils/session-cookie.ts`                                  |
| Middleware auth            | `backend/src/middlewares/require-auth.ts`                              |
| Middleware admin           | `backend/src/middlewares/require-admin.ts`                             |
| Rate limiter               | `backend/src/middlewares/rate-limiter.ts`                              |
| Error handler              | `backend/src/middlewares/error-handler.ts`                             |
| Helpers DTO                | `backend/src/api/http/dto.helpers.ts`                                  |
| Service users (RGPD)       | `backend/src/services/users/users.service.ts`                          |
