# Diagramme de classes — Architecture backend

> Architecture en couches du backend Node.js/TypeScript, basée sur le pattern
> **Repository** avec inversion de dépendances.
>
> Deux vues : la structure générique du pattern, puis une instance concrète
> sur le domaine **Recipe** (le plus représentatif).

---

## 1. Pattern architectural (vue générique)

```mermaid
classDiagram
    direction TB

    class Express_Router {
        <<framework>>
        +get(path, handler)
        +post(path, handler)
        +use(middleware)
    }

    class Middleware {
        <<interface>>
        +requireAuth(req, res, next)
        +requireAdmin(req, res, next)
        +rateLimiter(req, res, next)
        +errorHandler(err, req, res, next)
    }

    class Controller {
        <<factory function>>
        -service: Service
        +handle(req, res): Promise
    }

    class Service {
        <<business logic>>
        -repository: IRepository
        +executeUseCase(input): Promise
        -validateBusinessRule(): void
    }

    class IRepository {
        <<interface>>
        +findById(id): Promise
        +create(input): Promise
        +update(input): Promise
    }

    class RepositoryMysql {
        <<implementation>>
        -pool: MysqlConnectionPool
        +findById(id): Promise
        +create(input): Promise
    }

    class DTO {
        <<validation>>
        +parse(body): TypedInput
    }

    Express_Router --> Middleware : applique
    Express_Router --> Controller : route vers
    Controller --> DTO : valide avec
    Controller --> Service : appelle
    Service --> IRepository : dépend de
    RepositoryMysql ..|> IRepository : implémente
```

**Points-clés à défendre**
- **Inversion de dépendances** : les services ne dépendent que de l'interface du repository, pas de l'implémentation MySQL. Permet de remplacer la BDD ou de mocker en tests.
- **Injection par constructeur** : les services reçoivent leur repository en paramètre du constructeur. Le câblage se fait une fois au démarrage dans `app.ts`.
- **Controllers en factory functions** : `createXxxController(service)` retourne un objet de handlers Express → injection sans framework DI, fidèle au "from scratch".
- **DTO séparés** : la validation des inputs HTTP est isolée des entités métier → couplage faible entre la couche HTTP et la couche service.

---

## 2. Instance concrète — domaine Recipe

```mermaid
classDiagram
    direction TB

    class RecipesRouter {
        <<express.Router>>
        +GET /api/v1/recipes
        +POST /api/v1/recipes
        +GET /api/v1/recipes/:slug
    }

    class RecipesController {
        <<factory>>
        +getRecipes(req, res)
        +createRecipe(req, res)
        +getRecipe(req, res)
        +updateRecipe(req, res)
        +submitRecipe(req, res)
        +archiveRecipe(req, res)
    }

    class CreateRecipeBody {
        <<DTO>>
        +title: string
        +description: string
        +categoryId: number
        +ingredients: RecipeIngredientInput[]
        +steps: RecipeStepInput[]
        +parse(body): CreateRecipeBody
    }

    class RecipeService {
        -recipeRepository: RecipeRepository
        -slugService: RecipeSlugService
        +create(userId, input) Recipe
        +get(id, auth) Recipe
        +updateDraft(id, auth, input) Recipe
        +submit(id, auth) Recipe
        +archive(id, auth) bool
        +getRecentPublished(userId, limit) RecipeListItem[]
        -requireViewableRecipe()
        -requireEditableRecipe()
        -canViewRecipe()
        -canEditRecipe()
    }

    class RecipeSlugService {
        -recipeRepository: RecipeRepository
        +createDraftSlug(userId) string
        +createPublicSlug(title) string
    }

    class RecipeRepository {
        <<interface>>
        +create(input) Recipe
        +updateDraft(input) Recipe
        +submit(id, slug) Recipe
        +archive(id) bool
        +findById(id) Recipe
        +findPublished(userId, pagination) PaginatedResult
        +searchPublished(userId, filters, pagination) PaginatedResult
        +findPublishedBySlug(userId, slug) RecipeDetail
        +existsBySlug(slug) bool
    }

    class RecipeRepositoryMysql {
        -pool: MysqlPool
        +create(input) Recipe
        +findPublished(userId, pagination) PaginatedResult
        +searchPublished(userId, filters, pagination) PaginatedResult
    }

    class RecipeMapper {
        <<utility>>
        +toRecipe(row) Recipe
        +toRecipeDetail(rows) RecipeDetail
        +toRecipeListItem(row) RecipeListItem
    }

    class RequireAuth {
        <<middleware>>
        +requireAuth(req, res, next)
    }

    RecipesRouter --> RequireAuth : protège
    RecipesRouter --> RecipesController : route
    RecipesController ..> CreateRecipeBody : valide
    RecipesController --> RecipeService : utilise
    RecipeService --> RecipeRepository : dépend de
    RecipeService --> RecipeSlugService : utilise
    RecipeSlugService --> RecipeRepository : dépend de
    RecipeRepositoryMysql ..|> RecipeRepository : implémente
    RecipeRepositoryMysql --> RecipeMapper : utilise
```

**Q/R probable du jury**
- **"Pourquoi cette séparation interface/implémentation ?"** → Permet d'écrire les tests unitaires du service avec un repository en mémoire (cf. `tests/services/recipes/recipes.service.test.ts`), sans démarrer MySQL.
- **"Et si demain vous changez de SGBD ?"** → Il suffit d'écrire `RecipeRepositoryPostgres implements RecipeRepository`, sans toucher au service.
- **"Pourquoi un `RecipeSlugService` séparé ?"** → Single Responsibility : la génération du slug (avec vérification d'unicité en BDD) est une logique métier réutilisable, distincte du CRUD recette.
