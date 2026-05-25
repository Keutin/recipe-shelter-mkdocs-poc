# Diagramme de classes — Modèle de domaine

> Représentation UML du modèle métier de Recipe Shelter, dérivée directement
> du schéma MySQL (`backend/database/migrations/1_create_schema.sql`).
>
> Le modèle est découpé en 3 vues thématiques pour rester lisible.
> Toutes les associations indiquent la multiplicité.

---

## 1. Identité et authentification

```mermaid
classDiagram
    direction LR

    class Role {
        +Long id
        +String name
    }

    class User {
        +Long id
        +String mail UK
        +String username UK
        +String password
        +UserStatus status
        +DateTime emailValidatedAt
        +DateTime bannedAt
        +String bannedReason
        +DateTime createdAt
        +DateTime updatedAt
    }

    class UserStatus {
        <<enumeration>>
        inactive
        active
        banned
    }

    class EmailValidation {
        +Long id
        +String tokenHash UK
        +DateTime expiresAt
        +DateTime usedAt
        +DateTime createdAt
    }

    class PasswordReset {
        +Long id
        +String tokenHash UK
        +DateTime expiresAt
        +DateTime usedAt
        +DateTime createdAt
    }

    class UserModerationLog {
        +Long id
        +ModerationAction action
        +String reason
        +DateTime createdAt
    }

    class ModerationAction {
        <<enumeration>>
        ban
        unban
    }

    Role "1" <-- "*" User : a pour rôle
    User "1" o-- "*" EmailValidation : possède
    User "1" o-- "*" PasswordReset : possède
    User "1" <-- "*" UserModerationLog : cible
    User "1" <-- "*" UserModerationLog : exécutée par (admin)
    User "0..1" <-- "*" User : banni par
```

**Notes de défense soutenance**
- Le hash du mot de passe est stocké en bcrypt (`BCRYPT_COST` configurable, défaut 12).
- Les tokens de validation/reset sont stockés **hachés** côté serveur, jamais en clair → si la BDD fuite, les tokens ne sont pas exploitables.
- Le statut `banned` est gardé en parallèle d'un log immuable (`UserModerationLogs`) pour conserver l'historique des sanctions.

---

## 2. Recettes et contenus

```mermaid
classDiagram
    direction LR

    class Recipe {
        +Long id
        +String title
        +String slug UK
        +String description
        +String coverImage
        +int prepTimeMinutes
        +int restTimeMinutes
        +int cookTimeMinutes
        +int servings
        +RecipeStatus status
        +DateTime createdAt
        +DateTime submittedAt
        +DateTime moderatedAt
        +DateTime publishedAt
        +DateTime archivedAt
        +String rejectionReason
    }

    class RecipeStatus {
        <<enumeration>>
        draft
        pending
        published
        rejected
        archived
    }

    class RecipeCategory {
        +Long id
        +String name UK
        +String slug UK
        +String iconName
    }

    class RecipeStep {
        +int stepNumber PK
        +String description
    }

    class RecipeIngredient {
        +Long id
        +Decimal quantity
        +String unit
        +String note
        +int sortOrder
    }

    class Ingredient {
        +Long id
        +String name UK
        +String slug UK
    }

    class Tag {
        +Long id
        +String name UK
        +String slug UK
    }

    class Equipment {
        +Long id
        +String name UK
        +String slug UK
    }

    class User {
        +Long id
        +String username
    }

    User "1" <-- "*" Recipe : auteur
    User "0..1" <-- "*" Recipe : modéré par
    RecipeCategory "0..1" <-- "*" Recipe : appartient à
    Recipe "1" *-- "*" RecipeStep : contient
    Recipe "1" *-- "*" RecipeIngredient : utilise
    Ingredient "1" <-- "*" RecipeIngredient
    Recipe "*" -- "*" Tag : étiquetée
    Recipe "*" -- "*" Equipment : nécessite
```

**Notes de défense soutenance**
- Le cycle de vie d'une recette suit un **état explicite** (`draft → pending → published | rejected → archived`) → traçabilité claire pour la modération.
- `RecipeStep` est une **composition** (clé primaire composite `RecipeId + StepNumber`) : pas d'existence hors de sa recette.
- `RecipeIngredient` est une **classe d'association** : elle porte des attributs propres (`quantity`, `unit`, `note`, `sortOrder`) — c'est plus qu'une simple M2M.
- Le `slug` unique sur `Recipes` permet des URLs lisibles et SEO-friendly côté front (`/recipes/:slug`).

---

## 3. Interactions sociales

```mermaid
classDiagram
    direction LR

    class User {
        +Long id
        +String username
    }

    class Recipe {
        +Long id
        +String title
    }

    class Favorite {
        +DateTime createdAt
    }

    class Comment {
        +Long id
        +String comment
        +int rating
        +DateTime moderatedAt
        +DateTime deletedAt
        +DateTime createdAt
        +DateTime updatedAt
    }

    User "*" -- "*" Recipe : favoris (Favorite)
    User "1" <-- "*" Comment : auteur
    Recipe "1" <-- "*" Comment : commente
    Comment "0..1" <-- "*" Comment : réponse à (threading)
    User "0..1" <-- "*" Comment : modéré par
    User "0..1" <-- "*" Comment : supprimé par
```

**Notes de défense soutenance**
- `Favorite` est une **table d'association pure** (PK = `UserId + RecipeId`) → un utilisateur ne peut pas favoriser deux fois la même recette.
- `Comment` supporte le **threading** via `parentCommentId` (auto-référence), avec validation côté service que le parent est lui-même un commentaire racine (un seul niveau de réponse).
- La **suppression douce** (`deletedAt`/`deletedByUserId`) et la **modération** (`moderatedAt`/`moderatedByUserId`) sont séparées → on garde trace même après action modérateur.
- Note `rating` est `[1..5]` via contrainte SQL `CHECK`.
