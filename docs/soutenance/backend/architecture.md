---
title: Architecture backend
description: Architecture applicative du backend Recipe Shelter (Bloc 2)
tags:
  - bloc-2
  - backend
  - architecture
---

# Architecture du backend Recipe Shelter

> Document de référence pour la défense de soutenance (RNCP). Décrit l'architecture
> applicative du backend, ses choix techniques et ses partis pris pédagogiques.
> Versionné dans `documentation/soutenance/backend/architecture.md` (cible),
> brouillon courant dans `_draft_backend_docs/architecture.md`.

## 1. Vue d'ensemble

Recipe Shelter est une plateforme communautaire de partage de recettes de cuisine.
Le backend expose une API HTTP REST consommée exclusivement par le frontend
Angular (et accessoirement par les tests E2E et l'interface d'administration).
Il assume la totalité de la logique métier : gestion des comptes utilisateurs,
cycle de vie des recettes (brouillon → soumission → modération → publication →
archivage), interactions communautaires (commentaires, favoris) et workflows
transverses (validation d'email, réinitialisation de mot de passe, formulaire
de contact).

Le projet est délibérément construit **sans framework applicatif lourd**. Il ne
s'appuie ni sur NestJS, ni sur AdonisJS, ni sur un container d'injection de
dépendances. Express 5 fournit le routeur HTTP et la chaîne de middlewares,
`mysql2/promise` fournit le driver de base de données, le reste est écrit à la
main : couches applicatives, câblage des dépendances, validation des DTO,
gestion des erreurs, sérialisation des réponses. Ce choix est revendiqué : il
permet de **comprendre et de défendre chaque ligne de code** lors de la
soutenance, par opposition à une démarche où la magie d'un framework masque le
fonctionnement réel de l'application.

L'architecture est organisée en trois couches strictement séparées
(`api/` → `services/` → `repositories/`), reliées entre elles par un **câblage
manuel explicite** réalisé dans `backend/src/app.ts`. Chaque dépendance entre
composants est visible dans ce fichier, ce qui en fait la pièce centrale pour
expliquer comment l'application s'assemble.

La base de données est MySQL 8, accédée via un pool de connexions partagé. Les
transactions multi-tables (création d'une recette avec ses ingrédients, étapes,
équipements et tags par exemple) sont gérées explicitement avec
`BEGIN/COMMIT/ROLLBACK`. L'envoi d'emails (validation de compte, mot de passe
oublié, formulaire de contact) repose sur Nodemailer via un service `Mailer`
injecté aux services métier qui en ont besoin.

## 2. Stack technique

| Composant                | Bibliothèque         | Version    | Rôle / raison du choix                                                                                  |
| ------------------------ | -------------------- | ---------- | ------------------------------------------------------------------------------------------------------- |
| Runtime                  | Node.js              | >= 20      | LTS, ESM natif, test runner intégré (`node:test`).                                                      |
| Langage                  | TypeScript           | ^5.9.3     | Typage statique strict, contrats explicites entre couches, refactor sûr.                                |
| Routeur HTTP             | express              | ^5.2.1     | Minimaliste, pipeline de middlewares standard, large compatibilité.                                     |
| Driver base de données   | mysql2/promise       | ^3.18.2    | Pool de connexions, requêtes préparées, `namedPlaceholders`, support Promise natif.                     |
| Hashage mot de passe     | bcrypt               | ^6.0.0     | Standard éprouvé, coût configurable (`BCRYPT_COST`, défaut 12).                                         |
| JWT                      | jsonwebtoken         | ^9.0.3     | Signature et vérification des sessions, payload typé `AuthTokenPayload`.                                |
| Cookies                  | cookie-parser        | ^1.4.7     | Lecture du cookie de session HttpOnly côté serveur.                                                     |
| CORS                     | cors                 | ^2.8.6     | Allow-list explicite des origines autorisées, refus du wildcard avec `credentials: true`.               |
| Variables d'environnement| dotenv               | ^17.3.1    | Chargement du `.env` au démarrage, lecture typée via `utils/env.ts`.                                    |
| Envoi d'emails           | nodemailer           | ^8.0.7     | SMTP standard, abstrait derrière l'interface `Mailer`.                                                  |
| Test runner              | node:test (natif)    | n/a        | Pas de dépendance externe (Jest, Mocha) ; tests TypeScript via `tsx`.                                   |
| Lint                     | ESLint + ts-eslint   | ^9.39.4    | Règles strictes sur l'ordre des imports, les types implicites, le style.                                |

Toutes les dépendances de production tiennent sur **huit packages**. C'est
volontairement minimal : chaque ligne du `package.json` représente une décision
défendable plutôt qu'une dépendance transitive subie.

Référence : [backend/package.json](https://github.com/arthur-lagenebre/recipe-shelter-backend/blob/main/package.json#L40-L49).

## 3. Architecture en couches

L'architecture suit un découpage en trois couches, du plus proche du transport
HTTP au plus proche de la persistance.

```
┌──────────────────────────────────────────────────────────────────┐
│  api/        Controllers + Routes + DTO          (couche HTTP)   │
│      │       Parsing des requêtes, sérialisation des réponses    │
│      ▼                                                            │
│  services/   Logique métier, règles, orchestration               │
│      │       Indépendant d'Express, testable en isolation        │
│      ▼                                                            │
│  repositories/  Accès aux données : interface + impl + mapper    │
│                 Pattern Repository, isolation du SQL             │
└──────────────────────────────────────────────────────────────────┘
                              │
                              ▼
                         MySQL (pool mysql2)
```

### 3.1 Couche `api/` — controllers, routes et DTO

Chaque domaine fonctionnel dispose d'un sous-dossier dans `backend/src/api/`
qui contient au minimum trois fichiers :

- `*.controller.ts` — adapte la requête HTTP au service métier ;
- `*.routes.ts` — déclare les routes Express et leur enchaînement de middlewares ;
- `*.dto.ts` — valide et type le payload entrant, formate la sortie si besoin.

**Convention de nommage :** les fabriques exportées sont nommées
`createXxxController(service)` et `createXxxRouter(controller)`. Elles
retournent respectivement un objet contenant des `RequestHandler` et un
`Router` Express. Cette forme « factory » est la clé du câblage manuel : elle
permet d'injecter les dépendances depuis `app.ts` sans recourir à un container.

Extrait de [backend/src/api/recipes/recipes.controller.ts:7-20](https://github.com/arthur-lagenebre/recipe-shelter-backend/blob/main/src/api/recipes/recipes.controller.ts#L7) :

```ts
export function createRecipesController(recipeService: RecipeService) {
    return {
        getMyRecipes: asyncHandler(async (req, res) => {
            if (!req.auth) { /* 401 */ return; }
            const pagination = parsePaginationQuery(req.query, 10, 'RECIPES_PAGINATION');
            const result = await recipeService.getMine(req.auth.userId, pagination);
            res.status(200).json(result);
        }),
        // ...
    };
}
```

Les controllers ne contiennent **aucune logique métier**. Ils se contentent de :

1. lire la requête (auth, params, query, body) ;
2. valider/parser le payload via les helpers `parseXxx` du DTO ;
3. appeler le service ;
4. sérialiser la réponse JSON avec le bon code HTTP.

Le wrapper [asyncHandler](https://github.com/arthur-lagenebre/recipe-shelter-backend/blob/main/src/api/http/async-handler.ts#L3) capture les
exceptions des handlers `async` et les transmet à `next(err)`, ce qui les
achemine vers le middleware d'erreur global sans avoir à écrire des `try/catch`
dans chaque controller.

Les routes décrivent l'enchaînement des middlewares pour chaque endpoint.
Exemple [backend/src/api/recipes/recipes.routes.ts:20-35](https://github.com/arthur-lagenebre/recipe-shelter-backend/blob/main/src/api/recipes/recipes.routes.ts#L20) :

```ts
export function createRecipesRouter(controller: RecipesController) {
  const router = Router();
  router.get('/me', requireAuth, controller.getMyRecipes);
  router.post('/', requireAuth, controller.createRecipe);
  router.get('/', optionalAuth, controller.getRecipes);
  router.get('/:slug', optionalAuth, controller.getRecipeBySlug);
  // ...
  return router;
}
```

Les DTO (`*.dto.ts`) regroupent les fonctions de **validation et de normalisation
du payload entrant**. Elles lèvent des `HttpError` 400 quand l'input est
invalide, garantissant que les services métier reçoivent toujours des données
bien formées. Cette discipline rend les services indifférents à la couche
transport.

### 3.2 Couche `services/` — logique métier

Les services contiennent la logique applicative : règles métier, orchestration
de plusieurs repositories, calculs dérivés, contrôles d'autorisation
fonctionnels (différents de l'authentification, traitée en middleware).

Convention : chaque service est une **classe** dont le constructeur reçoit ses
dépendances en paramètres. Aucun service n'importe Express, ni les types
`Request`/`Response`. Cela permet de les tester sans avoir à monter un serveur
HTTP : il suffit d'instancier la classe avec des doublures de repositories.

Exemple — [backend/src/services/recipes/recipes.services.ts:24-25](https://github.com/arthur-lagenebre/recipe-shelter-backend/blob/main/src/services/recipes/recipes.services.ts#L24) :

```ts
export class RecipeService {
    constructor(
      private readonly recipeRepository: RecipeRepository,
      private readonly recipeSlugService: RecipeSlugService
    ) { }
    // ...
}
```

Quand un service a besoin de remonter une erreur HTTP au controller (par
exemple « recette introuvable » ou « accès refusé »), il jette une `HttpError`
construite via les helpers `notFound`, `forbidden`, `badRequest`, etc., depuis
[backend/src/utils/errors.ts](https://github.com/arthur-lagenebre/recipe-shelter-backend/blob/main/src/utils/errors.ts#L12). Le middleware
d'erreur global la traduit ensuite en réponse JSON normalisée.

Certains services orchestrent **plusieurs repositories**. Par exemple
`AuthService` reçoit le `UserRepository` et l'`EmailValidationService` pour
déclencher l'envoi de l'email de validation lors de l'inscription
([backend/src/services/auth/auth.service.ts:22](https://github.com/arthur-lagenebre/recipe-shelter-backend/blob/main/src/services/auth/auth.service.ts#L22)).
`EmailValidationService` reçoit lui-même le `UserRepository`, son propre
repository, le `Mailer` et l'URL du frontend. Cette composition est entièrement
visible dans `app.ts` (voir section 4).

### 3.3 Couche `repositories/` — accès aux données

C'est ici que se concentre **tout le SQL** de l'application. La couche est
structurée autour de quatre fichiers par domaine :

| Fichier                          | Rôle                                                                   |
| -------------------------------- | ---------------------------------------------------------------------- |
| `*.repository.interface.ts`      | Interface TypeScript décrivant le contrat (méthodes attendues).        |
| `*.repository.mysql.ts`          | Implémentation concrète utilisant `mysql2`.                            |
| `*.mapper.ts`                    | Conversion des `Row` SQL (PascalCase) vers les modèles métier (camelCase). |
| `*.types.ts`                     | Types `*Row` (forme SQL) et types métier (forme applicative).          |

Exemple — interface du repository des recettes,
[backend/src/repositories/recipes/recipe.repository.interface.ts:4-17](https://github.com/arthur-lagenebre/recipe-shelter-backend/blob/main/src/repositories/recipes/recipe.repository.interface.ts#L4) :

```ts
export interface RecipeRepository {
    create(input: RecipeInput): Promise<Recipe>;
    updateDraft(input: UpdateRecipeInput): Promise<Recipe>;
    submit(id: number, slug: string): Promise<Recipe>;
    archive(id: number): Promise<boolean>;
    findById(id: number): Promise<Recipe | null>;
    findByUserId(userId: number, pagination: PaginationOptions): Promise<PaginatedResult<RecipeSummary>>;
    searchPublished(userId: number | null, filters: RecipeSearchFilters, pagination: PaginationOptions): Promise<PaginatedResult<RecipeListItem>>;
    findPublishedBySlug(userId: number | null, slug: string): Promise<RecipeDetail | null>;
    // ...
}
```

L'implémentation MySQL est dans
[backend/src/repositories/recipes/recipe.repository.mysql.ts:20-21](https://github.com/arthur-lagenebre/recipe-shelter-backend/blob/main/src/repositories/recipes/recipe.repository.mysql.ts#L20) :

```ts
export class RecipeRepositoryMysql implements RecipeRepository {
    constructor(private readonly db: Pool) { }
    // ...
}
```

**Pourquoi cette séparation interface / implémentation ?**

1. **Testabilité.** Les services dépendent de l'interface, pas de
   l'implémentation MySQL. Les tests unitaires des services
   ([backend/tests/services/](https://github.com/arthur-lagenebre/recipe-shelter-backend/blob/main/tests/services/)) instancient un objet
   stub satisfaisant l'interface (souvent un objet littéral avec les méthodes
   mockées), sans avoir besoin d'une base de données réelle.

2. **Swap possible.** En cas de changement de SGBD (PostgreSQL,
   SQLite pour les tests, in-memory pour un script), il suffit de fournir une
   nouvelle implémentation de l'interface. Le câblage change uniquement dans
   `app.ts`. Aucune ligne de code métier ne bouge.

3. **Documentation explicite.** L'interface liste exactement ce que le
   repository sait faire. Lire `recipe.repository.interface.ts` suffit à
   comprendre l'API de persistance des recettes, sans avoir à parcourir des
   centaines de lignes de SQL.

Les **mappers** ([backend/src/repositories/recipes/recipe.mapper.ts:3](https://github.com/arthur-lagenebre/recipe-shelter-backend/blob/main/src/repositories/recipes/recipe.mapper.ts#L3))
isolent la traduction entre le format SQL (colonnes PascalCase typées
`RecipeRow`) et le format métier (propriétés camelCase typées `Recipe`). Cette
indirection permet de renommer une colonne sans propager le changement à toute
l'application, et garantit que les services manipulent toujours des modèles
typés.

## 4. Câblage manuel dans `app.ts`

Le fichier [backend/src/app.ts](https://github.com/arthur-lagenebre/recipe-shelter-backend/blob/main/src/app.ts#L69) est la **clé de voûte
pédagogique** du projet. Il rend explicite tout ce que des frameworks comme
NestJS ou Spring cachent dans leur container d'injection. Sa lecture suffit à
comprendre l'intégralité du graphe de dépendances de l'application.

L'assemblage se fait en cinq étapes successives dans `createApp()` :

### Étape 1 — middlewares globaux

[backend/src/app.ts:80-82](https://github.com/arthur-lagenebre/recipe-shelter-backend/blob/main/src/app.ts#L80) :

```ts
app.use(cors({ credentials: true, origin: origins }));
app.use(cookieParser());
app.use(express.json());
```

Ordre important : CORS doit s'appliquer avant tout, `cookieParser` doit avoir
peuplé `req.cookies` avant que `requireAuth` ne lise le cookie de session, et
`express.json()` doit avoir désérialisé le body avant que les controllers ne
lisent `req.body`.

Une garde explicite refuse `*` comme origine autorisée
([backend/src/app.ts:77-78](https://github.com/arthur-lagenebre/recipe-shelter-backend/blob/main/src/app.ts#L77)) : avec
`credentials: true`, le wildcard est interdit par la spec CORS et constituerait
une faille de sécurité. Le code lève une erreur au boot plutôt que de laisser
le serveur démarrer dans un état dangereux.

### Étape 2 — instanciation du `Mailer` et des repositories

[backend/src/app.ts:84-98](https://github.com/arthur-lagenebre/recipe-shelter-backend/blob/main/src/app.ts#L84) :

```ts
const mailer = new SmtpMailService(env.smtp);

const adminCommentRepository = new AdminCommentRepositoryMysql(pool);
const adminRecipeRepository = new AdminRecipeRepositoryMysql(pool);
const adminUserRepository = new AdminUserRepositoryMysql(pool);
const categoryRepository = new CategoryRepositoryMysql(pool);
const commentRepository = new CommentRepositoryMysql(pool);
// ... 13 repositories au total
const userRepository = new UserRepositoryMysql(pool);
```

Chaque repository reçoit le même `pool` mysql2. Une seule pool partagée pour
toute l'application : c'est la bonne pratique pour mutualiser les connexions
TCP vers MySQL.

### Étape 3 — configuration du middleware `requireAuth`

[backend/src/app.ts:100](https://github.com/arthur-lagenebre/recipe-shelter-backend/blob/main/src/app.ts#L100) :

```ts
configureAuthUserRepository(userRepository);
```

Le middleware `requireAuth` a besoin de re-vérifier que l'utilisateur extrait
du JWT existe toujours et est actif en base. Plutôt que d'importer
directement le repository (ce qui créerait une dépendance statique difficile à
tester), on lui injecte la dépendance via une fonction de configuration
exécutée une fois au démarrage. Voir
[backend/src/middlewares/require-auth.ts:22](https://github.com/arthur-lagenebre/recipe-shelter-backend/blob/main/src/middlewares/require-auth.ts#L22).

### Étape 4 — instanciation des services

[backend/src/app.ts:102-117](https://github.com/arthur-lagenebre/recipe-shelter-backend/blob/main/src/app.ts#L102) :

```ts
const adminCommentService = new AdminCommentService(adminCommentRepository);
const adminRecipeService  = new AdminRecipeService(recipeRepository, adminRecipeRepository);
const adminUserService    = new AdminUserService(userRepository, adminUserRepository);
const emailValidationService = new EmailValidationService(
    userRepository, emailValidationRepository, mailer, env.http.frontendBaseUrl
);
const authService     = new AuthService(userRepository, emailValidationService);
const categoryService = new CategoryService(categoryRepository);
const commentService  = new CommentService(commentRepository);
const contactService  = new ContactService(mailer);
const equipmentService = new EquipmentService(equipmentRepository);
const favoriteService  = new FavoriteService(favoriteRepository);
const ingredientService = new IngredientService(ingredientRepository);
const passwordResetService = new PasswordResetService(
    userRepository, passwordResetRepository, mailer, env.http.frontendBaseUrl
);
const recipeSlugService = new RecipeSlugService(recipeRepository);
const recipeService = new RecipeService(recipeRepository, recipeSlugService);
const tagService    = new TagService(tagRepository);
const usersService  = new UserService(userRepository, recipeRepository);
```

Chaque service reçoit explicitement les dépendances dont il a besoin. Aucune
résolution automatique, aucun décorateur, aucun fichier de configuration : ce
qui est ici est exactement ce qui s'exécute. Les **dépendances croisées sont
visibles à l'œil nu** : `AuthService` dépend du `UserRepository` ET de
`EmailValidationService`, `RecipeService` dépend du `RecipeRepository` ET du
`RecipeSlugService`, etc.

### Étape 5 — instanciation des controllers et montage des routes

[backend/src/app.ts:119-147](https://github.com/arthur-lagenebre/recipe-shelter-backend/blob/main/src/app.ts#L119) :

```ts
const recipesController = createRecipesController(recipeService);
// ...
app.use('/api/v1/recipes', createRecipesRouter(recipesController));
```

Tous les sous-routeurs sont préfixés par `/api/v1/` ce qui anticipe un
versionnement futur de l'API sans casser les clients existants.

Enfin, les deux middlewares terminaux sont montés :

[backend/src/app.ts:149-150](https://github.com/arthur-lagenebre/recipe-shelter-backend/blob/main/src/app.ts#L149) :

```ts
app.use(notFound);
app.use(errorHandler);
```

`notFound` génère une 404 normalisée pour toute route non matchée,
`errorHandler` est le **point unique de sérialisation des erreurs** vers une
réponse JSON `{ error: { message, code } }`.

### Comparaison avec un framework à conteneur DI

Avec NestJS, Spring ou équivalent, le même graphe serait construit par
décorateurs (`@Injectable`, `@Inject`, `@Module`) et résolu à l'exécution par
un container qui scannerait les métadonnées. Le développeur ne voit plus le
graphe : il déclare des intentions, le framework câble.

Le choix inverse — **câblage manuel explicite** — a été retenu pour Recipe
Shelter pour trois raisons pédagogiques :

1. **Lisibilité immédiate.** Un examinateur peut lire `app.ts` en deux minutes
   et comprendre l'architecture complète.
2. **Démonstration de l'inversion de dépendance.** L'absence de magie rend
   visible le fait que les services dépendent d'**interfaces** et que les
   implémentations leur sont **fournies de l'extérieur** — la définition même
   de l'IoC.
3. **Aucune dépendance cachée.** Le `package.json` ne contient aucun framework
   d'injection ; il n'y a donc rien à apprendre d'autre que TypeScript et
   Express pour modifier l'application.

À une échelle bien plus grande (centaines de services), un container DI
deviendrait préférable pour éviter un `app.ts` ingérable. À l'échelle du
projet (≈15 domaines, ≈25 services), le câblage manuel reste lisible et
parfaitement adapté.

## 5. Domaines fonctionnels

L'API expose **quinze sous-routeurs** sous le préfixe `/api/v1/`. Chaque domaine
correspond à un sous-dossier dans `api/`, `services/` et `repositories/`.

| Préfixe                              | Domaine                  | Description                                                            |
| ------------------------------------ | ------------------------ | ---------------------------------------------------------------------- |
| `/api/v1/auth`                       | Authentification         | Inscription, login, logout, mot de passe oublié, validation d'email.   |
| `/api/v1/users`                      | Utilisateurs             | Profil de l'utilisateur courant, données publiques d'un auteur.        |
| `/api/v1/recipes`                    | Recettes                 | CRUD recettes, recherche publique, brouillons, soumission, archivage.  |
| `/api/v1/recipes/:recipeId/comments` | Commentaires de recette  | Sous-routeur imbriqué : commentaires liés à une recette donnée.        |
| `/api/v1/comments`                   | Commentaires             | Endpoints transverses sur les commentaires (suppression par auteur).   |
| `/api/v1/favorites`                  | Favoris                  | Ajout/retrait d'une recette aux favoris de l'utilisateur.              |
| `/api/v1/categories`                 | Catégories               | Référentiel des catégories de recettes.                                |
| `/api/v1/tags`                       | Tags                     | Référentiel des tags applicables aux recettes.                         |
| `/api/v1/ingredients`                | Ingrédients              | Référentiel des ingrédients.                                           |
| `/api/v1/equipments`                 | Équipements              | Référentiel du matériel de cuisine.                                    |
| `/api/v1/contact`                    | Contact                  | Formulaire de contact public envoyé par email à l'administrateur.      |
| `/api/v1/health`                     | Santé                    | Endpoint de monitoring pour les sondes Kubernetes / supervision.       |
| `/api/v1/admin/recipes`              | Admin — recettes         | Modération : liste des recettes soumises, validation, rejet.           |
| `/api/v1/admin/comments`             | Admin — commentaires     | Modération des commentaires signalés.                                  |
| `/api/v1/admin/users`                | Admin — utilisateurs     | Gestion des comptes, bannissement, changement de rôle.                 |

Les trois domaines `admin/*` partagent une caractéristique : leurs routeurs
sont systématiquement protégés par le middleware `requireAdmin` (en plus de
`requireAuth`), qui vérifie que `req.auth.roleId === 1`.

Le sous-routeur `comments` imbriqué sous `recipes/:recipeId/comments` est
monté **avant** le routeur `recipes/` racine
([backend/src/app.ts:144-145](https://github.com/arthur-lagenebre/recipe-shelter-backend/blob/main/src/app.ts#L144)) pour qu'Express
matche la route la plus spécifique en premier.

## 6. Flux d'une requête

Suivons un cas concret : un utilisateur authentifié crée une recette via
`POST /api/v1/recipes`. Voici le chemin parcouru par la requête, du socket
TCP jusqu'à la base de données et retour.

```
Client (Angular)
    │
    │  POST /api/v1/recipes
    │  Cookie: rs_session=<jwt>
    │  Content-Type: application/json
    │  Body: { title, description, ingredients, steps, ... }
    ▼
┌─────────────────────────────────────────────────────────────────┐
│  Express — pipeline de middlewares globaux                      │
├─────────────────────────────────────────────────────────────────┤
│  1. cors          → valide l'origine, écrit Access-Control-*    │
│  2. cookieParser  → req.cookies.rs_session = '<jwt>'            │
│  3. express.json  → req.body = { title, ... }                   │
└─────────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────────┐
│  Router /api/v1/recipes → POST '/'                              │
├─────────────────────────────────────────────────────────────────┤
│  4. requireAuth                                                 │
│     - lit le JWT dans req.cookies.rs_session                    │
│     - jwt.verify(token, JWT_SECRET)                             │
│     - parseAuthPayload(payload)                                 │
│     - userRepository.findById(userId) → vérifie status='active' │
│     - req.auth = { userId, username, roleId, status }           │
│  5. controller.createRecipe (asyncHandler)                      │
└─────────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────────┐
│  Controller — RecipesController.createRecipe                    │
├─────────────────────────────────────────────────────────────────┤
│  6. parseCreateRecipeBody(req.body)                             │
│     → lève HttpError 400 si le payload est invalide             │
│  7. recipeService.create(req.auth.userId, body)                 │
└─────────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────────┐
│  Service — RecipeService.create                                 │
├─────────────────────────────────────────────────────────────────┤
│  8. recipeSlugService.createDraftSlug(userId)                   │
│     → calcule un slug unique pour le brouillon                  │
│  9. normalizeCreateRecipeInput(...)                             │
│     → trim, valeurs par défaut, dédoublonnage des tags          │
│ 10. recipeRepository.create(normalizedInput)                    │
└─────────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────────┐
│  Repository — RecipeRepositoryMysql.create                      │
├─────────────────────────────────────────────────────────────────┤
│ 11. connection.beginTransaction()                               │
│ 12. INSERT INTO Recipes ...                                     │
│ 13. INSERT INTO RecipeIngredients ... (N lignes)                │
│ 14. INSERT INTO RecipeSteps ... (N lignes)                      │
│ 15. INSERT INTO RecipeEquipments ... (N lignes)                 │
│ 16. INSERT INTO RecipeTags ... (N lignes)                       │
│ 17. connection.commit()  (rollback en cas d'erreur)             │
│ 18. findById(recipeId) → SELECT + mapRecipe(row)                │
└─────────────────────────────────────────────────────────────────┘
    │
    ▼  Recipe (modèle métier)
┌─────────────────────────────────────────────────────────────────┐
│  Controller — sérialisation                                     │
├─────────────────────────────────────────────────────────────────┤
│ 19. res.status(201).json(result)                                │
└─────────────────────────────────────────────────────────────────┘
    │
    ▼
Client reçoit 201 Created + Recipe au format JSON
```

En cas d'exception levée à n'importe quelle étape, `asyncHandler` la transmet
à `next(err)`, ce qui court-circuite la chaîne et invoque
[errorHandler](https://github.com/arthur-lagenebre/recipe-shelter-backend/blob/main/src/middlewares/error-handler.ts#L12) qui produit la
réponse `{ error: { message, code } }` avec le bon code HTTP.

Pour la visualisation graphique de ce pipeline, voir
[diagrams/g2-pipeline-middleware.md](diagrams/g2-pipeline-middleware.md).
Pour le diagramme C4 de l'architecture globale, voir
[diagrams/g1-architecture-c4.md](diagrams/g1-architecture-c4.md).

## 7. Persistance

### 7.1 Pool MySQL

Le module [backend/src/db/pool.ts](https://github.com/arthur-lagenebre/recipe-shelter-backend/blob/main/src/db/pool.ts#L5) instancie une
unique pool `mysql2/promise` partagée par tous les repositories :

```ts
export const pool = mysql.createPool({
  host: env.db.host,
  port: env.db.port,
  user: env.db.user,
  password: env.db.password,
  database: env.db.name,
  waitForConnections: true,
  connectionLimit: env.db.connectionLimit,
  enableKeepAlive: true,
  keepAliveInitialDelay: 0,
  timezone: 'Z',
  namedPlaceholders: true
});
```

Points notables :

- `connectionLimit` paramétrable via `DB_CONNECTION_LIMIT` (défaut 10).
- `waitForConnections: true` : si la pool est saturée, les requêtes attendent
  une connexion plutôt que d'échouer immédiatement.
- `enableKeepAlive: true` : évite que les firewalls coupent les connexions
  inactives.
- `timezone: 'Z'` : force UTC pour éviter les ambiguïtés de fuseau horaire
  entre l'application et MySQL.
- `namedPlaceholders: true` : autorise la syntaxe `:nom` dans les requêtes
  préparées (utilisé pour les requêtes longues à nombreux paramètres).

### 7.2 Helper `query`

[backend/src/db/query.ts:21](https://github.com/arthur-lagenebre/recipe-shelter-backend/blob/main/src/db/query.ts#L21) enveloppe `pool.execute`
avec :

- mesure du temps d'exécution (`performance.now()`) ;
- log debug systématique, log warning au-delà de 200 ms (`SLOW_QUERY_MS`) ;
- traduction des erreurs SQL en `DbError` typé via
  [toDbError](https://github.com/arthur-lagenebre/recipe-shelter-backend/blob/main/src/db/errors.ts#L15) ;
- support d'une connexion explicite (pour participer à une transaction).

### 7.3 Transactions

Le helper [transaction](https://github.com/arthur-lagenebre/recipe-shelter-backend/blob/main/src/db/transaction.ts#L5) standardise le pattern
`beginTransaction / commit / rollback` :

```ts
export async function transaction<T>(fn: (tx: PoolConnection) => Promise<T>): Promise<T> {
    const conn = await pool.getConnection();
    try {
        await conn.beginTransaction();
        const result = await fn(conn);
        await conn.commit();
        return result;
    } catch (err) {
        await conn.rollback();
        throw err;
    } finally {
        conn.release();
    }
}
```

Les repositories qui ont besoin d'écrire sur plusieurs tables de manière
atomique (création d'une recette avec ses ingrédients, étapes, équipements,
tags) gèrent leur propre transaction inline avec
`connection.beginTransaction()` pour pouvoir intercaler des requêtes
intermédiaires et utiliser les `insertId` au fil de l'eau — voir
[backend/src/repositories/recipes/recipe.repository.mysql.ts:23-56](https://github.com/arthur-lagenebre/recipe-shelter-backend/blob/main/src/repositories/recipes/recipe.repository.mysql.ts#L23).

### 7.4 Schéma de base de données

Le schéma relationnel complet est documenté sous forme graphique dans
[database/Schema.svg](database/Schema.svg).
Les principales tables sont :

- `Users`, `Roles` — comptes utilisateurs et rôles applicatifs ;
- `Recipes`, `RecipeIngredients`, `RecipeSteps`, `RecipeEquipments`, `RecipeTags` —
  cœur du modèle, une recette + ses dépendances ;
- `Categories`, `Tags`, `Ingredients`, `Equipments` — référentiels ;
- `Comments`, `Favorites` — interactions communautaires ;
- `EmailValidations`, `PasswordResets` — tokens à durée de vie limitée pour les
  workflows d'authentification.

Les contraintes d'intégrité (clés étrangères, `ON DELETE CASCADE`, index
uniques sur slug et email) sont déclarées au niveau du schéma SQL et servent
de filet de sécurité en complément des règles applicatives.

## 8. Envoi d'emails

L'envoi d'emails est centralisé dans une **interface `Mailer`**
([backend/src/services/mail/mail.types.ts](https://github.com/arthur-lagenebre/recipe-shelter-backend/blob/main/src/services/mail/mail.types.ts))
implémentée par [SmtpMailService](https://github.com/arthur-lagenebre/recipe-shelter-backend/blob/main/src/services/mail/mail.service.ts#L19)
qui utilise Nodemailer en arrière-plan.

Trois services métier consomment le `Mailer` :

| Service                       | Email envoyé                       | Déclencheur                                    |
| ----------------------------- | ---------------------------------- | ---------------------------------------------- |
| `EmailValidationService`      | Email de validation de compte      | Inscription, ré-envoi sur demande              |
| `PasswordResetService`        | Email de réinitialisation          | Formulaire « mot de passe oublié »             |
| `PasswordResetService`        | Email de confirmation de changement| Changement effectif du mot de passe            |
| `ContactService`              | Email de contact à l'administrateur| Soumission du formulaire `/api/v1/contact`     |

Les emails applicatifs (validation, mot de passe) utilisent l'expéditeur
`SMTP_FROM`. L'email de contact utilise également `SMTP_FROM` comme expéditeur
mais place l'adresse du visiteur en `Reply-To` pour permettre une réponse
directe.

Le service vérifie la complétude de la configuration SMTP avant tout envoi et
lève une `HttpError` 500 normalisée (`MAIL_SEND_FAILED` ou `CONTACT_SEND_FAILED`)
si une variable obligatoire manque. Cela évite de découvrir un problème de
configuration uniquement à l'exécution.

## 9. Authentification

L'authentification repose sur un **JWT signé** transporté dans un **cookie
HttpOnly + Secure + SameSite**. Le détail des règles de sécurité (politique
de mot de passe, rate limiting, durée de vie des tokens, headers cookies) est
documenté dans [securite.md](securite.md).

En synthèse :

- À l'inscription, `AuthService.register`
  ([backend/src/services/auth/auth.service.ts:39](https://github.com/arthur-lagenebre/recipe-shelter-backend/blob/main/src/services/auth/auth.service.ts#L39))
  valide le payload, hashe le mot de passe avec bcrypt (`BCRYPT_COST=12`),
  crée l'utilisateur en statut `inactive` et délègue à
  `EmailValidationService` l'envoi de l'email d'activation.
- Au login, `AuthService.login` vérifie le hash bcrypt et le statut du compte
  (`inactive`, `banned` → 401), puis signe un JWT contenant
  `{ sub, username, roleId, status }`.
- Le controller pose le cookie de session via les options définies dans
  `env.auth.sessionCookie*` (HttpOnly, Secure en production, SameSite=lax par
  défaut, expiration alignée sur le JWT).
- Sur chaque requête authentifiée, [requireAuth](https://github.com/arthur-lagenebre/recipe-shelter-backend/blob/main/src/middlewares/require-auth.ts#L69)
  lit le cookie, vérifie le JWT, **re-charge l'utilisateur depuis la base** et
  contrôle qu'il est toujours `active`. Cette double vérification permet de
  révoquer immédiatement un utilisateur banni sans attendre l'expiration du
  JWT.
- [optionalAuth](https://github.com/arthur-lagenebre/recipe-shelter-backend/blob/main/src/middlewares/require-auth.ts#L94) est une variante
  qui peuple `req.auth` si le cookie est valide mais n'échoue pas s'il est
  absent — utilisée pour les routes publiques qui adaptent leur réponse selon
  que l'utilisateur est connecté ou non (ex. marquer les recettes en favori).

## 10. Configuration

Toute la configuration runtime de l'application transite par un **objet `env`
unique et typé** exporté depuis [backend/src/utils/env.ts](https://github.com/arthur-lagenebre/recipe-shelter-backend/blob/main/src/utils/env.ts#L71).
Aucun module n'accède directement à `process.env` en dehors de ce fichier.

### 10.1 Lecture typée et fallbacks

Le module définit des fonctions utilitaires qui convertissent les chaînes
brutes de `process.env` en valeurs typées avec un fallback explicite :

- `readNumber(value, fallback)` — entier, retombe sur `fallback` si invalide.
- `readBoolean(value, fallback)` — accepte `1/0/true/false/yes/no/on/off`.
- `readString(value, fallback)` — non vide après trim.
- `readOptionalString(value)` — `undefined` si absent.
- `readSameSite(value, fallback)` — restreint à `'strict' | 'lax' | 'none'`.
- `readDurationMs(value, fallback)` — parse `7d`, `30m`, `500ms`, etc.

Cette discipline garantit qu'un `env.auth.bcryptCost` est **toujours un
`number`**, pas un `string | undefined` qu'il faudrait re-vérifier dans chaque
appel à bcrypt.

### 10.2 Variables d'environnement principales

| Variable                                | Défaut                              | Rôle                                                    |
| --------------------------------------- | ----------------------------------- | ------------------------------------------------------- |
| `NODE_ENV`                              | `development`                       | Active des comportements production (cookie secure).    |
| `PORT`                                  | `3000`                              | Port d'écoute HTTP.                                     |
| `CORS_ALLOWED_ORIGINS`                  | `http://localhost:4200,...`         | Allow-list CSV d'origines (refus de `*`).               |
| `FRONTEND_BASE_URL`                     | `http://localhost:4200`             | Base URL utilisée dans les liens des emails.            |
| `DB_HOST`, `DB_PORT`, `DB_USER`, ...    | `127.0.0.1`, `3306`, `root`, ...    | Connexion MySQL.                                        |
| `DB_CONNECTION_LIMIT`                   | `10`                                | Taille de la pool de connexions.                        |
| `JWT_SECRET`                            | **obligatoire** (sinon throw)       | Clé de signature des JWT.                               |
| `JWT_EXPIRES_IN`                        | `7d`                                | Durée de vie du JWT (format `7d`, `30m`, ...).          |
| `AUTH_SESSION_COOKIE_NAME`              | `rs_session`                        | Nom du cookie de session.                               |
| `AUTH_SESSION_COOKIE_SAME_SITE`         | `lax`                               | Politique SameSite (`strict`, `lax`, `none`).           |
| `AUTH_SESSION_COOKIE_SECURE`            | `true` en prod                      | Flag Secure sur le cookie.                              |
| `BCRYPT_COST`                           | `12`                                | Coût bcrypt (10–14 recommandé).                         |
| `AUTH_RATE_LIMIT_MAX_ATTEMPTS`          | `5`                                 | Nombre max de tentatives par fenêtre (login, register). |
| `AUTH_RATE_LIMIT_WINDOW_MS`             | `900000` (15 min)                   | Fenêtre du rate limiter.                                |
| `SMTP_HOST`, `SMTP_PORT`, `SMTP_USER`...| vide                                | Configuration SMTP (validée au moment de l'envoi).      |
| `CONTACT_RECIPIENT_EMAIL`               | vide                                | Destinataire du formulaire de contact.                  |

### 10.3 Validation au boot

Deux validations sont effectuées **au démarrage** plutôt qu'à l'usage :

1. **`JWT_SECRET` absent** → `throw new Error('JWT_SECRET is required')` au
   chargement du module env, le serveur ne démarre pas.
2. **`CORS_ALLOWED_ORIGINS` contient `*` avec `credentials: true`** → throw
   dans `createApp`, le serveur ne démarre pas.

Ces deux gardes interdisent de lancer l'application dans un état non
sécurisable. Les autres variables ont des défauts raisonnables pour le
développement, et la validation SMTP est différée au premier envoi d'email
(pour permettre un démarrage sans SMTP en dev local).

## 11. Gestion des erreurs

Le projet utilise une classe d'erreur applicative unique,
[HttpError](https://github.com/arthur-lagenebre/recipe-shelter-backend/blob/main/src/utils/errors.ts#L1) (et non `AppError`), qui encapsule
trois informations :

- `status: number` — code HTTP (400, 401, 403, 404, 409, 500) ;
- `message: string` — message humain ;
- `code?: string` — code applicatif stable (ex. `AUTH_INVALID_CREDENTIALS`,
  `RECIPES_NOT_FOUND`) que le frontend peut matcher pour afficher un message
  localisé.

Des helpers `badRequest`, `unauthorized`, `forbidden`, `notFound`, `conflict`,
`internalError` couvrent les cas usuels.

Le middleware [errorHandler](https://github.com/arthur-lagenebre/recipe-shelter-backend/blob/main/src/middlewares/error-handler.ts#L12) est
le **point unique de sérialisation** des erreurs. Toute exception levée dans
un controller ou un service, propagée par `next(err)`, est traduite en :

```json
{ "error": { "message": "...", "code": "..." } }
```

avec le bon code HTTP. Les erreurs 5xx sont loggées avec leur stack ; les 4xx
sont retournées sans bruit (elles correspondent à des comportements clients
attendus).

Le middleware [notFound](https://github.com/arthur-lagenebre/recipe-shelter-backend/blob/main/src/middlewares/not-found.ts#L3) intercepte les
routes non matchées et émet une 404 normalisée (`ROUTE_NOT_FOUND`).

Le catalogue exhaustif des codes d'erreur applicatifs (avec leur signification
et la couche qui les émet) est documenté dans
[errors.md](errors.md).

## 12. Tests

Les tests automatisés vivent dans [backend/tests/](https://github.com/arthur-lagenebre/recipe-shelter-backend/blob/main/tests/) et
reproduisent la structure de `src/` :

```
backend/tests/
├── api/
│   ├── admin/          (3 fichiers : admin.comments|recipes|users.dto.test.ts)
│   ├── auth/           (auth.controller + auth.dto)
│   ├── comments/       (comments.dto)
│   ├── contact/        (contact.dto)
│   ├── recipes/        (recipes.dto)
│   └── users/          (users.dto)
├── middlewares/        (error-handler, not-found, rate-limiter,
│                        require-admin, require-auth)
├── repositories/
│   ├── comments/       (comments.mapper)
│   └── recipes/        (recipe.mapper)
├── services/
│   ├── admin/          (admin.comments|recipes|users.service)
│   ├── auth/           (auth, email-validation, password-policy,
│   │                    password-reset)
│   ├── comments/       (comments.service)
│   ├── recipes/        (recipe-slug, recipes.service)
│   └── users/          (users.service)
└── utils/
    ├── pagination.test.ts
    └── security/password-reset-token.test.ts
```

Soit **29 fichiers de tests** couvrant les trois couches. Les services sont
testés en isolation avec des doublures de repositories ; les middlewares sont
testés avec des objets `req`/`res` simulés ; les mappers sont testés sur
des lignes SQL fabriquées à la main ; les DTO sont testés sur des payloads
valides et invalides.

Le **test runner est le runner natif de Node**
(`node --import tsx --test "tests/**/*.test.ts"`, voir
[backend/package.json:14](https://github.com/arthur-lagenebre/recipe-shelter-backend/blob/main/package.json#L14)), exécuté via `tsx` pour
transpiler le TypeScript à la volée. Aucune dépendance externe (Jest, Mocha,
Vitest) : seul le runtime Node est requis. C'est cohérent avec la philosophie
« from scratch » du projet et avec la volonté de minimiser la surface
d'apprentissage de la stack.

Les tests E2E (parcours complets avec une base de données réelle) ne sont
pas dans ce répertoire : ils relèvent de la stratégie de test du frontend
(Playwright/Cypress) et sont documentés séparément.

---

**Documents associés** :

- [diagrams/g1-architecture-c4.md](diagrams/g1-architecture-c4.md) — diagrammes C4 contexte/conteneur/composant.
- [diagrams/g2-pipeline-middleware.md](diagrams/g2-pipeline-middleware.md) — diagramme de séquence du pipeline de middlewares.
- [securite.md](securite.md) — sécurité applicative en détail (JWT, cookies, bcrypt, rate limiting, CORS).
- [errors.md](errors.md) — catalogue exhaustif des codes d'erreur applicatifs.
- `CAHIER_DES_CHARGES.md` — exigences fonctionnelles et non fonctionnelles (à la racine du workspace, hors du site).
