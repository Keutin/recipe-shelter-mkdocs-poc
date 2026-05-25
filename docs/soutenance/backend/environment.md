# Variables d'environnement du backend Recipe Shelter

> Document de référence pour la défense de soutenance (RNCP). Décrit la gestion
> centralisée des variables d'environnement du backend : pourquoi un module
> dédié, comment chaque variable est lue et validée, et ce qu'il faut fournir
> pour déployer. Versionné dans `documentation/soutenance/backend/environment.md`
> (cible), brouillon courant dans `_draft_backend_docs/environment.md`.

## 1. Pourquoi un module `env.ts` centralisé

Le backend ne lit jamais `process.env.X` directement dans le code métier. La
totalité des variables d'environnement est résolue dans un seul fichier,
`backend/src/utils/env.ts`, qui exporte un objet `env` typé. Tous les
consommateurs (services, repositories, configuration d'Express, middlewares
d'authentification, mailer) importent cet objet plutôt que d'aller chercher la
variable brute. Ce parti pris a trois conséquences directes.

**Validation au démarrage (fail-fast).** Le fichier est évalué dès le premier
import, qui est déclenché par `backend/src/app.ts`. Toute variable obligatoire
manquante fait planter le processus immédiatement, avant même que le serveur
HTTP n'écoute. C'est le cas de `JWT_SECRET` qui, s'il est absent, lève
explicitement l'erreur `JWT_SECRET is required`
([backend/src/utils/env.ts:90`). Le mode "le
serveur démarre mais explose à la première requête" est ainsi écarté.

**Typage TypeScript des consommateurs.** L'objet `env` est typé par inférence.
Les services qui en dépendent reçoivent des `number` quand le code attend un
`number`, pas des `string | undefined`. Il n'y a pas de `parseInt` éparpillé
dans le code métier, pas de `process.env.X === 'true'` répété un peu partout.
La conversion de type est faite une fois, à un endroit, avec des helpers dont
le comportement est documenté ci-dessous.

**Defaults explicites et documentés.** Chaque variable a soit une valeur par
défaut codée en clair dans `env.ts`, soit elle est marquée comme obligatoire.
Quand on lit le fichier on connaît immédiatement le comportement par défaut du
backend en l'absence de `.env`. C'est notamment ce qui permet aux tests
d'intégration et aux développeurs qui clonent le repo de démarrer sans
configuration préalable, à l'exception du `JWT_SECRET`.

Le chargement du fichier `.env` est délégué à `dotenv` via l'import
`import 'dotenv/config';` en première ligne du module. Aucun appel manuel à
`dotenv.config()` n'est nécessaire ailleurs dans le code.

## 2. Architecture du parsing

Le module définit six fonctions utilitaires privées, toutes situées en haut de
`backend/src/utils/env.ts`. Elles encapsulent la lecture d'une variable
d'environnement et la conversion vers le type cible attendu par les
consommateurs.

### 2.1 `readString(value, fallback)` — [env.ts:27-29`

```ts
function readString(value: string | undefined, fallback: string): string {
  return value && value.trim() ? value.trim() : fallback;
}
```

Retourne la valeur trimmée si elle est non vide, sinon le fallback. Une
variable définie mais composée uniquement d'espaces est considérée comme non
définie. Cette règle évite les pièges classiques (`DB_USER= `, copié-collé
maladroit) et garantit que les valeurs reçues côté métier sont toujours
significatives.

### 2.2 `readNumber(value, fallback)` — [env.ts:3-10`

```ts
function readNumber(value: string | undefined, fallback: number): number {
  if (!value?.trim())
    return fallback;

  const number = Number(value);

  return Number.isFinite(number) ? number : fallback;
}
```

Convertit via `Number(...)`. Toute valeur non numérique (lettre, chaîne vide,
`NaN`, `Infinity`) retombe sur le fallback. Cette tolérance est volontaire : on
préfère démarrer avec une valeur sûre que planter sur une faute de frappe
dans le `.env`, à l'exception explicite des variables réellement
obligatoires (cf. `JWT_SECRET`).

### 2.3 `readBoolean(value, fallback)` — [env.ts:12-25`

```ts
function readBoolean(value: string | undefined, fallback: boolean): boolean {
  const normalizedValue = value?.trim().toLowerCase();

  if (!normalizedValue)
    return fallback;

  if (['1', 'true', 'yes', 'on'].includes(normalizedValue))
    return true;

  if (['0', 'false', 'no', 'off'].includes(normalizedValue))
    return false;

  return fallback;
}
```

Reconnaît plusieurs notations courantes (`true/false`, `1/0`, `yes/no`,
`on/off`) et est case-insensitive. Une valeur non reconnue retombe sur le
fallback. L'intention est de tolérer les conventions des différents
environnements d'hébergement (Railway, Docker, systemd) sans avoir à mémoriser
laquelle est attendue.

### 2.4 `readOptionalString(value)` — [env.ts:31-33`

```ts
function readOptionalString(value: string | undefined): string | undefined {
  return value && value.trim() ? value.trim() : undefined;
}
```

Variante de `readString` sans fallback : la fonction retourne `undefined` si la
variable est absente ou vide. Elle est utilisée pour les variables réellement
optionnelles, dont la simple absence est un signal métier — par exemple
`AUTH_SESSION_COOKIE_DOMAIN`, qui par défaut n'est pas positionné sur le
cookie de session pour rester compatible avec `localhost`.

### 2.5 `readSameSite(value, fallback)` — [env.ts:35-42`

```ts
function readSameSite(value, fallback: 'strict' | 'lax' | 'none') {
  const normalizedValue = value?.trim().toLowerCase();

  if (normalizedValue === 'strict' || normalizedValue === 'lax' || normalizedValue === 'none')
    return normalizedValue;

  return fallback;
}
```

Helper spécialisé pour la valeur de l'attribut `SameSite` du cookie de
session. Le type de retour est une union littérale (`'strict' | 'lax' |
'none'`), ce qui force les consommateurs à traiter exactement ces trois cas et
permet à TypeScript de vérifier l'exhaustivité.

### 2.6 `readDurationMs(value, fallback)` — [env.ts:44-65`

```ts
function readDurationMs(value: string, fallback: number): number {
  const match = value.trim().match(/^(\d+)(ms|s|m|h|d)?$/i);
  // ... amount * multiplier (ms, s, m, h, d)
}
```

Parse une durée exprimée avec un suffixe d'unité (`7d`, `15m`, `3600s`,
`500ms`) et la convertit en millisecondes. Cette fonction est utilisée
exactement une fois, en interne, pour dériver la durée de vie du cookie de
session à partir de `JWT_EXPIRES_IN` (cf. §5.3).

### 2.7 Le cas particulier `JWT_SECRET` (fail-fast)

`JWT_SECRET` est la seule variable du backend qui ne passe pas par un helper.
Elle est lue directement, avec une IIFE qui lève une exception si la variable
est absente :

```ts
jwtSecret: process.env.JWT_SECRET ?? (() => { throw new Error('JWT_SECRET is required'); })(),
```

[env.ts:90`. Le choix est délibéré : un secret de
signature JWT n'a pas de valeur par défaut acceptable. Toute valeur générique
("changeme", "secret"...) ferait du backend une cible triviale pour la
fabrication de tokens. Plutôt que de proposer un fallback dangereux, on
préfère casser le démarrage et obliger l'opérateur à fournir une vraie
valeur. C'est l'unique variable strictement obligatoire du backend, et c'est
la seule qui doit être pensée comme telle lors d'un déploiement.

## 3. Inventaire complet des variables

Le backend lit vingt-neuf variables au total, regroupées en sept sous-systèmes
fonctionnels. Le tableau ci-dessous donne pour chacune le type cible, la valeur
par défaut, le caractère obligatoire ou non, la ligne exacte de `env.ts` et un
résumé de l'usage.

### 3.1 Runtime

| Nom        | Type   | Défaut        | Requis ? | Fichier:ligne   | Description                                                                                 |
| ---------- | ------ | ------------- | -------- | --------------- | ------------------------------------------------------------------------------------------- |
| `NODE_ENV` | string | `development` | non      | env.ts:67       | Mode d'exécution. Influence le défaut de `AUTH_SESSION_COOKIE_SECURE` (cf. §5.1).            |
| `PORT`     | number | `3000`        | non      | env.ts:73       | Port HTTP d'écoute du serveur Express.                                                      |

### 3.2 Database (MySQL)

| Nom                   | Type   | Défaut            | Requis ? | Fichier:ligne | Description                                                                  |
| --------------------- | ------ | ----------------- | -------- | ------------- | ---------------------------------------------------------------------------- |
| `DB_HOST`             | string | `127.0.0.1`       | non      | env.ts:81     | Hôte du serveur MySQL.                                                       |
| `DB_PORT`             | number | `3306`            | non      | env.ts:82     | Port du serveur MySQL.                                                       |
| `DB_USER`             | string | `root`            | non      | env.ts:83     | Utilisateur MySQL utilisé par le pool.                                       |
| `DB_PASSWORD`         | string | `` (vide)         | non      | env.ts:84     | Mot de passe MySQL. Secret, ne doit jamais être commit.                      |
| `DB_NAME`             | string | `recipe_shelter`  | non      | env.ts:85     | Nom de la base de données cible.                                             |
| `DB_CONNECTION_LIMIT` | number | `10`              | non      | env.ts:86     | Taille maximale du pool de connexions `mysql2/promise`.                      |

Les défauts sont calibrés pour un MySQL local sans configuration. En
production, l'ensemble des cinq premières variables doit être réécrit, et il
est recommandé de revoir `DB_CONNECTION_LIMIT` en fonction de la capacité de
l'instance MySQL hébergée.

### 3.3 Authentification (JWT + session)

| Nom                                | Type     | Défaut                          | Requis ? | Fichier:ligne | Description                                                                                       |
| ---------------------------------- | -------- | ------------------------------- | -------- | ------------- | ------------------------------------------------------------------------------------------------- |
| `JWT_SECRET`                       | string   | aucun                           | **oui**  | env.ts:90     | Secret HMAC de signature des tokens JWT. Absence = crash au démarrage.                            |
| `JWT_EXPIRES_IN`                   | string   | `7d`                            | non      | env.ts:68, 91 | Durée de validité du JWT (notation `jsonwebtoken` : `7d`, `15m`, `3600s`).                        |
| `AUTH_SESSION_COOKIE_NAME`         | string   | `rs_session`                    | non      | env.ts:92     | Nom du cookie HttpOnly portant le JWT.                                                            |
| `AUTH_SESSION_COOKIE_DOMAIN`       | string?  | `undefined`                     | non      | env.ts:93     | Domaine du cookie. Vide = cookie restreint à l'host courant (utile en local).                     |
| `AUTH_SESSION_COOKIE_SAME_SITE`    | enum     | `lax`                           | non      | env.ts:94     | Attribut `SameSite` du cookie. Valeurs autorisées : `strict`, `lax`, `none`.                      |
| `AUTH_SESSION_COOKIE_SECURE`       | boolean  | `nodeEnv === 'production'`      | non      | env.ts:95     | Attribut `Secure` du cookie. Vrai en production par défaut (cf. §5.1).                            |
| `AUTH_SESSION_COOKIE_MAX_AGE_MS`   | number   | dérivé de `JWT_EXPIRES_IN`      | non      | env.ts:96     | Durée de vie du cookie en ms. Par défaut, équivalent ms de `JWT_EXPIRES_IN` (cf. §5.3).           |
| `AUTH_DEFAULT_ROLE_NAME`           | string   | `user`                          | non      | env.ts:97     | Nom du rôle assigné à un nouvel inscrit (clé étrangère vers la table `roles`).                    |
| `BCRYPT_COST`                      | number   | `12`                            | non      | env.ts:98     | Facteur de coût bcrypt. Plus élevé = hash plus lent et plus résistant.                            |

### 3.4 Rate limiting (anti brute force)

| Nom                            | Type   | Défaut       | Requis ? | Fichier:ligne | Description                                                                          |
| ------------------------------ | ------ | ------------ | -------- | ------------- | ------------------------------------------------------------------------------------ |
| `AUTH_RATE_LIMIT_MAX_ATTEMPTS` | number | `5`          | non      | env.ts:99     | Nombre maximal de tentatives de connexion par fenêtre.                               |
| `AUTH_RATE_LIMIT_WINDOW_MS`    | number | `900000`     | non      | env.ts:100    | Largeur de la fenêtre glissante, en millisecondes (défaut : 15 minutes).             |

Le rate limiting est appliqué sur les endpoints d'authentification (login,
demande de reset password) pour limiter l'efficacité d'attaques par force
brute. Voir `documentation/soutenance/backend/securite.md` pour la mise en
œuvre.

### 3.5 CORS et frontend

| Nom                    | Type   | Défaut                                                  | Requis ? | Fichier:ligne | Description                                                                                 |
| ---------------------- | ------ | ------------------------------------------------------- | -------- | ------------- | ------------------------------------------------------------------------------------------- |
| `CORS_ALLOWED_ORIGINS` | string | `http://localhost:4200,http://127.0.0.1:4200`           | non      | env.ts:76     | Liste d'origines autorisées par CORS, séparées par des virgules. Pas de wildcard accepté.   |
| `FRONTEND_BASE_URL`    | string | `http://localhost:4200`                                 | non      | env.ts:77     | URL de base du frontend, utilisée pour construire les liens dans les emails transactionnels. |

`CORS_ALLOWED_ORIGINS` est une chaîne — le découpage en tableau est fait dans
le middleware CORS. Le format CSV est volontaire : il facilite la déclaration
dans les UI d'hébergeurs (Railway, Heroku) qui n'acceptent qu'une seule string
par variable d'environnement.

### 3.6 SMTP (envoi d'emails)

| Nom              | Type    | Défaut    | Requis ? | Fichier:ligne | Description                                                                                                |
| ---------------- | ------- | --------- | -------- | ------------- | ---------------------------------------------------------------------------------------------------------- |
| `SMTP_HOST`      | string  | `` (vide) | non      | env.ts:104    | Hôte du serveur SMTP. Vide = mailer non fonctionnel mais ne fait pas crasher le backend.                   |
| `SMTP_PORT`      | number  | `587`     | non      | env.ts:105    | Port SMTP. 587 = STARTTLS, 465 = TLS implicite.                                                            |
| `SMTP_SECURE`    | boolean | `false`   | non      | env.ts:106    | Active le mode TLS implicite. À mettre à `true` si `SMTP_PORT=465`.                                        |
| `SMTP_USER`      | string  | `` (vide) | non      | env.ts:107    | Identifiant SMTP. Secret.                                                                                  |
| `SMTP_PASSWORD`  | string  | `` (vide) | non      | env.ts:108    | Mot de passe SMTP. Secret, ne doit jamais être commit.                                                     |
| `SMTP_FROM`      | string  | `` (vide) | non      | env.ts:109    | Adresse `From:` des emails transactionnels (validation de compte, reset password, notification contact).   |

Toutes les variables SMTP ont un fallback vide qui n'empêche pas le backend de
démarrer. C'est un choix : on accepte qu'un environnement de développement
fonctionne sans mailer (avec des emails qui partent en erreur loggée plutôt
qu'envoyés). En production il faut évidemment fournir l'ensemble des
identifiants.

### 3.7 Contact

| Nom                       | Type   | Défaut    | Requis ? | Fichier:ligne | Description                                                                |
| ------------------------- | ------ | --------- | -------- | ------------- | -------------------------------------------------------------------------- |
| `CONTACT_RECIPIENT_EMAIL` | string | `` (vide) | non      | env.ts:110    | Adresse qui reçoit les messages soumis via le formulaire de contact.       |

Cette variable est distincte de `SMTP_FROM` : `SMTP_FROM` est l'expéditeur
technique des emails sortants, `CONTACT_RECIPIENT_EMAIL` est la destination
métier des messages de contact (typiquement une boîte interne à l'équipe).

## 4. Variables secrètes vs configuration

Toutes les variables ne sont pas équivalentes vis-à-vis de la sécurité. On
distingue deux catégories.

**Secrets — ne doivent jamais être commit, jamais loggés, doivent être stockés
dans un coffre (variables d'environnement de l'hébergeur, gestionnaire de
secrets) :**

- `JWT_SECRET` — la compromission permet de fabriquer des tokens valides pour
  n'importe quel utilisateur, y compris admin.
- `DB_PASSWORD` — accès complet à la base, donc à toutes les données
  utilisateur et à la possibilité de tout supprimer.
- `SMTP_PASSWORD` — accès à la boîte d'envoi, donc capacité à usurper l'identité
  du service auprès des utilisateurs (phishing, reset password forgé).
- `SMTP_USER` — moins critique seul, mais combiné au password donne un accès
  complet.

**Configuration — visible dans le code, dans la doc, dans les exemples, sans
risque :**

- `NODE_ENV`, `PORT`, `DB_HOST`, `DB_PORT`, `DB_USER`, `DB_NAME`,
  `DB_CONNECTION_LIMIT`.
- `JWT_EXPIRES_IN`, `AUTH_SESSION_COOKIE_*` (nom, domain, sameSite, secure,
  maxAge), `AUTH_DEFAULT_ROLE_NAME`, `BCRYPT_COST`.
- `AUTH_RATE_LIMIT_*`.
- `CORS_ALLOWED_ORIGINS`, `FRONTEND_BASE_URL`.
- `SMTP_HOST`, `SMTP_PORT`, `SMTP_SECURE`, `SMTP_FROM`,
  `CONTACT_RECIPIENT_EMAIL`.

Le repository applique cette distinction de la manière suivante :

- `backend/.gitignore` inclut `.env` (et toutes ses variantes : `.env.local`,
  `.env.*.local`). Le fichier `.env` réel n'est donc jamais poussé.
- `backend/.env.example` est commit dans le dépôt. Il contient la liste des
  variables avec des placeholders (`your_database_password`,
  `your_long_random_jwt_secret`) pour servir de modèle. C'est ce fichier que
  l'on copie en `.env` au premier clonage.
- Aucun secret de production n'est jamais dans le code ni dans la
  documentation. La doc déploiement (`_draft_deployment/`) explique où injecter
  ces secrets côté hébergeur.

Le commit `feat: ajoute .env.example` est cité en soutenance comme exemple de
ce que peut être un template de configuration commit-safe.

## 5. Comportement environnement-dépendant

Trois variables ont un comportement qui dépend d'autres variables ou de
l'environnement d'exécution. C'est volontaire : ces dépendances évitent au
développeur de devoir maintenir manuellement des cohérences entre paramètres
liés.

### 5.1 `AUTH_SESSION_COOKIE_SECURE` dépend de `NODE_ENV`

[env.ts:95` :

```ts
sessionCookieSecure: readBoolean(process.env.AUTH_SESSION_COOKIE_SECURE, nodeEnv === 'production'),
```

Si la variable n'est pas fournie, la valeur par défaut est `true` en
production et `false` en développement. C'est une protection : oublier de
passer `AUTH_SESSION_COOKIE_SECURE=true` en production aurait pour effet
d'émettre un cookie de session sans le flag `Secure`, donc transmissible en
clair sur HTTP. En faisant dépendre le défaut de `NODE_ENV`, on garantit que
le comportement sûr est le défaut quand on est dans un environnement qui le
justifie. Inversement, en développement local on accepte un cookie non sécurisé
pour pouvoir utiliser `http://localhost:4200` sans certificat TLS.

### 5.2 `CORS_ALLOWED_ORIGINS` doit être réécrit en production

Le défaut `http://localhost:4200,http://127.0.0.1:4200`
([env.ts:76`) est calibré pour un développement
local avec Angular CLI sur son port standard. Il ne convient évidemment pas à
une instance déployée : il faut renseigner les origines réelles du frontend
hébergé (`https://recipe-shelter.fr`, `https://staging.recipe-shelter.fr`,
etc.).

Ne pas réécrire cette variable a deux conséquences. Premièrement les requêtes
du frontend déployé sont rejetées par CORS (préflight `OPTIONS` qui renvoie le
mauvais `Access-Control-Allow-Origin`). Deuxièmement le mode "credentials"
est inopérant car le middleware CORS refuse explicitement le wildcard `*`
quand `credentials: true` (cf. `architecture.md` §2).

### 5.3 `AUTH_SESSION_COOKIE_MAX_AGE_MS` dérivé de `JWT_EXPIRES_IN`

[env.ts:67-69, 96` :

```ts
const jwtExpiresIn = readString(process.env.JWT_EXPIRES_IN, '7d');
const defaultSessionCookieMaxAgeMs = readDurationMs(jwtExpiresIn, 604800000);
// ...
sessionCookieMaxAgeMs: readNumber(process.env.AUTH_SESSION_COOKIE_MAX_AGE_MS, defaultSessionCookieMaxAgeMs),
```

Si `AUTH_SESSION_COOKIE_MAX_AGE_MS` n'est pas fourni, la durée du cookie est
calculée à partir de `JWT_EXPIRES_IN`. Concrètement, si l'opérateur fixe
`JWT_EXPIRES_IN=7d`, le cookie a une durée de vie de 604 800 000 ms (sept
jours) sans qu'il ait besoin de la calculer lui-même. Si `JWT_EXPIRES_IN`
n'est pas non plus fourni, le défaut de défaut est sept jours.

Le bénéfice : on évite l'incohérence où le cookie expire dans le navigateur
mais le JWT à l'intérieur est encore valide côté serveur (ou l'inverse). Il
reste possible d'override manuellement
`AUTH_SESSION_COOKIE_MAX_AGE_MS` pour les cas où l'on veut explicitement
découpler la durée du cookie de celle du JWT (exemple : on souhaite que le
cookie soit purgé à la fermeture du navigateur, on positionne donc une valeur
faible).

## 6. Préparation au déploiement

Pour déployer le backend dans un environnement de production (Railway,
Heroku, VPS, Docker en orchestrateur), il faut au minimum fournir les
variables suivantes — toutes les autres peuvent rester sur leurs défauts, mais
gagneront à être revues.

**Strictement obligatoires (le backend ne démarre pas sinon) :**

- `JWT_SECRET` — chaîne aléatoire d'au moins 32 octets. Générer avec
  `openssl rand -base64 48` ou équivalent. Ne jamais réutiliser un secret de
  développement.

**Obligatoires en pratique (le backend démarre mais ne fonctionne pas) :**

- `DB_HOST`, `DB_PORT`, `DB_USER`, `DB_PASSWORD`, `DB_NAME` — coordonnées de
  la base MySQL hébergée. Sans ces variables, le pool est créé avec des
  défauts qui pointent sur un MySQL local inexistant.
- `SMTP_HOST`, `SMTP_PORT`, `SMTP_USER`, `SMTP_PASSWORD`, `SMTP_FROM` —
  configuration du serveur SMTP. Sans cela, les emails de validation et de
  reset password ne partent pas, ce qui rend l'inscription inutilisable.
- `CORS_ALLOWED_ORIGINS` — origines réelles du frontend déployé. Sans cela,
  le frontend hébergé ne peut pas appeler le backend.
- `FRONTEND_BASE_URL` — URL réelle du frontend. Utilisée pour construire les
  liens cliquables dans les emails (validation de compte, reset password). Si
  la valeur est `http://localhost:4200`, les utilisateurs reçoivent des liens
  inutilisables.

**Recommandés en production :**

- `AUTH_SESSION_COOKIE_SECURE=true` — explicite plutôt que dérivé de
  `NODE_ENV`, pour ne pas dépendre d'un effet de bord (cf. §5.1).
- `AUTH_SESSION_COOKIE_DOMAIN=.recipe-shelter.fr` — partage le cookie entre
  les sous-domaines (`api.`, `www.`). Indispensable si frontend et backend
  sont sur des sous-domaines différents.
- `NODE_ENV=production` — active aussi les chemins "production" d'Express
  (template caching, désactivation des stack traces verbeuses).
- `CONTACT_RECIPIENT_EMAIL` — adresse interne qui reçoit les messages du
  formulaire de contact.

**Optionnels mais pertinents à revoir :**

- `BCRYPT_COST` — peut être monté à 13 ou 14 si la machine cible le permet
  sans dégrader le temps de réponse du login au-delà de ~250 ms.
- `AUTH_RATE_LIMIT_MAX_ATTEMPTS` et `AUTH_RATE_LIMIT_WINDOW_MS` — à
  ajuster selon le profil de trafic attendu et la tolérance fonctionnelle au
  blocage temporaire.
- `DB_CONNECTION_LIMIT` — à régler en fonction du nombre maximal de
  connexions accepté par l'instance MySQL hébergée (souvent 50 à 100 sur les
  plans entry-level).

Le détail des étapes d'installation et la mise à jour de chaque variable côté
Railway sont décrits dans
``_draft_deployment/02-plan-railway.md``.
Ce document couvre notamment la création du projet Railway, le branchement du
service MySQL géré et l'import des variables via l'interface web ou la CLI.

## 7. Points d'amélioration identifiés

L'implémentation actuelle est volontairement simple, à l'image du reste du
backend (cf. `architecture.md` §1 sur l'absence de framework). Trois limites
sont identifiées et assumées pour le périmètre du projet de certification.

**Pas de schéma de validation déclaratif.** La validation des variables est
manuelle, helper par helper. Il n'y a pas de bibliothèque comme `zod`,
`envalid`, ou `@nestjs/config` qui définirait un schéma explicite ("PORT est un
entier entre 1 et 65535", "SMTP_PORT vaut 25, 465, 587 ou 2525"). En
conséquence :

- Une valeur numérique aberrante (`PORT=99999`, `BCRYPT_COST=-1`) sera
  acceptée et provoquera une erreur plus loin dans le code, parfois opaque.
- Une typo silencieuse (`AUTH_SESSION_COOKIE_SAMESITE` au lieu de
  `..._SAME_SITE`) sera ignorée sans message d'erreur, le fallback sera
  utilisé sans que personne ne le remarque.
- Il n'existe pas d'introspection automatique : pour savoir quelles variables
  sont lues, il faut lire `env.ts` à la main. C'est ce que fait justement ce
  document.

Une évolution naturelle serait de remplacer les helpers par un schéma `zod`
qui validerait à la fois le format, les bornes, et produirait un message
d'erreur explicite au démarrage en cas de problème. Le coût serait une
dépendance supplémentaire — ce qui à l'échelle du projet est acceptable mais
contrevient au principe de minimalisme retenu.

**Pas de support `.env.local` ni `.env.production` séparés.** Le module
`dotenv` est appelé avec sa configuration par défaut, qui charge uniquement
`.env`. Il n'y a pas de cascade `.env.production.local > .env.production >
.env.local > .env` comme dans Vite ou Next.js. Conséquence : pour basculer
entre plusieurs environnements en local (dev, test, staging), il faut copier
le bon fichier sur `.env`. C'est gérable à l'échelle du projet mais
mériterait une amélioration si le périmètre grandissait — typiquement en
utilisant `dotenv-flow` ou en chargeant explicitement
`.env.${NODE_ENV}` au démarrage.

**`JWT_SECRET` lisible en clair en mémoire.** Le secret de signature est
chargé en mémoire dès le démarrage et y reste pendant toute la vie du
processus. Toute compromission du processus (dump mémoire, débogueur attaché,
lecture de `/proc/self/environ` sur Linux) expose le secret. En production
sérieuse on délèguerait la signature à un KMS (AWS KMS, GCP Cloud KMS, HashiCorp
Vault Transit), de sorte que le secret ne quitte jamais le coffre — le backend
appelle une API de signature à la place. C'est hors périmètre pour le projet
de certification mais constitue une réponse intéressante à apporter en
soutenance si le jury interroge la gestion des secrets.

Aucune de ces trois limites n'empêche le bon fonctionnement du backend dans
le périmètre démontré. Elles sont mentionnées pour montrer la conscience des
limites de l'implémentation et la capacité à proposer une évolution
proportionnée au besoin réel.

