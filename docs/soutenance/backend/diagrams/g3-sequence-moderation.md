# G3 — Séquence : modération d'une recette

Une recette publiée par un membre de Recipe Shelter suit un cycle de vie contrôlé. L'auteur la crée d'abord en brouillon (`draft`), peut la modifier librement, puis la soumet pour validation (`pending`). Un modérateur consulte la file d'attente, ouvre la fiche complète, et choisit de publier (`published`) ou de refuser (`rejected`) la recette avec une raison. Une recette publiée ou refusée peut ensuite être archivée (`archived`) par l'auteur ou un admin. Le diagramme ci-dessous trace ces transitions de bout en bout.

```mermaid
sequenceDiagram
    autonumber
    actor Auteur
    participant Frontend
    participant RecipesController as RecipesController<br/>(POST /api/v1/recipes)
    participant RecipeService
    participant SlugSvc as RecipeSlugService
    participant Repo as RecipeRepository<br/>(MySQL)
    participant AdminCtrl as AdminRecipesController<br/>(/admin/recipes)
    participant AdminSvc as AdminRecipeService
    actor Admin

    rect rgb(245,245,245)
    Note over Auteur,Repo: 1. Création du brouillon
    Auteur->>Frontend: Remplit le formulaire
    Frontend->>RecipesController: POST /api/v1/recipes (JWT)
    RecipesController->>RecipeService: create(userId, input)
    RecipeService->>SlugSvc: createDraftSlug(userId)
    SlugSvc-->>RecipeService: "draft_<userId>_<ts>_<rand>"
    RecipeService->>Repo: INSERT Recipes(status='draft', slug=draft_...)
    Repo-->>RecipeService: Recipe (id, status='draft')
    RecipeService-->>Frontend: 201 Created
    end

    rect rgb(245,245,245)
    Note over Auteur,Repo: 2. Soumission pour modération
    Auteur->>Frontend: Clique "Soumettre"
    Frontend->>RecipesController: POST /api/v1/recipes/me/:id/submit
    RecipesController->>RecipeService: submit(recipeId, auth)
    RecipeService->>Repo: findById(recipeId)
    Repo-->>RecipeService: recipe (draft|rejected)
    RecipeService->>SlugSvc: createPublicSlug(title)
    SlugSvc->>Repo: existsBySlug(candidate) (boucle)
    SlugSvc-->>RecipeService: slug unique
    RecipeService->>Repo: UPDATE status='pending', SubmittedAt=NOW, Slug=public
    Repo-->>Frontend: 200 OK
    end

    rect rgb(245,245,245)
    Note over Admin,Repo: 3. Consultation de la file
    Admin->>Frontend: Ouvre l'espace modération
    Frontend->>AdminCtrl: GET /api/v1/admin/recipes/pending
    AdminCtrl->>AdminSvc: getPendingRecipesForAdmin()
    AdminSvc->>Repo: SELECT WHERE Status='pending'
    Repo-->>AdminCtrl: RecipePending[]
    AdminCtrl-->>Frontend: 200 OK (liste)
    Frontend->>AdminCtrl: GET /api/v1/admin/recipes/:id
    AdminCtrl->>AdminSvc: getRecipeForAdmin(id)
    AdminSvc->>Repo: fiche complète (ingrédients, étapes, tags, équipements)
    Repo-->>Frontend: 200 OK (RecipeAdmin)
    end

    alt 4a. Approbation
        Admin->>Frontend: Clique "Approuver"
        Frontend->>AdminCtrl: POST /api/v1/admin/recipes/:id/approve
        AdminCtrl->>AdminSvc: approve(id, adminUserId)
        AdminSvc->>Repo: findById (vérifie status='pending')
        AdminSvc->>Repo: UPDATE status='published', PublishedAt=NOW, ModeratedByUserId=adminId
        Repo-->>Frontend: 200 OK { ok: true }
        Note over AdminSvc,Repo: Pas d'email auteur dans l'impl. actuelle
    else 4b. Refus
        Admin->>Frontend: Saisit la raison + clique "Refuser"
        Frontend->>AdminCtrl: POST /api/v1/admin/recipes/:id/reject { rejectionReason }
        AdminCtrl->>AdminSvc: reject(id, adminUserId, rejectionReason)
        AdminSvc->>Repo: findById (vérifie status='pending')
        AdminSvc->>Repo: UPDATE status='rejected', ModeratedAt=NOW, ModeratedByUserId, RejectionReason
        Repo-->>Frontend: 200 OK { ok: true }
        Note over AdminSvc,Repo: Pas d'envoi d'email auteur<br/>(à prévoir évolution)
    end
```

## Statuts d'une recette

| Statut | Déclencheur | Modifiable par auteur | Visible par public |
|---|---|---|---|
| `draft` | `POST /recipes` (création) | Oui (`PATCH /recipes/me/:id`) | Non |
| `pending` | `POST /recipes/me/:id/submit` | Non | Non (file admin uniquement) |
| `published` | `POST /admin/recipes/:id/approve` | Non (relecture admin) | Oui |
| `rejected` | `POST /admin/recipes/:id/reject` | Oui — peut corriger et resoumettre | Non |
| `archived` | `POST /recipes/me/:id/archive` ou `POST /admin/recipes/:id/archive` | Non | Non |

Note : la table `Recipes` porte aussi les colonnes `SubmittedAt`, `ModeratedAt`, `ModeratedByUserId`, `PublishedAt`, `ArchivedAt`, `RejectionReason` qui tracent l'historique de modération (cf. `admin.recipe.repository.mysql.ts:38`).

## Endpoints utilisés

- `POST /api/v1/recipes` — `recipes.routes.ts:24` → `RecipesController.createRecipe` (`recipes.controller.ts:22`)
- `POST /api/v1/recipes/me/:id/submit` — `recipes.routes.ts:31` → `RecipesController.submitRecipe` (`recipes.controller.ts:99`)
- `GET  /api/v1/admin/recipes/pending` — `admin.recipes.routes.ts:21` → `AdminRecipesController.listPendingRecipes` (`admin.recipes.controller.ts:9`)
- `GET  /api/v1/admin/recipes/:id` — `admin.recipes.routes.ts:23` → `AdminRecipesController.getRecipeAdmin` (`admin.recipes.controller.ts:19`)
- `POST /api/v1/admin/recipes/:id/approve` — `admin.recipes.routes.ts:24` → `AdminRecipesController.approveRecipe` (`admin.recipes.controller.ts:26`)
- `POST /api/v1/admin/recipes/:id/reject` — `admin.recipes.routes.ts:25` → `AdminRecipesController.rejectRecipe` (`admin.recipes.controller.ts:34`)

Les routes admin sont protégées par la chaîne `requireAuth` puis `requireAdmin` (cf. `admin.recipes.routes.ts:21-27`).
