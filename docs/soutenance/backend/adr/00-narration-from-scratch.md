# Narration "from scratch" — défense pour la soutenance

> **À qui s'adresse ce document** : à Arthur, pour préparer la défense de la
> question critique du jury sur le Bloc 2 :
> *"Le cahier des charges dit 'sans utiliser de frameworks ou de librairies
> prédéfinies', or vous utilisez Express, bcrypt, jsonwebtoken… Comment
> défendez-vous votre interprétation ?"*
>
> **Format** : argumentaire structuré + Q/R typiques. À reformuler avec tes
> propres mots avant la soutenance — un jury repère immédiatement un
> argumentaire récité.

---

## 1. Le piège de la formulation du cahier

La phrase exacte du cahier des charges est :

> *"Développer les fonctionnalités du back-end sans utiliser de frameworks
> ou de librairies prédéfinies. Utiliser PHP, Python ou Node.js pour coder
> les contrôleurs, les modèles et les vues. Utiliser les concepts liés à
> la Programmation Orientée Objet (POO)."*

Cette formulation laisse une marge d'interprétation considérable. Prise au
pied de la lettre :

- Pas de Node.js sans `node:http` (qui est une "librairie")
- Pas de connexion à MySQL sans driver
- Pas de hash de mot de passe sans bibliothèque cryptographique
- Pas de JWT sans implémenter la spec RFC 7519 à la main
- Pas de SMTP sans client mail

→ Ce serait absurde. **L'interprétation usuelle en formation RNCP** est :

> *"Sans framework full-stack opinionated qui pré-décide l'architecture
> et auto-génère du code (NestJS, AdonisJS, Symfony, Laravel, Spring Boot).
> Sans ORM (Sequelize, TypeORM, Prisma, Doctrine, Eloquent) qui masque
> le SQL et le mapping objet-relationnel."*

C'est cette interprétation qu'il faut **défendre activement**, pas
subir comme une excuse.

---

## 2. Ce qu'Arthur a écrit lui-même (la réalité du code)

| Composant | Status | Localisation |
|---|---|---|
| **Routes HTTP** (POST/GET/PUT/DELETE) | Écrites à la main | `src/api/*/routes.ts` |
| **Controllers** | Factory functions, signatures Express-natives | `src/api/*/controller.ts` |
| **Services** (logique métier) | Classes TypeScript, méthodes publiques + privées | `src/services/**/*.ts` |
| **Repositories** | Interface + implémentation MySQL séparées | `src/repositories/**/*.ts` |
| **DTOs et validation** | Parseurs custom, sans librairie type Zod/Joi | `src/api/*/dto.ts` + `src/api/http/dto.helpers.ts` |
| **Mappers** SQL → objet métier | Écrits à la main, fonction par entité | `src/repositories/**/*.mapper.ts` |
| **Middlewares** (auth, admin, rate-limit, errors) | Implémentés à la main | `src/middlewares/*.ts` |
| **Auth complète** (register/login/email validation/password reset) | Composée manuellement à partir de `bcrypt` et `jsonwebtoken` | `src/services/auth/**` |
| **Politique de mot de passe** | Règles métier custom | `src/services/auth/password-policy.ts` |
| **Gestion des sessions** (cookie HttpOnly + verify + status check à chaque requête) | Middleware custom | `src/middlewares/require-auth.ts` |
| **Transactions SQL** | Helper manuel `withTransaction()` | `src/db/transaction.ts` |
| **Génération de slug** (draft + public, avec vérif d'unicité) | Service métier | `src/services/recipes/recipe-slug.service.ts` |
| **Machine à états** des recettes (`draft → pending → published`) | Validation manuelle dans le service | `src/services/recipes/recipes.services.ts` |
| **Génération + hash des tokens** (validation email, reset password) | crypto natif Node | `src/utils/security/password-reset-token.ts` |
| **Pagination** | Helper custom (parsing + bornage) | `src/utils/pagination.ts` |
| **Templates emails** | Composés à la main dans `MailService` | `src/services/mail/mail.service.ts` |

**Volume** : ~80 fichiers TypeScript dans `src/`, ~40 fichiers de tests
dans `tests/`. La logique métier est entièrement la production d'Arthur.

---

## 3. Ce que sont les librairies utilisées (et pourquoi elles sont OK)

| Lib | Rôle | Équivalent "fait main" possible ? | Verdict |
|---|---|---|---|
| `express` | Routeur HTTP + middlewares + req/res | Oui (`node:http` + parsing manuel) | Boilerplate sans valeur pédagogique |
| `mysql2` | Driver MySQL (protocole TCP binaire) | Non | Impossible sans réimplémenter le protocole MySQL |
| `bcrypt` | Hash de mot de passe (Blowfish-based KDF) | Non | Réimplémenter une primitive crypto = faute professionnelle |
| `jsonwebtoken` | Génération + vérification de JWT (RFC 7519) | Théoriquement oui | Spec normalisée, bibliothèque audited, refaire à la main = risque sécurité gratuit |
| `cookie-parser` | Parse l'en-tête `Cookie:` (RFC 6265) | Oui (trivial) | 30 lignes, gardé par commodité |
| `cors` | Génère les en-têtes CORS (préflight, credentials) | Oui mais subtil | 200 lignes pour gérer tous les cas correctement |
| `dotenv` | Lit `.env` en variables d'environnement | Oui (50 lignes) | Standard de l'écosystème |
| `nodemailer` | Client SMTP (auth, TLS, multipart) | Non raisonnablement | Spec SMTP + extensions |

**Le critère commun** : aucune de ces librairies ne contient de **logique
métier** ou de **structure d'architecture**. Ce sont des **briques d'infrastructure**
qui implémentent des **standards** (HTTP, TCP MySQL, RFC JWT, RFC SMTP, RFC
Cookie, spec CORS).

Le cahier dit "from scratch sur les contrôleurs, modèles et vues" → ces
3 couches sont écrites à la main. ✅

---

## 4. Le test acide — 4 critères qui distinguent "from scratch" de "framework-driven"

Pour défendre que le projet est "from scratch" au sens du cahier :

| Critère | Réponse Recipe Shelter |
|---|---|
| 1. **Code généré automatiquement** par un scaffold (`nest generate`, `php artisan make`, etc.) ? | ❌ Aucun |
| 2. **Décorateurs magiques** (`@Controller`, `@Get`, `@Inject`, `@Entity`) ? | ❌ Aucun |
| 3. **ORM** qui auto-mappe les tables, génère le SQL, gère les relations ? | ❌ Aucun. Toutes les requêtes SQL sont écrites à la main dans les `*.repository.mysql.ts` |
| 4. **Injection de dépendances** automatique (container DI, métadonnées de classe) ? | ❌ Aucun. Le câblage des dépendances est explicite dans `src/app.ts` (`new XxxService(repo)`) |

Si les 4 réponses sont "Non", le projet est **architecturalement** from scratch.
C'est le cas ici.

---

## 5. Réponses-types aux attaques probables du jury

### Attaque 1 — *"Express est un framework, non ?"*

> *"Express est un micro-routeur HTTP, pas un framework full-stack. Il ne
> dicte ni l'architecture, ni la structure des fichiers, ni le pattern de
> persistance. Il fournit deux choses : un routeur (`app.get`, `app.post`)
> et un pipeline de middlewares. Ce qui aurait été l'équivalent fait main
> avec `node:http` : ~150 lignes de boilerplate sans valeur pédagogique
> (parsing de l'URL, dispatch par méthode, chaînage des middlewares).
> L'architecture en couches (controllers → services → repositories), le
> pattern Repository, les DTOs, l'auth, la modération — tout cela, je
> l'ai écrit à la main."*

### Attaque 2 — *"Vous utilisez `jsonwebtoken`, c'est une librairie."*

> *"JWT est un standard RFC 7519. Réimplémenter une bibliothèque de
> signature/vérification cryptographique à la main, c'est s'exposer à
> des failles classiques (algorithme `none`, timing attacks sur la
> comparaison HMAC, mauvaise gestion du padding). Le cahier des charges
> mentionne par ailleurs que la sécurité est un critère évalué. Utiliser
> une bibliothèque auditée pour la primitive cryptographique, puis
> implémenter toute la logique de session par-dessus (où je stocke le
> token, comment je le révoque, comment je re-vérifie le statut user à
> chaque requête), c'est la séparation responsable."*

### Attaque 3 — *"Et `bcrypt` ?"*

> *"Même argument. bcrypt implémente un Key Derivation Function lent
> (volontairement coûteux) basé sur Blowfish. C'est une primitive
> cryptographique. Réimplémenter ça à la main est non seulement
> au-delà du périmètre pédagogique, mais introduit un risque sécurité
> majeur. Ce que j'ai écrit moi-même, c'est la politique de mot de passe
> (`src/services/auth/password-policy.ts`) qui valide les règles métier
> avant le hash, et la stratégie de stockage (coût configurable via
> `BCRYPT_COST`)."*

### Attaque 4 — *"Pourquoi pas un ORM ? Ça aurait été plus simple."*

> *"Justement. L'absence d'ORM est un choix pédagogique délibéré. Avec un
> ORM, je n'aurais pas eu à concevoir mes mappers, ni à gérer mes
> transactions explicitement, ni à optimiser mes requêtes (FULLTEXT index
> sur `Recipes.Title`, JOIN multi-table dans la recherche). En écrivant
> mes repositories MySQL à la main, j'ai dû maîtriser le SQL en profondeur,
> ce qui est exactement ce que demande le bloc 2 sur la 'manipulation des
> données'. Le coût : plus de code à maintenir. Le bénéfice : compréhension
> totale du flux de données."*

### Attaque 5 — *"Pourquoi un découpage en interface + implémentation pour les repositories ?"*

> *"Inversion de dépendances, un des principes SOLID. Les services ne
> connaissent pas MySQL : ils dépendent d'une interface `RecipeRepository`.
> L'implémentation `RecipeRepositoryMysql` est branchée au démarrage dans
> `src/app.ts`. Deux bénéfices : (1) je peux tester les services unitairement
> en branchant un repository en mémoire — c'est ce qui me permet d'avoir
> ~40 fichiers de tests qui tournent sans MySQL ; (2) si on devait
> migrer vers Postgres ou un autre SGBD, seule l'implémentation change.
> Le pattern Repository est documenté depuis le DDD d'Eric Evans."*

---

## 6. Stratégie de discours pendant la soutenance

**Si la question vient au début** : ne pas être sur la défensive. Répondre
de façon affirmative et structurée :

> *"Je veux clarifier ce point dès le début car la formulation du cahier
> peut être ambiguë. Ce que j'ai écrit moi-même, c'est l'architecture
> complète : controllers, services, repositories, DTOs, middlewares,
> politique d'auth, machine à états des recettes, modération. Ce que
> j'utilise comme bibliothèques externes, ce sont uniquement des briques
> d'infrastructure standard : un routeur HTTP, un driver MySQL, un
> hasher de mot de passe. Aucun framework full-stack, aucun ORM,
> aucun scaffolding. Si vous voulez, je peux vous montrer un controller
> et un service pour illustrer la séparation."*

**Si la question vient en milieu/fin** : ramener à du concret en montrant
un fichier. Le plus parlant : `src/app.ts` (le câblage manuel de toutes
les dépendances) et `src/repositories/recipes/recipe.repository.mysql.ts`
(le SQL écrit à la main, les JOIN, la pagination).

**Ne jamais dire** :
- *"Ben tout le monde utilise Express"* (argument d'autorité faible)
- *"On nous a dit que c'était OK"* (renvoie au formateur, fragilise la défense)
- *"C'est juste un détail technique"* (sous-estime le jury)

**Toujours ramener à** :
- Les **principes SOLID** mis en œuvre (inversion de dépendances, SRP)
- Les **standards** que les libs respectent (RFC JWT, protocole MySQL)
- Le **code écrit à la main** dans les couches métier

---

## 7. Documents complémentaires

- [adr-001-express-comme-couche-http.md](adr-001-express-comme-couche-http.md) — décision formelle d'utiliser Express
- [adr-002-pattern-repository-interface-impl.md](adr-002-pattern-repository-interface-impl.md) — séparation interface/impl
- [adr-003-jwt-cookie-httponly.md](adr-003-jwt-cookie-httponly.md) — stratégie d'authentification
- Tous les ADRs servent de **support écrit** que le jury peut consulter et
  qui démontrent que les choix sont **réfléchis et tracés**, pas faits au
  hasard.
