# Diagramme de séquence — Cycle de vie d'une recette

> Couvre le scénario complet : un utilisateur crée une recette brouillon,
> la complète, la soumet à modération, puis un administrateur la publie
> ou la rejette.
>
> Met en évidence la **machine à états** de la recette
> (`draft → pending → published | rejected → archived`).

---

## Vue d'ensemble — machine à états

```mermaid
stateDiagram-v2
    [*] --> draft : POST /recipes (createRecipe)
    draft --> draft : PUT /recipes/:id (updateDraft)
    draft --> pending : POST /recipes/:id/submit (submitRecipe)
    pending --> published : POST /admin/recipes/:id/approve
    pending --> rejected : POST /admin/recipes/:id/reject
    rejected --> draft : (auteur peut re-éditer)
    published --> archived : DELETE /recipes/:id (archiveRecipe)
    published --> archived : POST /admin/recipes/:id/archive
    archived --> [*]

    note right of draft
        Slug temporaire
        Visible uniquement
        par son auteur
    end note

    note right of pending
        Slug temporaire
        Verrouillé pour modification
        Visible par les admins
    end note

    note right of published
        Slug public définitif
        Visible par tous
        Indexable
    end note
```

---

## Création + soumission par l'utilisateur

```mermaid
sequenceDiagram
    autonumber
    actor U as Auteur (User connecté)
    participant FE as Front Angular<br/>recipe-form.ts
    participant API as POST /api/v1/recipes
    participant AUTH as requireAuth
    participant CTRL as RecipesController.createRecipe
    participant DTO as CreateRecipeBody
    participant SVC as RecipeService.create
    participant SLUG as RecipeSlugService
    participant REPO as RecipeRepository
    participant DB as MySQL (transaction)

    U->>FE: Remplit le formulaire<br/>(titre, ingrédients, étapes, etc.)
    FE->>API: POST /recipes + cookie rs_session
    API->>AUTH: vérifie session
    AUTH-->>API: req.auth = { userId, ... }
    API->>CTRL: createRecipe(req, res)
    CTRL->>DTO: parse(req.body)
    DTO-->>CTRL: input typé
    CTRL->>SVC: create(userId, input)
    SVC->>SLUG: createDraftSlug(userId)
    SLUG->>REPO: existsBySlug(candidat)
    REPO-->>SLUG: false
    SLUG-->>SVC: "draft-42-abc123"
    SVC->>REPO: create({ ...input, slug, status: draft })

    rect rgb(245, 245, 245)
        Note over REPO,DB: Transaction MySQL
        REPO->>DB: BEGIN
        REPO->>DB: INSERT INTO Recipes(...)
        REPO->>DB: INSERT INTO RecipeIngredients(...)
        REPO->>DB: INSERT INTO RecipeSteps(...)
        REPO->>DB: INSERT INTO RecipeTags(...)
        REPO->>DB: INSERT INTO RecipeEquipments(...)
        REPO->>DB: COMMIT
    end

    DB-->>REPO: OK
    REPO-->>SVC: Recipe (avec id)
    SVC-->>CTRL: Recipe
    CTRL-->>FE: 201 { recipe }
    FE-->>U: Redirection /me/recipes/list

    Note over U,FE: L'utilisateur édite encore<br/>puis décide de soumettre

    U->>FE: Clique "Soumettre à modération"
    FE->>API: POST /recipes/:id/submit
    API->>AUTH: req.auth
    API->>CTRL: submitRecipe(req, res)
    CTRL->>SVC: submit(recipeId, auth)
    SVC->>REPO: findById(recipeId)
    REPO-->>SVC: Recipe
    SVC->>SVC: requireEditableRecipe()<br/>(vérifie owner + statut draft)
    SVC->>SLUG: createPublicSlug(recipe.title)
    SLUG-->>SVC: "tarte-aux-pommes-de-mamie"
    SVC->>REPO: submit(recipeId, publicSlug)
    REPO->>DB: UPDATE Recipes SET<br/>Status='pending', SubmittedAt=NOW(),<br/>Slug=?
    DB-->>REPO: OK
    REPO-->>SVC: Recipe
    SVC-->>CTRL: Recipe en attente
    CTRL-->>FE: 200 { recipe }
```

---

## Modération par l'administrateur

```mermaid
sequenceDiagram
    autonumber
    actor A as Administrateur
    participant FE as Front Angular<br/>admin/review.ts
    participant API as POST /api/v1/admin/recipes/:id/approve
    participant AUTH as requireAuth + requireAdmin
    participant CTRL as AdminRecipesController
    participant SVC as AdminRecipeService
    participant REPO as AdminRecipeRepository
    participant DB as MySQL

    A->>FE: Ouvre le panneau de modération
    FE->>API: GET /admin/recipes/pending
    Note over API,DB: Récupère la liste<br/>des recettes en attente
    API-->>FE: [{ recipe1, recipe2, ... }]
    FE-->>A: Affiche la liste

    A->>FE: Clique sur une recette
    FE->>API: GET /admin/recipes/:id
    API-->>FE: Détail complet (ingrédients, étapes, auteur)
    FE-->>A: Affiche détail

    alt L'admin approuve
        A->>FE: Clique "Approuver"
        FE->>API: POST /admin/recipes/:id/approve
        API->>AUTH: vérifie session admin
        API->>CTRL: approveRecipe(req, res)
        CTRL->>SVC: approve(recipeId, adminUserId)
        SVC->>REPO: findByIdForAdmin(recipeId)
        REPO-->>SVC: RecipeAdmin
        SVC->>SVC: requireModeratableRecipe()<br/>(statut == pending)
        SVC->>REPO: publish(recipeId, adminUserId)
        REPO->>DB: UPDATE Recipes SET<br/>Status='published',<br/>PublishedAt=NOW(),<br/>ModeratedAt=NOW(),<br/>ModeratedByUserId=?
        DB-->>REPO: OK
        REPO-->>SVC: true
        SVC-->>CTRL: true
        CTRL-->>FE: 200 { success: true }
    else L'admin rejette
        A->>FE: Saisit motif + clique "Rejeter"
        FE->>API: POST /admin/recipes/:id/reject<br/>{ rejectionReason }
        API->>CTRL: rejectRecipe(req, res)
        CTRL->>SVC: reject(recipeId, adminUserId, reason)
        SVC->>REPO: findByIdForAdmin(recipeId)
        REPO-->>SVC: RecipeAdmin
        SVC->>SVC: requireModeratableRecipe()
        SVC->>REPO: reject(recipeId, adminUserId, reason)
        REPO->>DB: UPDATE Recipes SET<br/>Status='rejected',<br/>ModeratedAt=NOW(),<br/>ModeratedByUserId=?,<br/>RejectionReason=?
        DB-->>REPO: OK
        REPO-->>SVC: true
        SVC-->>CTRL: true
        CTRL-->>FE: 200 { success: true }
    end

    Note over A,FE: L'auteur de la recette<br/>verra le statut mis à jour<br/>dans /me/recipes/list
```

---

## Notes de défense soutenance

**Justifications de design**

- **Statut explicite plutôt qu'un booléen `published`** → permet la traçabilité fine (qui a modéré, quand, pourquoi rejeté). Coût : un type ENUM à maintenir. Bénéfice : audit trail complet et logique de transitions claire.
- **Slug en deux temps** (`draft-X-Y` pendant brouillon, slug définitif au moment de la publication) → évite de "réserver" un slug public pour une recette qui pourrait ne jamais être publiée. Et évite les conflits si plusieurs auteurs préparent des recettes avec le même titre.
- **Transaction MySQL** pour la création → si une INSERT dans `RecipeIngredients` échoue, on rollback tout. Garantit la cohérence : pas de recette orpheline sans ingrédients.
- **Séparation `RecipeService` (auteur) vs `AdminRecipeService` (modération)** → respecte le Single Responsibility Principle, et permet d'avoir des routes Express séparées avec des middlewares différents (`requireAuth` vs `requireAdmin`).
- **`requireEditableRecipe` côté service, pas seulement côté middleware** → l'autorisation métier (owner + statut draft) est plus fine que ce qu'un middleware peut faire. Garde la logique d'autorisation au plus près de la règle métier.

**Q/R probable du jury**

- *"Que se passe-t-il si l'admin clique deux fois sur Approuver ?"* → `requireModeratableRecipe()` vérifie `statut === pending`. Le 2e appel renverra une erreur métier. Pas besoin d'idempotence niveau infrastructure pour ce cas.
- *"Comment gérez-vous les conflits d'édition concurrents ?"* → Le projet n'implémente pas d'optimistic locking (pas de colonne `version`). Pour ce projet pédagogique c'est acceptable, mais c'est une évolution à mentionner si on attend de la concurrence forte.
- *"Pourquoi pas un système de notification email à l'auteur quand la recette est approuvée/rejetée ?"* → C'est une évolution naturelle. Le service `MailService` existe déjà (utilisé pour validation email + reset password) → il suffirait d'ajouter un appel après `publish`/`reject`.
