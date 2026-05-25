# Data dictionary du backend Recipe Shelter

> Document de référence pour la défense de soutenance (RNCP). Décrit le schéma
> physique de la base MySQL, table par table, colonne par colonne, avec les
> contraintes, les index et les choix de modélisation.

## 1. Pourquoi un data dictionary

Le data dictionary est la **source de vérité documentaire** du modèle de
données. Il sert trois objectifs précis dans le cadre de la défense.

Premièrement, il fournit au jury un point d'entrée unique pour évaluer le
**bloc 3 (conception et développement)** : plutôt que de demander à l'examinateur
de lire un script SQL de 306 lignes, on lui présente un document structuré qui
explique chaque table dans son contexte métier, avec les contraintes et les
choix de modélisation associés.

Deuxièmement, il garantit la **cohérence schéma ↔ code applicatif**. Le backend
ne s'appuie sur aucun ORM : les requêtes SQL sont écrites à la main dans la
couche `repositories/`, et les types TypeScript des entités du domaine sont
rédigés à la main également. Le risque d'une dérive silencieuse entre la base
et le code est réel ; ce document est l'outil qui permet de la détecter en
révision.

Troisièmement, il sert de **support à l'audit RGPD**. Toutes les données
personnelles (`Mail`, `Username`, `Password`, journaux de modération) sont
visibles dans un seul document, ce qui facilite l'inventaire des traitements,
la justification des durées de conservation et l'identification des trous
résiduels (le plus important étant l'absence d'un statut `deleted` pour les
utilisateurs — voir §3.2).

La **source unique de vérité** est le script de migration
``backend/database/migrations/1_create_schema.sql``.
Toute divergence entre ce document et ce fichier doit être traitée comme un
bug du document, jamais du SQL.

## 2. Vue d'ensemble

Le schéma compte **16 tables** organisées en cinq domaines fonctionnels.

| Domaine                | Tables                                                                                | Rôle                                                              |
| ---------------------- | ------------------------------------------------------------------------------------- | ----------------------------------------------------------------- |
| Auth / Users           | `Roles`, `Users`, `EmailValidations`, `PasswordResets`                                | Comptes, sessions, validation email, réinitialisation mot de passe. |
| Modération utilisateur | `UserModerationLogs`                                                                  | Audit trail des bans et débans administrateurs.                   |
| Catalogues             | `RecipeCategories`, `Ingredients`, `Equipments`, `Tags`                               | Référentiels normalisés réutilisés par les recettes.              |
| Recettes               | `Recipes`, `RecipeSteps`, `RecipeIngredients`, `RecipeEquipments`, `RecipeTags`       | Recette principale et ses contenus liés.                          |
| Social                 | `Favorites`, `Comments`                                                               | Interactions communautaires sur les recettes publiées.            |

Toutes les tables utilisent le moteur **InnoDB**, le charset **utf8mb4** et la
collation **utf8mb4\_unicode\_ci**. Les clés étrangères sont systématiquement
déclarées, avec des règles `ON UPDATE CASCADE` et des règles `ON DELETE` qui
varient selon la sémantique métier (voir §11).

```
        Roles ◄────── Users ◄─────── UserModerationLogs
                       ▲   │
                       │   ├── EmailValidations
                       │   ├── PasswordResets
                       │   ├── Recipes ────► RecipeCategories
                       │   │     │
                       │   │     ├── RecipeSteps
                       │   │     ├── RecipeIngredients ──► Ingredients
                       │   │     ├── RecipeEquipments ──► Equipments
                       │   │     └── RecipeTags ────────► Tags
                       │   │
                       │   ├── Favorites ◄── Recipes
                       │   └── Comments  ◄── Recipes
                       │
                       └── self-FK : Users.BannedByUserId
```

Diagramme ERD complet attendu dans
[`documentation/soutenance/backend/diagrams/g5-erd.md`](documentation/soutenance/backend/diagrams/g5-erd.md).

## 3. Auth et Users

### 3.1. `Roles`

Référentiel des rôles applicatifs. Trois rôles sont attendus en seed :
`user`, `admin`, `superadmin`. La table est intentionnellement triviale : un
identifiant et un nom unique, sans hiérarchie matérialisée. La logique
d'autorisation (qui peut modérer, qui peut bannir) est tranchée côté
services à partir du nom du rôle, pas par une table de permissions.

| Colonne | Type              | Nullable | Default | Contraintes               |
| ------- | ----------------- | -------- | ------- | ------------------------- |
| `Id`    | BIGINT UNSIGNED   | NOT NULL | AUTO_INCREMENT | PK                  |
| `Name`  | VARCHAR(64)       | NOT NULL | —       | UNIQUE (`roles_name_UK`)  |

### 3.2. `Users`

Table centrale du domaine. Stocke les comptes utilisateurs et leur état de
modération. Le champ `Password` contient un hash bcrypt (jamais le mot de
passe en clair), conformément au coût configuré par la variable
d'environnement `BCRYPT_COST` (défaut 12).

| Colonne             | Type              | Nullable | Default            | Contraintes                                                |
| ------------------- | ----------------- | -------- | ------------------ | ---------------------------------------------------------- |
| `Id`                | BIGINT UNSIGNED   | NOT NULL | AUTO_INCREMENT     | PK                                                         |
| `Mail`              | VARCHAR(255)      | NOT NULL | —                  | UNIQUE (`users_mail_UK`)                                   |
| `Username`          | VARCHAR(64)       | NOT NULL | —                  | UNIQUE (`users_username_UK`)                               |
| `Password`          | VARCHAR(255)      | NOT NULL | —                  | Hash bcrypt (jamais en clair)                              |
| `RoleId`            | BIGINT UNSIGNED   | NOT NULL | —                  | FK → `Roles.Id`                                            |
| `Status`            | ENUM(...)         | NOT NULL | `'inactive'`       | Voir ENUM ci-dessous                                       |
| `EmailValidatedAt`  | DATETIME          | NULL     | —                  | Renseigné après usage d'un `EmailValidations.TokenHash`    |
| `BannedByUserId`    | BIGINT UNSIGNED   | NULL     | —                  | Self-FK → `Users.Id`                                       |
| `BannedReason`      | TEXT              | NULL     | —                  | Motif lisible saisi par l'admin                            |
| `BannedAt`          | DATETIME          | NULL     | —                  | Date du dernier ban appliqué                               |
| `CreatedAt`         | DATETIME          | NOT NULL | CURRENT_TIMESTAMP  | —                                                          |
| `UpdatedAt`         | DATETIME          | NOT NULL | CURRENT_TIMESTAMP  | `ON UPDATE CURRENT_TIMESTAMP`                              |

**ENUM `Users.Status`** : `'inactive' | 'active' | 'banned'`.
- `inactive` : compte créé, email non encore validé. Valeur par défaut.
- `active` : email validé, le compte peut s'authentifier et publier.
- `banned` : compte verrouillé par un administrateur. La session est rejetée
  à l'entrée des middlewares d'authentification.

**Point RGPD signalé en Vague 1 `securite.md`** : il n'existe **pas** de
valeur `'deleted'` dans cet ENUM, et aucune colonne `DeletedAt` n'existe sur
`Users`. Le soft-delete utilisateur n'est donc pas implémenté à ce jour, ce
qui constitue un trou identifié dans l'inventaire des traitements (droit à
l'effacement, art. 17 RGPD). Mention attendue à la soutenance, avec piste
de remédiation : ajout d'une valeur `deleted` à l'ENUM et anonymisation des
champs `Mail` / `Username` en place.

**Foreign keys**

| Contrainte                  | Cible                | ON UPDATE | ON DELETE  | Justification                                                  |
| --------------------------- | -------------------- | --------- | ---------- | -------------------------------------------------------------- |
| `users_role_FK`             | `Roles.Id`           | CASCADE   | RESTRICT   | On refuse de supprimer un rôle encore utilisé.                 |
| `users_banned_by_user_FK`   | `Users.Id` (self-FK) | CASCADE   | SET NULL   | Si l'admin qui a banni est supprimé, on conserve la trace du ban mais on oublie qui l'a posé. |

**Index secondaires** : `idx_users_role_id`, `idx_users_status`,
`idx_users_banned_by_user_id`. Les deux premiers servent les requêtes
d'administration (listing par rôle, par état) ; le troisième sert
l'intégrité de la self-FK.

### 3.3. `UserModerationLogs`

Journal d'audit des actions de modération sur les comptes. Une ligne est
insérée à chaque ban et à chaque déban, avec l'identifiant de l'admin qui a
agi, le motif et la date. La table est en **append-only** côté code : aucun
chemin applicatif ne supprime ni ne modifie une ligne existante.

| Colonne     | Type              | Nullable | Default            | Contraintes                                  |
| ----------- | ----------------- | -------- | ------------------ | -------------------------------------------- |
| `Id`        | BIGINT UNSIGNED   | NOT NULL | AUTO_INCREMENT     | PK                                           |
| `UserId`    | BIGINT UNSIGNED   | NOT NULL | —                  | FK → `Users.Id` (utilisateur ciblé)          |
| `AdminId`   | BIGINT UNSIGNED   | NOT NULL | —                  | FK → `Users.Id` (admin qui a agi)            |
| `Action`    | ENUM(...)         | NOT NULL | —                  | Voir ENUM ci-dessous                         |
| `Reason`    | TEXT              | NOT NULL | —                  | Motif obligatoire                            |
| `CreatedAt` | DATETIME          | NOT NULL | CURRENT_TIMESTAMP  | —                                            |

**ENUM `UserModerationLogs.Action`** : `'ban' | 'unban'`.
- `ban` : l'admin a posé un ban (`Users.Status` passe à `'banned'`).
- `unban` : l'admin a levé un ban (`Users.Status` repasse à `'active'`).

**Foreign keys** : les deux FK pointent vers `Users.Id` avec `ON DELETE
RESTRICT`. C'est délibéré : un journal d'audit perdrait sa valeur probante
si un utilisateur ciblé ou un admin auteur pouvait être effacé en cascade.

**Index secondaires** : `idx_user_moderation_logs_user_id`,
`idx_user_moderation_logs_admin_id`, `idx_user_moderation_logs_created_at`.
Le tri chronologique est explicitement indexé pour la consultation
"historique des actions d'un admin sur les 30 derniers jours".

### 3.4. `EmailValidations`

Stocke les jetons à usage unique générés à l'inscription pour valider
l'adresse mail du compte. Le jeton n'est jamais stocké en clair : la
colonne `TokenHash` contient son empreinte **SHA-256 en hexadécimal**, soit
exactement 64 caractères, d'où le type `CHAR(64)`.

| Colonne     | Type            | Nullable | Default            | Contraintes                                            |
| ----------- | --------------- | -------- | ------------------ | ------------------------------------------------------ |
| `Id`        | BIGINT UNSIGNED | NOT NULL | AUTO_INCREMENT     | PK                                                     |
| `UserId`    | BIGINT UNSIGNED | NOT NULL | —                  | FK → `Users.Id`                                        |
| `TokenHash` | CHAR(64)        | NOT NULL | —                  | UNIQUE (`email_validations_tokenhash_UK`), SHA-256 hex |
| `ExpiresAt` | DATETIME        | NOT NULL | —                  | Vérifié à la consommation                              |
| `UsedAt`    | DATETIME        | NULL     | —                  | NULL = jeton encore utilisable                         |
| `CreatedAt` | DATETIME        | NOT NULL | CURRENT_TIMESTAMP  | —                                                      |

**Foreign key** : `email_validations_user_FK` → `Users.Id` avec `ON DELETE
CASCADE`. Cohérent avec la sémantique : si le compte disparaît, ses jetons
résiduels n'ont plus de sens.

### 3.5. `PasswordResets`

Structure jumelle de `EmailValidations`, mais pour le workflow de
réinitialisation de mot de passe. Mêmes choix techniques : hash SHA-256
hex en `CHAR(64)`, unicité du hash, expiration explicite, marqueur de
consommation `UsedAt`. La séparation en deux tables (plutôt qu'une table
générique `Tokens` avec un discriminant) est volontaire : les deux
workflows ont des durées d'expiration différentes et la séparation simplifie
les requêtes de purge.

| Colonne     | Type            | Nullable | Default            | Contraintes                                          |
| ----------- | --------------- | -------- | ------------------ | ---------------------------------------------------- |
| `Id`        | BIGINT UNSIGNED | NOT NULL | AUTO_INCREMENT     | PK                                                   |
| `UserId`    | BIGINT UNSIGNED | NOT NULL | —                  | FK → `Users.Id`                                      |
| `TokenHash` | CHAR(64)        | NOT NULL | —                  | UNIQUE (`password_resets_tokenhash_UK`), SHA-256 hex |
| `ExpiresAt` | DATETIME        | NOT NULL | —                  | —                                                    |
| `UsedAt`    | DATETIME        | NULL     | —                  | —                                                    |
| `CreatedAt` | DATETIME        | NOT NULL | CURRENT_TIMESTAMP  | —                                                    |

**Foreign key** : `password_resets_user_FK` → `Users.Id` avec `ON DELETE
CASCADE` (même raison qu'en §3.4).

## 4. Catalogues

Les quatre tables suivantes sont des **référentiels** : peu d'écritures
(seeds, ajouts ponctuels par les admins), beaucoup de lectures. Elles sont
toutes structurées sur le couple `(Name, Slug)` avec deux contraintes
d'unicité, le slug servant d'identifiant stable dans les URL frontend.

### 4.1. `RecipeCategories`

Catégories métier d'une recette (entrées, plats, desserts, etc.). Contient
en plus un `IconName` côté frontend pour afficher une icône cohérente.

| Colonne      | Type              | Nullable | Default            | Contraintes                                         |
| ------------ | ----------------- | -------- | ------------------ | --------------------------------------------------- |
| `Id`         | BIGINT UNSIGNED   | NOT NULL | AUTO_INCREMENT     | PK                                                  |
| `Name`       | VARCHAR(100)      | NOT NULL | —                  | UNIQUE (`recipe_categories_name_UK`)                |
| `Slug`       | VARCHAR(100)      | NOT NULL | —                  | UNIQUE (`recipe_categories_slug_UK`)                |
| `IconName`   | VARCHAR(64)       | NOT NULL | —                  | Nom d'icône côté frontend (ex. `'utensils'`)        |
| `CreatedAt`  | DATETIME          | NOT NULL | CURRENT_TIMESTAMP  | —                                                   |
| `UpdatedAt`  | DATETIME          | NOT NULL | CURRENT_TIMESTAMP  | `ON UPDATE CURRENT_TIMESTAMP`                       |

### 4.2. `Ingredients`

Catalogue des ingrédients réutilisables (farine, sucre, oeuf, etc.). Pas
de timestamps : le catalogue est considéré comme stable, les mutations
restent rares.

| Colonne | Type              | Nullable | Default        | Contraintes                              |
| ------- | ----------------- | -------- | -------------- | ---------------------------------------- |
| `Id`    | BIGINT UNSIGNED   | NOT NULL | AUTO_INCREMENT | PK                                       |
| `Name`  | VARCHAR(255)      | NOT NULL | —              | UNIQUE (`ingredients_name_UK`)           |
| `Slug`  | VARCHAR(255)      | NOT NULL | —              | UNIQUE (`ingredients_slug_UK`)           |

### 4.3. `Equipments`

Catalogue des équipements de cuisine (four, robot, fouet, etc.). Même
structure minimale que `Ingredients`.

| Colonne | Type              | Nullable | Default        | Contraintes                            |
| ------- | ----------------- | -------- | -------------- | -------------------------------------- |
| `Id`    | BIGINT UNSIGNED   | NOT NULL | AUTO_INCREMENT | PK                                     |
| `Name`  | VARCHAR(255)      | NOT NULL | —              | UNIQUE (`equipments_name_UK`)          |
| `Slug`  | VARCHAR(255)      | NOT NULL | —              | UNIQUE (`equipments_slug_UK`)          |

### 4.4. `Tags`

Catalogue des tags libres associables à une recette (végétarien, sans
gluten, rapide, etc.).

| Colonne | Type              | Nullable | Default        | Contraintes                       |
| ------- | ----------------- | -------- | -------------- | --------------------------------- |
| `Id`    | BIGINT UNSIGNED   | NOT NULL | AUTO_INCREMENT | PK                                |
| `Name`  | VARCHAR(255)      | NOT NULL | —              | UNIQUE (`tags_name_UK`)           |
| `Slug`  | VARCHAR(255)      | NOT NULL | —              | UNIQUE (`tags_slug_UK`)           |

## 5. Recettes

### 5.1. `Recipes`

Entité métier principale du domaine. Une recette appartient à un
utilisateur (`UserId`), peut être rattachée à une catégorie (`CategoryId`,
nullable), et traverse un cycle de vie matérialisé par la colonne `Status`.

| Colonne              | Type              | Nullable | Default            | Contraintes                                                       |
| -------------------- | ----------------- | -------- | ------------------ | ----------------------------------------------------------------- |
| `Id`                 | BIGINT UNSIGNED   | NOT NULL | AUTO_INCREMENT     | PK                                                                |
| `UserId`             | BIGINT UNSIGNED   | NOT NULL | —                  | FK → `Users.Id` (auteur)                                          |
| `CategoryId`         | BIGINT UNSIGNED   | NULL     | —                  | FK → `RecipeCategories.Id`                                        |
| `Title`              | VARCHAR(255)      | NOT NULL | —                  | Indexé en FULLTEXT (`ft_recipes_title`)                           |
| `Slug`               | VARCHAR(255)      | NOT NULL | —                  | UNIQUE (`recipes_slug_UK`)                                        |
| `Description`        | TEXT              | NOT NULL | —                  | —                                                                 |
| `RecipeCoverImage`   | VARCHAR(2048)     | NULL     | —                  | URL ou chemin de l'image de couverture                            |
| `PrepTimeMinutes`    | INT               | NOT NULL | —                  | Temps de préparation                                              |
| `RestTimeMinutes`    | INT               | NULL     | —                  | Temps de repos                                                    |
| `CookTimeMinutes`    | INT               | NULL     | —                  | Temps de cuisson                                                  |
| `Servings`           | INT               | NOT NULL | —                  | Nombre de portions                                                |
| `Status`             | ENUM(...)         | NOT NULL | `'draft'`          | Voir ENUM ci-dessous                                              |
| `CreatedAt`          | DATETIME          | NOT NULL | CURRENT_TIMESTAMP  | —                                                                 |
| `SubmittedAt`        | DATETIME          | NULL     | —                  | Date de passage `draft → pending`                                 |
| `ModeratedAt`        | DATETIME          | NULL     | —                  | Date de la décision de modération                                 |
| `ModeratedByUserId`  | BIGINT UNSIGNED   | NULL     | —                  | FK → `Users.Id` (modérateur)                                      |
| `PublishedAt`        | DATETIME          | NULL     | —                  | Date de passage à `published`                                     |
| `ArchivedAt`         | DATETIME          | NULL     | —                  | Date de passage à `archived`                                      |
| `RejectionReason`    | TEXT              | NULL     | —                  | Motif si `Status = 'rejected'`                                    |
| `UpdatedAt`          | DATETIME          | NOT NULL | CURRENT_TIMESTAMP  | `ON UPDATE CURRENT_TIMESTAMP`                                     |

**ENUM `Recipes.Status`** :
`'draft' | 'pending' | 'published' | 'rejected' | 'archived'`.

- `draft` : brouillon créé par l'auteur, modifiable, invisible au public.
- `pending` : soumis à modération. **Attention : la valeur est `pending`,
  pas `pending_review`.** Plusieurs documents internes ont employé l'un
  pour l'autre ; la valeur canonique est celle déclarée ici.
- `published` : validé par un modérateur, visible publiquement.
- `rejected` : refusé par un modérateur, motif dans `RejectionReason`.
- `archived` : retiré de la diffusion publique sans suppression physique.

**Foreign keys**

| Contrainte                       | Cible                  | ON UPDATE | ON DELETE  | Justification                                                |
| -------------------------------- | ---------------------- | --------- | ---------- | ------------------------------------------------------------ |
| `recipes_user_FK`                | `Users.Id`             | CASCADE   | RESTRICT   | On refuse de supprimer un auteur tant qu'il a des recettes.  |
| `recipes_category_FK`            | `RecipeCategories.Id`  | CASCADE   | RESTRICT   | On refuse de supprimer une catégorie utilisée.               |
| `recipes_moderated_by_user_FK`   | `Users.Id`             | CASCADE   | SET NULL   | Si le modérateur disparaît, on conserve la décision mais on oublie qui l'a prise. |

**Index secondaires** : `idx_recipes_user_id` (listing par auteur),
`idx_recipes_category_id` (filtre par catégorie),
`idx_recipes_status` (file de modération, listing public),
`idx_recipes_moderated_by_user_id` (audit par modérateur),
`ft_recipes_title` (**FULLTEXT** sur `Title`).

**Note sur le FULLTEXT** : la recherche textuelle de recettes par titre
s'appuie sur cet index FULLTEXT MySQL natif (`MATCH(Title) AGAINST(?)`).
Le choix d'un index FULLTEXT plutôt qu'un `LIKE '%mot%'` est défendable :
performance acceptable au-delà de quelques centaines de recettes, et
ranking pertinent géré par MySQL. La limite (insensibilité aux fautes
d'orthographe, dépendance à la longueur minimale de mot configurée dans
MySQL) est assumée pour le périmètre actuel.

### 5.2. `RecipeSteps`

Étapes d'une recette, ordonnées par `StepNumber`. La clé primaire
composite `(RecipeId, StepNumber)` garantit qu'une étape donnée d'une
recette est unique sans nécessiter de contrainte d'unicité supplémentaire.

| Colonne       | Type              | Nullable | Default | Contraintes                          |
| ------------- | ----------------- | -------- | ------- | ------------------------------------ |
| `RecipeId`    | BIGINT UNSIGNED   | NOT NULL | —       | PK composite, FK → `Recipes.Id`      |
| `StepNumber`  | INT               | NOT NULL | —       | PK composite                         |
| `Description` | TEXT              | NOT NULL | —       | Texte libre de l'étape               |

**Foreign key** : `recipe_steps_recipe_FK` → `Recipes.Id` avec `ON DELETE
CASCADE`. La suppression d'une recette efface ses étapes.

### 5.3. `RecipeIngredients`

Association `Recipes` ↔ `Ingredients` enrichie d'une quantité, d'une
unité, d'une note libre et d'un ordre d'affichage. Contrairement à
`RecipeSteps`, on utilise un `Id` auto-incrémenté plutôt qu'une PK
composite, car le même ingrédient peut apparaître plusieurs fois dans
une recette avec des notes différentes (ex. "100 g pour la pâte" et
"50 g pour le dressage").

| Colonne        | Type              | Nullable | Default        | Contraintes                                |
| -------------- | ----------------- | -------- | -------------- | ------------------------------------------ |
| `Id`           | BIGINT UNSIGNED   | NOT NULL | AUTO_INCREMENT | PK                                         |
| `RecipeId`     | BIGINT UNSIGNED   | NOT NULL | —              | FK → `Recipes.Id`                          |
| `IngredientId` | BIGINT UNSIGNED   | NOT NULL | —              | FK → `Ingredients.Id`                      |
| `Quantity`     | DECIMAL(10,3)     | NOT NULL | —              | Précision 3 décimales (ex. `0.250`)        |
| `Unit`         | VARCHAR(64)       | NULL     | —              | Unité libre (`g`, `ml`, `cuillère`, etc.)  |
| `Note`         | VARCHAR(255)      | NULL     | —              | Précision libre par ingrédient             |
| `SortOrder`    | INT               | NOT NULL | `1`            | Ordre d'affichage dans la recette          |

**Foreign keys**

| Contrainte                          | Cible             | ON UPDATE | ON DELETE | Justification                                       |
| ----------------------------------- | ----------------- | --------- | --------- | --------------------------------------------------- |
| `recipe_ingredients_recipe_FK`      | `Recipes.Id`      | CASCADE   | CASCADE   | Supprimer une recette efface ses ingrédients.       |
| `recipe_ingredients_ingredient_FK`  | `Ingredients.Id`  | CASCADE   | RESTRICT  | On refuse de supprimer un ingrédient référencé.     |

**Index secondaires** : `idx_recipe_ingredients_recipe_sort (RecipeId,
SortOrder)` pour la liste ordonnée des ingrédients d'une recette,
`idx_recipe_ingredients_ingredient_id` pour le sens inverse (recettes
contenant un ingrédient donné).

### 5.4. `RecipeEquipments`

Association `Recipes` ↔ `Equipments`, sans donnée supplémentaire. PK
composite `(RecipeId, EquipmentId)` : une recette ne référence qu'une
seule fois un équipement.

| Colonne       | Type              | Nullable | Default | Contraintes                          |
| ------------- | ----------------- | -------- | ------- | ------------------------------------ |
| `RecipeId`    | BIGINT UNSIGNED   | NOT NULL | —       | PK composite, FK → `Recipes.Id`      |
| `EquipmentId` | BIGINT UNSIGNED   | NOT NULL | —       | PK composite, FK → `Equipments.Id`   |

**Foreign keys**

| Contrainte                            | Cible           | ON UPDATE | ON DELETE | Justification                                  |
| ------------------------------------- | --------------- | --------- | --------- | ---------------------------------------------- |
| `recipe_equipment_recipe_FK`          | `Recipes.Id`    | CASCADE   | CASCADE   | Supprimer une recette efface ses associations. |
| `recipe_equipements_equipment_FK`     | `Equipments.Id` | CASCADE   | RESTRICT  | On refuse de supprimer un équipement utilisé.  |

**Index secondaire** : `idx_recipe_equipments_equipment_id` pour le sens
inverse.

### 5.5. `RecipeTags`

Association `Recipes` ↔ `Tags`. Même structure et mêmes choix que
`RecipeEquipments`.

| Colonne    | Type              | Nullable | Default | Contraintes                       |
| ---------- | ----------------- | -------- | ------- | --------------------------------- |
| `RecipeId` | BIGINT UNSIGNED   | NOT NULL | —       | PK composite, FK → `Recipes.Id`   |
| `TagId`    | BIGINT UNSIGNED   | NOT NULL | —       | PK composite, FK → `Tags.Id`      |

**Foreign keys**

| Contrainte                | Cible          | ON UPDATE | ON DELETE | Justification                              |
| ------------------------- | -------------- | --------- | --------- | ------------------------------------------ |
| `recipe_tags_recipe_FK`   | `Recipes.Id`   | CASCADE   | CASCADE   | Supprimer une recette efface ses tags.     |
| `recipe_tags_tag_FK`      | `Tags.Id`      | CASCADE   | RESTRICT  | On refuse de supprimer un tag référencé.   |

**Index secondaire** : `idx_recipe_tags_tag_id`.

## 6. Social

### 6.1. `Favorites`

Mise en favori d'une recette par un utilisateur. PK composite
`(UserId, RecipeId)` : un utilisateur ne met une recette en favori
qu'une fois. Pas de soft-delete : retirer un favori, c'est supprimer la
ligne. La sémantique métier ne nécessite pas de conserver l'historique.

| Colonne     | Type              | Nullable | Default            | Contraintes                       |
| ----------- | ----------------- | -------- | ------------------ | --------------------------------- |
| `UserId`    | BIGINT UNSIGNED   | NOT NULL | —                  | PK composite, FK → `Users.Id`     |
| `RecipeId`  | BIGINT UNSIGNED   | NOT NULL | —                  | PK composite, FK → `Recipes.Id`   |
| `CreatedAt` | DATETIME          | NOT NULL | CURRENT_TIMESTAMP  | —                                 |

**Foreign keys** : `favorites_user_FK` et `favorites_recipe_FK`, toutes
deux en `ON DELETE CASCADE`. Conséquence : supprimer un utilisateur ou
une recette efface les favoris associés sans alerte. C'est cohérent avec
la nature transitoire de l'information.

**Index secondaire** : `idx_favorites_recipe_id` pour les agrégats
"nombre de favoris d'une recette".

### 6.2. `Comments`

Commentaires et notes des utilisateurs sur les recettes publiées.
Structure la plus dense du schéma : elle porte en même temps la
hiérarchie de discussion (thread), la modération, le soft-delete et la
notation.

| Colonne             | Type                | Nullable | Default            | Contraintes                                |
| ------------------- | ------------------- | -------- | ------------------ | ------------------------------------------ |
| `Id`                | BIGINT UNSIGNED     | NOT NULL | AUTO_INCREMENT     | PK                                         |
| `RecipeId`          | BIGINT UNSIGNED     | NOT NULL | —                  | FK → `Recipes.Id`                          |
| `UserId`            | BIGINT UNSIGNED     | NOT NULL | —                  | FK → `Users.Id` (auteur du commentaire)    |
| `ParentCommentId`   | BIGINT UNSIGNED     | NULL     | —                  | Self-FK → `Comments.Id` (réponse à...)     |
| `ModeratedAt`       | DATETIME            | NULL     | —                  | Date de la décision de modération          |
| `ModeratedByUserId` | BIGINT UNSIGNED     | NULL     | —                  | FK → `Users.Id` (modérateur)               |
| `DeletedAt`         | DATETIME            | NULL     | —                  | Soft-delete : NULL = visible               |
| `DeletedByUserId`   | BIGINT UNSIGNED     | NULL     | —                  | FK → `Users.Id` (qui a supprimé)           |
| `Rating`            | TINYINT UNSIGNED    | NULL     | —                  | CHECK : NULL ou compris entre 1 et 5       |
| `Comment`           | TEXT                | NOT NULL | —                  | Texte du commentaire                       |
| `CreatedAt`         | DATETIME            | NOT NULL | CURRENT_TIMESTAMP  | —                                          |
| `UpdatedAt`         | DATETIME            | NOT NULL | CURRENT_TIMESTAMP  | `ON UPDATE CURRENT_TIMESTAMP`              |

**CHECK constraint `comments_rating_chk`** :

```sql
CHECK (Rating IS NULL OR (Rating BETWEEN 1 AND 5))
```

Garantit au niveau base que la note, lorsqu'elle est présente, est un
entier de 1 à 5. La validation applicative côté DTO double cette
garantie, mais la contrainte SQL est la défense en profondeur.

**Soft-delete via `DeletedAt`** : cohérent avec l'ADR-004 sur la
modération des contenus. Une suppression de commentaire (par l'auteur
ou par un modérateur) renseigne `DeletedAt` et `DeletedByUserId` sans
effacer la ligne, ce qui permet à la fois de conserver la trace pour
audit et de maintenir la structure des threads de réponse (un
commentaire supprimé reste un noeud parent valide pour ses réponses).

**Foreign keys**

| Contrainte                  | Cible                  | ON UPDATE | ON DELETE  | Justification                                            |
| --------------------------- | ---------------------- | --------- | ---------- | -------------------------------------------------------- |
| `comments_recipe_FK`        | `Recipes.Id`           | CASCADE   | CASCADE    | Supprimer la recette efface ses commentaires.            |
| `comments_user_FK`          | `Users.Id`             | CASCADE   | CASCADE    | Supprimer l'utilisateur efface ses commentaires.         |
| `comments_parent_FK`        | `Comments.Id` (self)   | CASCADE   | SET NULL   | Si le parent disparaît, la réponse devient orpheline plutôt que d'être effacée. |
| `comments_moderated_by_FK`  | `Users.Id`             | CASCADE   | SET NULL   | On conserve la décision sans le décideur.                |
| `comments_deleted_by_FK`    | `Users.Id`             | CASCADE   | SET NULL   | Idem.                                                    |

**Note sur l'asymétrie avec `Users`** : `comments_user_FK` est en
`CASCADE` alors que `recipes_user_FK` est en `RESTRICT`. La raison est
pédagogique et métier : on accepte qu'une suppression utilisateur
emporte ses commentaires (volume élevé, faible valeur unitaire), mais
pas ses recettes (valeur de contenu trop élevée, modération en amont).
À nouveau, le soft-delete utilisateur (§3.2) rendrait toute cette
discussion sans objet : c'est l'angle d'attaque prioritaire pour la
remédiation RGPD.

**Index secondaires** : `idx_comments_recipe_id` (commentaires d'une
recette), `idx_comments_user_id` (commentaires d'un utilisateur),
`idx_comments_parent_comment_id` (réponses à un commentaire),
`idx_comments_deleted_at` (filtrer les supprimés), `idx_comments_moderated_at`
(file de modération).

## 7. Conventions de modélisation

Les choix suivants sont appliqués de manière homogène à l'ensemble du
schéma. Les exceptions sont explicitement documentées dans les sections
par table ci-dessus.

| Convention                    | Règle                                                                                                       |
| ----------------------------- | ----------------------------------------------------------------------------------------------------------- |
| Clé primaire                  | `Id BIGINT UNSIGNED AUTO_INCREMENT` pour les entités. PK composite pour les pures tables d'association.     |
| Naming                        | Tables en **PascalCase pluriel** (`Users`, `Recipes`, `RecipeIngredients`). Colonnes en **PascalCase**.     |
| Timestamps                    | `CreatedAt DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP` et `UpdatedAt ... ON UPDATE CURRENT_TIMESTAMP` quand l'entité est mutable. |
| Soft-delete                   | Colonne `DeletedAt DATETIME NULL` quand le besoin métier l'exige (`Comments` uniquement à ce jour).        |
| Slugs                         | `Slug VARCHAR(255) NOT NULL UNIQUE` pour toute entité référencée dans une URL frontend.                    |
| Identifiants externes hashés  | Jetons stockés en `CHAR(64)` (SHA-256 hex), jamais en clair.                                               |
| Charset                       | `utf8mb4` / `utf8mb4_unicode_ci` sur toutes les tables (support emoji et caractères étendus).             |
| Moteur                        | `InnoDB` exclusivement (transactions, contraintes référentielles).                                          |
| ENUMs                         | Préférés aux tables de référence pour les états bornés et stables (`Users.Status`, `Recipes.Status`, `UserModerationLogs.Action`). |

## 8. Cohérence avec le code applicatif

Le backend ne s'appuie sur aucun ORM. Les types TypeScript du domaine
sont définis à la main dans `backend/src/repositories/**/*.repository.mysql.ts`,
et chaque colonne du schéma a un pendant dans l'interface du repository
correspondant. Ce document est l'outil de référence pour vérifier que
les deux représentations restent synchronisées en revue de code.

Trois exemples concrets :

- `backend/src/repositories/users/user.repository.mysql.ts` doit refléter
  les champs de la table `Users` (§3.2), notamment l'ENUM `Status` et la
  self-FK `BannedByUserId`. Une dérive typique à surveiller : ajouter
  `'deleted'` dans le type TypeScript sans avoir d'abord migré l'ENUM
  côté SQL.
- `backend/src/repositories/recipes/recipe.repository.mysql.ts` doit
  exposer les cinq valeurs de `Recipes.Status` (§5.1). Toute
  apparition de `'pending_review'` dans le code est un bug de
  documentation à corriger : la valeur canonique est `'pending'`.
- `backend/src/repositories/comments/comments.repository.mysql.ts` doit
  matérialiser le contrat de soft-delete : les requêtes de lecture
  publique filtrent obligatoirement sur `DeletedAt IS NULL`, et la
  contrainte `CHECK` sur `Rating` (§6.2) est doublée par une validation
  applicative côté DTO.

La règle de revue est simple : **toute modification de
`1_create_schema.sql` doit s'accompagner d'une mise à jour visible des
repositories concernés et de ce data dictionary, dans le même commit**.

## 9. Cross-références

- Diagramme entité-relation visuel :
  [`documentation/soutenance/backend/diagrams/g5-erd.md`](documentation/soutenance/backend/diagrams/g5-erd.md)
  (à venir).
- Architecture applicative, §9 Persistence (pool MySQL, transactions,
  pattern Repository) :
  ``_draft_backend_docs/architecture.md``.
- Catalogue des erreurs renvoyées par la couche persistance (codes
  applicatifs, mapping HTTP) :
  ``_draft_backend_docs/errors.md``.
- Source unique de vérité du schéma :
  ``backend/database/migrations/1_create_schema.sql``.
- Inventaire RGPD et trous résiduels (notamment l'absence de
  soft-delete utilisateur) : `_draft_securite/securite.md`, Vague 1.
- ADR-004 sur le soft-delete des commentaires : à référencer depuis le
  dossier `documentation/soutenance/backend/decisions/`.
