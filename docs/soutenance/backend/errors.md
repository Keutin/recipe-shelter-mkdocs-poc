# Catalogue des codes d'erreur

> Source : `backend/src` au commit courant.
> Tous les chemins sont relatifs à `backend/`.

## Format de réponse

Toutes les erreurs HTTP sont sérialisées par `src/middlewares/error-handler.ts:18-23` au format suivant :

```json
{
  "error": {
    "message": "string",
    "code": "STRING_CONSTANT"
  }
}
```

- Le statut HTTP provient de `err.statusCode ?? err.status ?? 500` (`error-handler.ts:13`).
- Si l'erreur n'a pas de `message`, le défaut est `"Internal server error"`.
- Si l'erreur n'a pas de `code`, le défaut est `"INTERNAL_ERROR"`.
- Toute erreur de statut >= 500 est loggée via `logger.error('[http] Internal error', err)` (`error-handler.ts:15-16`).

Le middleware ne renvoie **pas** de champ `details` : seuls `message` et `code` sont exposés au client.

Quelques controllers court-circuitent le handler en répondant directement (cf. `auth.controller.ts`, `users.controller.ts`, `recipes.controller.ts`, `rate-limiter.ts`, `not-found.ts`) mais respectent le même format `{ error: { message, code } }`.

## Hiérarchie

Le projet n'a pas de classe `AppError`. Deux classes coexistent :

### `HttpError` — erreurs métier / contrôle de flux HTTP
Définie dans `src/utils/errors.ts:1-10`.

```ts
class HttpError extends Error {
  constructor(public readonly status: number, message: string, public readonly code?: string)
}
```

Fabriques exposées (`src/utils/errors.ts:12-17`) :

| Fabrique | HTTP par défaut | Message par défaut |
| --- | --- | --- |
| `badRequest(message, code?)` | 400 | (obligatoire) |
| `unauthorized(message?, code?)` | 401 | `"Unauthorized"` |
| `forbidden(message?, code?)` | 403 | `"Forbidden"` |
| `notFound(message?, code?)` | 404 | `"Not found"` |
| `conflict(message?, code?)` | 409 | `"Conflict"` |
| `internalError(message?, code?)` | 500 | `"Internal Error"` |

### `DbError` — erreurs de persistence
Définie dans `src/db/errors.ts:1-13`. Construite par `toDbError()` (`src/db/errors.ts:15-31`) à partir d'une exception MySQL, appelée depuis `src/db/query.ts:34`. Elle n'a **pas** de `statusCode` → le handler la traite en 500 et expose le code MySQL natif (ex. `ER_DUP_ENTRY`, `ER_NO_REFERENCED_ROW_2`) tel quel dans le champ `code`.

## Catalogue par domaine

### Authentification (`AUTH_*`, `USER_BANNED`, `EMAIL_NOT_VALIDATED`)

| Code | HTTP | Levé dans | Contexte | Message FR proposé |
| --- | --- | --- | --- | --- |
| `AUTH_UNAUTHORIZED` | 401 | `api/auth/auth.controller.ts:29`, `api/users/users.controller.ts:10,28,40,54`, `api/recipes/recipes.controller.ts:11,24,74,87,101,114` | `req.auth` absent dans un handler protégé | "Non authentifié" |
| `AUTH_NO_TOKEN` | 401 | `middlewares/require-auth.ts:73` | Cookie de session manquant | "Cookie de session manquant" |
| `AUTH_BAD_TOKEN` | 401 | `middlewares/require-auth.ts:80,85,90` | Token JWT invalide, expiré, ou payload corrompu | "Token invalide ou expiré" |
| `AUTH_MISSING_FIELDS` | 400 | `services/auth/auth.service.ts:45,75`, `api/auth/auth.dto.ts:43,61` | Body register/login incomplet | "Champs manquants" |
| `AUTH_INVALID_EMAIL` | 400 | `api/auth/auth.dto.ts:45,95` | Format email invalide (register / resend) | "Email invalide" |
| `AUTH_WEAK_USERNAME` | 400 | `services/auth/auth.service.ts:55`, `api/auth/auth.dto.ts:47` | Username trop court (register) | "Nom d'utilisateur trop court" |
| `AUTH_WEAK_PASSWORD` | 400 | `services/auth/auth.service.ts:49`, `api/auth/auth.dto.ts:49` | Mot de passe trop faible (register) | "Mot de passe trop faible" |
| `AUTH_EMAIL_TAKEN` | 409 | `services/auth/auth.service.ts:52` | Email déjà utilisé à l'inscription | "Email déjà utilisé" |
| `AUTH_USERNAME_TAKEN` | 409 | `services/auth/auth.service.ts:57` | Username déjà utilisé à l'inscription | "Nom d'utilisateur déjà utilisé" |
| `AUTH_ROLE_NOT_FOUND` | 400 | `services/auth/auth.service.ts:61` | Rôle par défaut introuvable (config) | "Rôle par défaut introuvable" |
| `AUTH_INVALID_CREDENTIALS` | 401 | `services/auth/auth.service.ts:79,83` | Login : email/password invalides | "Identifiants invalides" |
| `EMAIL_NOT_VALIDATED` | 401 | `services/auth/auth.service.ts:86` | Login : email non validé | "Email non validé" |
| `USER_BANNED` | 401 / 403 | `services/auth/auth.service.ts:89` (401), `services/auth/email-validation.service.ts:66` (403) | Utilisateur banni | "Compte banni" |
| `AUTH_FORGOT_PASSWORD_INVALID_EMAIL` | 400 | `api/auth/auth.controller.ts:47` | Mot de passe oublié : email manquant | "Email requis" |
| `AUTH_RESET_PASSWORD_MISSING_TOKEN` | 400 | `api/auth/auth.dto.ts:83` | Reset password : token manquant | "Token requis" |
| `AUTH_RESET_PASSWORD_BAD_TOKEN` | 400 | `api/auth/auth.controller.ts:68` | Reset password : token invalide / expiré | "Token de réinitialisation invalide ou expiré" |
| `AUTH_RESET_PASSWORD_BAD_PASSWORD` | 400 | `api/auth/auth.controller.ts:74` | Reset password : mot de passe non conforme | "Mot de passe invalide" |
| `AUTH_EMAIL_VALIDATION_MISSING_TOKEN` | 400 | `api/auth/auth.dto.ts:71`, `services/auth/email-validation.service.ts:46` | Validation email : token absent | "Token requis" |
| `AUTH_EMAIL_VALIDATION_INVALID_TOKEN` | 400 | `services/auth/email-validation.service.ts:52,63,74` | Validation email : token invalide | "Token de validation invalide" |
| `AUTH_EMAIL_VALIDATION_TOKEN_USED` | 400 | `services/auth/email-validation.service.ts:55` | Validation email : token déjà consommé | "Token déjà utilisé" |
| `AUTH_EMAIL_VALIDATION_TOKEN_EXPIRED` | 400 | `services/auth/email-validation.service.ts:58` | Validation email : token expiré | "Token expiré" |
| `AUTH_VALIDATION_RESEND_MISSING_EMAIL` | 400 | `services/auth/email-validation.service.ts:28`, `api/auth/auth.dto.ts:93` | Renvoi email de validation : email manquant | "Email requis" |
| `AUTH_VALIDATION_RESEND_NOT_ALLOWED` | 400 | `services/auth/email-validation.service.ts:36` | Renvoi email refusé (compte déjà actif) | "Le renvoi n'est autorisé que pour les comptes inactifs" |

### Utilisateurs (`USER_*`, `USERS_*`)

| Code | HTTP | Levé dans | Contexte | Message FR proposé |
| --- | --- | --- | --- | --- |
| `USER_NOT_FOUND` | 404 | `services/users/users.service.ts:32,48,73,97,129`, `services/admin/admin.users.service.ts:38,70,81` | Utilisateur introuvable (self + admin) | "Utilisateur introuvable" |
| `USERS_UPDATE_EMAIL_BAD_BODY` | 400 | `api/users/users.dto.ts:21` | PATCH email : body non-objet | "Corps de requête invalide" |
| `USERS_UPDATE_EMAIL_MISSING_EMAIL` | 400 | `api/users/users.dto.ts:27`, `services/users/users.service.ts:63` | PATCH email : email absent | "Nouvel email requis" |
| `USERS_UPDATE_EMAIL_MISSING_PASSWORD` | 400 | `api/users/users.dto.ts:30` | PATCH email : password de confirmation absent | "Mot de passe actuel requis" |
| `USERS_UPDATE_EMAIL_INVALID_EMAIL` | 400 | `services/users/users.service.ts:68` | PATCH email : format invalide | "Format email invalide" |
| `USERS_UPDATE_EMAIL_BAD_PASSWORD` | 401 | `services/users/users.service.ts:78` | PATCH email : password de confirmation incorrect | "Mot de passe actuel incorrect" |
| `USERS_UPDATE_EMAIL_SAME_EMAIL` | 400 | `services/users/users.service.ts:81` | PATCH email : identique à l'actuel | "Le nouvel email doit être différent" |
| `USERS_UPDATE_EMAIL_ALREADY_USED` | 409 | `services/users/users.service.ts:86` | PATCH email : déjà pris | "Email déjà utilisé" |
| `USERS_UPDATE_PASSWORD_BAD_BODY` | 400 | `api/users/users.dto.ts:37` | PATCH password : body invalide | "Corps de requête invalide" |
| `USERS_UPDATE_PASSWORD_MISSING_CURRENT` | 400 | `api/users/users.dto.ts:43` | PATCH password : mot de passe actuel absent | "Mot de passe actuel requis" |
| `USERS_UPDATE_PASSWORD_MISSING_NEW` | 400 | `api/users/users.dto.ts:46` | PATCH password : nouveau mot de passe absent | "Nouveau mot de passe requis" |
| `USERS_UPDATE_PASSWORD_BAD_CURRENT` | 401 | `services/users/users.service.ts:102` | PATCH password : mot de passe actuel faux | "Mot de passe actuel incorrect" |
| `USERS_UPDATE_PASSWORD_SAME_PASSWORD` | 400 | `services/users/users.service.ts:105` | PATCH password : identique à l'actuel | "Le nouveau mot de passe doit être différent" |
| `USERS_UPDATE_PASSWORD_WEAK_PASSWORD` | 400 | `services/users/users.service.ts:110` | PATCH password : règle de robustesse non satisfaite | (message dynamique de la règle) |
| `USERS_UPDATE_USERNAME_BAD_BODY` | 400 | `api/users/users.dto.ts:53` | PATCH username : body invalide | "Corps de requête invalide" |
| `USERS_UPDATE_USERNAME_MISSING_USERNAME` | 400 | `api/users/users.dto.ts:59`, `services/users/users.service.ts:121` | PATCH username : absent | "Nouveau nom d'utilisateur requis" |
| `USERS_UPDATE_USERNAME_MISSING_PASSWORD` | 400 | `api/users/users.dto.ts:62` | PATCH username : password de confirmation absent | "Mot de passe actuel requis" |
| `USERS_UPDATE_USERNAME_WEAK_USERNAME` | 400 | `services/users/users.service.ts:124` | PATCH username : trop court | "Nom d'utilisateur trop court" |
| `USERS_UPDATE_USERNAME_BAD_PASSWORD` | 401 | `services/users/users.service.ts:134` | PATCH username : password de confirmation faux | "Mot de passe actuel incorrect" |
| `USERS_UPDATE_USERNAME_SAME_USERNAME` | 400 | `services/users/users.service.ts:137` | PATCH username : identique à l'actuel | "Le nouveau nom d'utilisateur doit être différent" |
| `USERS_UPDATE_USERNAME_ALREADY_USED` | 409 | `services/users/users.service.ts:142` | PATCH username : déjà pris | "Nom d'utilisateur déjà utilisé" |
| `USERS_MISSING_USERNAME` | 400 | `api/users/users.dto.ts:71` | DTO parsing username (route partagée) | "Nom d'utilisateur requis" |

### Recettes (`RECIPES_*`, `RECIPE_*`)

| Code | HTTP | Levé dans | Contexte | Message FR proposé |
| --- | --- | --- | --- | --- |
| `RECIPES_BAD_ID` | 400 | `api/recipes/recipes.dto.ts:106` | Paramètre `:recipeId` non entier positif | "L'id de recette doit être un entier positif" |
| `RECIPES_BAD_SLUG` | 400 | `api/recipes/recipes.dto.ts:115` | Paramètre `:slug` vide | "Slug de recette requis" |
| `RECIPES_NOT_FOUND` | 404 | `services/recipes/recipes.services.ts:61,92,104`, `services/admin/admin.recipes.services.ts:23,44,56,65` | Recette introuvable (lecture, update, archive, modération) | "Recette introuvable" |
| `NOT_FOUND` | 404 | `api/recipes/recipes.controller.ts:63` | Recette introuvable au niveau controller | "Recette introuvable" |
| `RECIPES_ACCESS_DENIED` | 403 | `services/recipes/recipes.services.ts:64,95` | Lecture d'une recette non publiée par un non-propriétaire | "Accès à la recette refusé" |
| `RECIPES_ARCHIVE_FORBIDDEN` | 403 | `services/recipes/recipes.services.ts:67`, `services/admin/admin.recipes.services.ts:47` | Recette non archivable dans son état actuel | "Recette non archivable" |
| `RECIPES_EDIT_FORBIDDEN` | 403 | `services/recipes/recipes.services.ts:107` | Recette non éditable dans son état actuel | "Recette non éditable" |
| `RECIPES_MODERATE_FORBIDDEN` | 403 | `services/admin/admin.recipes.services.ts:68` | Recette non modérable dans son état actuel | "Recette non modérable" |
| `RECIPES_CREATE_BAD_BODY` | 400 | `api/recipes/recipes.dto.ts:72` (préfixe) | POST recette : body non-objet | "Corps de requête invalide" |
| `RECIPES_CREATE_MISSING_TITLE` | 400 | `api/recipes/recipes.dto.ts:74` (préfixe) | POST recette : titre absent | "Le titre est requis" |
| `RECIPES_CREATE_WEAK_TITLE` | 400 | `api/recipes/recipes.dto.ts:77` (préfixe) | POST recette : titre < 5 caractères | "Le titre doit faire au moins 5 caractères" |
| `RECIPES_CREATE_BAD_CATEGORY` | 400 | `api/recipes/recipes.dto.ts:79` (préfixe) | POST recette : `categoryId` invalide | "La catégorie doit être un nombre" |
| `RECIPES_CREATE_BAD_DESCRIPTION` | 400 | `api/recipes/recipes.dto.ts:80` (préfixe) | POST recette : description invalide | "La description doit être une chaîne" |
| `RECIPES_CREATE_BAD_COVER_IMAGE_URL` | 400 | `api/recipes/recipes.dto.ts:81` | POST recette : URL d'image invalide | "L'URL de l'image de couverture doit être une chaîne ou null" |
| `RECIPES_CREATE_BAD_PREP_TIME` | 400 | `api/recipes/recipes.dto.ts:82` | POST recette : `prepTimeMinutes` invalide | "Le temps de préparation doit être un nombre" |
| `RECIPES_CREATE_BAD_REST_TIME` | 400 | `api/recipes/recipes.dto.ts:83` | POST recette : `restTimeMinutes` invalide | "Le temps de repos doit être un nombre" |
| `RECIPES_CREATE_BAD_COOK_TIME` | 400 | `api/recipes/recipes.dto.ts:84` | POST recette : `cookTimeMinutes` invalide | "Le temps de cuisson doit être un nombre" |
| `RECIPES_CREATE_BAD_SERVINGS` | 400 | `api/recipes/recipes.dto.ts:85` | POST recette : `servings` invalide | "Le nombre de portions doit être un nombre" |
| `RECIPES_CREATE_BAD_TAGS` | 400 | `api/recipes/recipes.dto.ts:86` | POST recette : `tagIds` non-tableau | "Les tags doivent être un tableau" |
| `RECIPES_CREATE_BAD_TAG_ID` | 400 | `api/recipes/recipes.dto.ts:67` | POST recette : tag id non entier positif | "Le tagId doit être un entier positif" |
| `RECIPES_CREATE_BAD_INGREDIENTS` | 400 | `api/recipes/recipes.dto.ts:87` | POST recette : `ingredients` non-tableau | "Les ingrédients doivent être un tableau" |
| `RECIPES_CREATE_BAD_INGREDIENT` | 400 | `api/recipes/recipes.dto.ts:32` | POST recette : item ingrédient mal formé | "Ingrédient #N invalide" |
| `RECIPES_CREATE_BAD_INGREDIENT_ID` | 400 | `api/recipes/recipes.dto.ts:39` | POST recette : `ingredientId` manquant/invalide | "ingredientId doit être un nombre" |
| `RECIPES_CREATE_BAD_INGREDIENT_QUANTITY` | 400 | `api/recipes/recipes.dto.ts:40` | POST recette : quantité invalide | "La quantité doit être un nombre" |
| `RECIPES_CREATE_BAD_INGREDIENT_UNIT` | 400 | `api/recipes/recipes.dto.ts:34` | POST recette : unité invalide | "L'unité doit être une chaîne ou null" |
| `RECIPES_CREATE_BAD_INGREDIENT_NOTE` | 400 | `api/recipes/recipes.dto.ts:35` | POST recette : note invalide | "La note doit être une chaîne" |
| `RECIPES_CREATE_BAD_INGREDIENT_SORT_ORDER` | 400 | `api/recipes/recipes.dto.ts:36` | POST recette : `sortOrder` invalide | "sortOrder doit être un nombre" |
| `RECIPES_CREATE_BAD_STEPS` | 400 | `api/recipes/recipes.dto.ts:88` | POST recette : `steps` non-tableau | "Les étapes doivent être un tableau" |
| `RECIPES_CREATE_BAD_STEP` | 400 | `api/recipes/recipes.dto.ts:49` | POST recette : item step mal formé | "Étape #N invalide" |
| `RECIPES_CREATE_BAD_STEP_NUMBER` | 400 | `api/recipes/recipes.dto.ts:52` | POST recette : `stepNumber` invalide | "stepNumber doit être un nombre" |
| `RECIPES_CREATE_BAD_STEP_DESCRIPTION` | 400 | `api/recipes/recipes.dto.ts:53` | POST recette : description d'étape absente | "La description de l'étape est requise" |
| `RECIPES_CREATE_BAD_EQUIPMENTS` | 400 | `api/recipes/recipes.dto.ts:89` | POST recette : `equipments` non-tableau | "Les équipements doivent être un tableau" |
| `RECIPES_CREATE_BAD_EQUIPMENT` | 400 | `api/recipes/recipes.dto.ts:59` | POST recette : item équipement invalide | "Équipement #N invalide" |
| `RECIPES_CREATE_BAD_EQUIPMENT_ID` | 400 | `api/recipes/recipes.dto.ts:62` | POST recette : `equipmentId` manquant | "equipmentId doit être un nombre" |
| `RECIPES_UPDATE_BAD_BODY` | 400 | `api/recipes/recipes.dto.ts:72` (préfixe) | PATCH recette : body invalide | "Corps de requête invalide" |
| `RECIPES_UPDATE_MISSING_TITLE` | 400 | `api/recipes/recipes.dto.ts:74` (préfixe) | PATCH recette : titre absent | "Le titre est requis" |
| `RECIPES_UPDATE_WEAK_TITLE` | 400 | `api/recipes/recipes.dto.ts:77` (préfixe) | PATCH recette : titre < 5 caractères | "Le titre doit faire au moins 5 caractères" |
| `RECIPES_UPDATE_BAD_CATEGORY` | 400 | `api/recipes/recipes.dto.ts:79` (préfixe) | PATCH recette : catégorie invalide | "La catégorie doit être un nombre" |
| `RECIPES_UPDATE_BAD_DESCRIPTION` | 400 | `api/recipes/recipes.dto.ts:80` | PATCH recette : description invalide | "La description doit être une chaîne" |
| `RECIPES_UPDATE_BAD_COVER_IMAGE_URL` | 400 | `api/recipes/recipes.dto.ts:81` | PATCH recette : URL image invalide | "URL de couverture invalide" |
| `RECIPES_UPDATE_BAD_PREP_TIME` | 400 | `api/recipes/recipes.dto.ts:82` | PATCH recette : prep time invalide | "Temps de préparation invalide" |
| `RECIPES_UPDATE_BAD_REST_TIME` | 400 | `api/recipes/recipes.dto.ts:83` | PATCH recette : rest time invalide | "Temps de repos invalide" |
| `RECIPES_UPDATE_BAD_COOK_TIME` | 400 | `api/recipes/recipes.dto.ts:84` | PATCH recette : cook time invalide | "Temps de cuisson invalide" |
| `RECIPES_UPDATE_BAD_SERVINGS` | 400 | `api/recipes/recipes.dto.ts:85` | PATCH recette : servings invalide | "Nombre de portions invalide" |
| `RECIPES_UPDATE_BAD_TAGS` | 400 | `api/recipes/recipes.dto.ts:86` | PATCH recette : tagIds invalide | "Les tags doivent être un tableau" |
| `RECIPES_UPDATE_BAD_INGREDIENTS` | 400 | `api/recipes/recipes.dto.ts:87` | PATCH recette : ingredients invalide | "Les ingrédients doivent être un tableau" |
| `RECIPES_UPDATE_BAD_STEPS` | 400 | `api/recipes/recipes.dto.ts:88` | PATCH recette : steps invalide | "Les étapes doivent être un tableau" |
| `RECIPES_UPDATE_BAD_EQUIPMENTS` | 400 | `api/recipes/recipes.dto.ts:89` | PATCH recette : equipments invalide | "Les équipements doivent être un tableau" |
| `RECIPES_FEED_BAD_QUERY` | 400 | `api/recipes/recipes.dto.ts:122` | GET feed : query non-objet | "Query invalide" |
| `RECIPES_FEED_BAD_LIMIT` | 400 | `api/recipes/recipes.dto.ts:127` | GET feed : `limit` invalide | "Limit doit être un entier positif" |
| `RECIPES_SEARCH_BAD_QUERY` | 400 | `api/recipes/recipes.dto.ts:172` | GET search : query non-objet | "Query invalide" |
| `RECIPES_SEARCH_BAD_Q` | 400 | `api/recipes/recipes.dto.ts:178` | GET search : `q` non-string | "Le terme de recherche doit être une chaîne" |
| `RECIPES_SEARCH_BAD_CATEGORY` | 400 | `api/recipes/recipes.dto.ts:186` | GET search : `categoryId` invalide | "categoryId doit être un entier positif" |
| `RECIPES_SEARCH_BAD_TAGS` | 400 | `api/recipes/recipes.dto.ts:163` | GET search : `tagIds` invalides | "Les ids de tags doivent être une liste séparée par virgules d'entiers positifs" |
| `RECIPES_SEARCH_BAD_INGREDIENTS` | 400 | `api/recipes/recipes.dto.ts:167` | GET search : `ingredientIds` invalides | "Les ids d'ingrédients doivent être une liste séparée par virgules d'entiers positifs" |
| `RECIPES_SEARCH_BAD_TOTAL_TIME` | 400 | `api/recipes/recipes.dto.ts:201` | GET search : `maxTotalTimeMinutes` invalide | "Le temps total doit être un entier positif" |
| `RECIPES_PAGINATION_BAD_QUERY` | 400 | `utils/pagination.ts:27` via `recipes.controller.ts:16,37,45` | Pagination recettes : query non-objet | "Query invalide" |
| `RECIPES_PAGINATION_BAD_PAGE` | 400 | `utils/pagination.ts:30` | Pagination recettes : `page` invalide | "Page doit être un entier positif" |
| `RECIPES_PAGINATION_BAD_LIMIT` | 400 | `utils/pagination.ts:31` | Pagination recettes : `limit` invalide | "Limit doit être un entier positif" |

### Modération (admin) (`ADMIN_*`)

| Code | HTTP | Levé dans | Contexte | Message FR proposé |
| --- | --- | --- | --- | --- |
| `ADMIN_ACCESS_REQUIRED` | 403 | `middlewares/require-admin.ts:8` | Route admin appelée par un non-admin | "Accès administrateur requis" |
| `ADMIN_USERS_BAD_ID` | 400 | `api/admin/admin.users.dto.ts:11` | Paramètre `:userId` invalide | "L'id utilisateur doit être un entier positif" |
| `ADMIN_USERS_BAN_BAD_BODY` | 400 | `api/admin/admin.users.dto.ts:17,26` | POST ban : body invalide | "Corps de requête invalide" |
| `ADMIN_USERS_BAN_MISSING_REASON` | 400 | `api/admin/admin.users.dto.ts:17,28` ; `services/admin/admin.users.service.ts:93` | POST ban : motif absent | "Le motif de ban est requis" |
| `ADMIN_USERS_BAN_REASON_TOO_SHORT` | 400 | `api/admin/admin.users.dto.ts:31` ; `services/admin/admin.users.service.ts:98` | POST ban : motif < 10 caractères | "Le motif de ban doit faire au moins 10 caractères" |
| `ADMIN_USERS_BAN_REASON_TOO_LONG` | 400 | `api/admin/admin.users.dto.ts:34` ; `services/admin/admin.users.service.ts:104` | POST ban : motif > 1000 caractères | "Le motif de ban doit faire au plus 1000 caractères" |
| `ADMIN_USERS_BAN_SELF_FORBIDDEN` | 403 | `services/admin/admin.users.service.ts:65` | Un admin tente de se bannir lui-même | "Un administrateur ne peut pas se bannir lui-même" |
| `ADMIN_USERS_UNBAN_BAD_BODY` | 400 | `api/admin/admin.users.dto.ts:21,26` | POST unban : body invalide | "Corps de requête invalide" |
| `ADMIN_USERS_UNBAN_MISSING_REASON` | 400 | `api/admin/admin.users.dto.ts:21,28` ; `services/admin/admin.users.service.ts:93` | POST unban : motif absent | "Le motif de débannissement est requis" |
| `ADMIN_USERS_UNBAN_REASON_TOO_SHORT` | 400 | `api/admin/admin.users.dto.ts:31` ; `services/admin/admin.users.service.ts:98` | POST unban : motif < 10 caractères | "Le motif doit faire au moins 10 caractères" |
| `ADMIN_USERS_UNBAN_REASON_TOO_LONG` | 400 | `api/admin/admin.users.dto.ts:34` ; `services/admin/admin.users.service.ts:104` | POST unban : motif > 1000 caractères | "Le motif doit faire au plus 1000 caractères" |
| `ADMIN_RECIPES_REJECT_BAD_BODY` | 400 | `api/admin/admin.recipes.dto.ts:9` | POST reject recette : body invalide | "Corps de requête invalide" |
| `ADMIN_RECIPES_REJECT_MISSING_REASON` | 400 | `api/admin/admin.recipes.dto.ts:14` | POST reject recette : motif absent | "Le motif de rejet est requis" |
| `ADMIN_COMMENTS_BAD_ID` | 400 | `api/admin/admin.comments.dto.ts:13` | Param admin comment id invalide | "L'id du commentaire doit être un entier positif" |
| `ADMIN_COMMENTS_UPDATE_BAD_BODY` | 400 | `api/admin/admin.comments.dto.ts:20` | PATCH commentaire admin : body invalide | "Corps de requête invalide" |
| `ADMIN_COMMENTS_UPDATE_BAD_RATING` | 400 | `api/admin/admin.comments.dto.ts:26` | PATCH commentaire admin : rating invalide | "La note doit être entre 1 et 5" |

### Commentaires (`COMMENT_*`, `COMMENTS_*`)

| Code | HTTP | Levé dans | Contexte | Message FR proposé |
| --- | --- | --- | --- | --- |
| `COMMENTS_BAD_ID` | 400 | `api/comments/comments.dto.ts:66` | Param `:commentId` invalide | "L'id du commentaire doit être un entier positif" |
| `COMMENTS_BAD_RECIPE_ID` | 400 | `api/comments/comments.dto.ts:70` | Param `:recipeId` invalide (route commentaires) | "L'id de recette doit être un entier positif" |
| `COMMENT_NOT_FOUND` | 404 | `services/comments/comments.service.ts:25,42` | Commentaire introuvable (user-facing : update / delete) | "Commentaire introuvable" |
| `COMMENTS_NOT_FOUND` | 404 | `services/comments/comments.service.ts:54`, `services/admin/admin.comments.services.ts:29,34,43,48,57,62,71,80` | Aucun commentaire trouvé (liste / admin) | "Commentaire(s) introuvable(s)" |
| `COMMENTS_PARENT_NOT_FOUND` | 404 | `services/comments/comments.service.ts:63` | POST réponse : commentaire parent introuvable | "Commentaire parent introuvable" |
| `COMMENT_ACCESS_DENIED` | 403 | `services/comments/comments.service.ts:28,33,45` | Action sur commentaire d'un autre user | "Accès au commentaire refusé" |
| `COMMENT_CANNOT_BE_CREATED` | 500 | `services/comments/comments.service.ts:16` | Création échouée (anomalie persistance) | "Impossible de créer le commentaire" |
| `COMMENTS_CREATE_BAD_BODY` | 400 | `api/comments/comments.dto.ts:44` | POST commentaire : body invalide | "Corps de requête invalide" |
| `COMMENTS_CREATE_MISSING_COMMENT` | 400 | `api/comments/comments.dto.ts:46` | POST commentaire : texte absent | "Le commentaire est requis" |
| `COMMENTS_CREATE_BAD_PARENT_COMMENT_ID` | 400 | `api/comments/comments.dto.ts:34,37` | POST commentaire : parent id invalide | "L'id du commentaire parent doit être un entier positif ou null" |
| `COMMENTS_CREATE_BAD_RATING` | 400 | `api/comments/comments.dto.ts:25,28` (préfixe) | POST commentaire : note invalide | "La note doit être un entier entre 1 et 5" |
| `COMMENTS_CREATE_REPLY_WITH_RATING` | 400 | `api/comments/comments.dto.ts:51`, `services/comments/comments.service.ts:66` | POST commentaire : réponse avec note (interdit) | "Les commentaires de type réponse ne peuvent pas avoir de note" |
| `COMMENTS_CREATE_NESTED_REPLY` | 400 | `services/comments/comments.service.ts:66` | POST commentaire : profondeur > 1 niveau | "Une seule profondeur de réponse est autorisée" |
| `COMMENTS_UPDATE_BAD_BODY` | 400 | `api/comments/comments.dto.ts:58` | PATCH commentaire : body invalide | "Corps de requête invalide" |
| `COMMENTS_UPDATE_MISSING_COMMENT` | 400 | `api/comments/comments.dto.ts:60` | PATCH commentaire : texte absent | "Le commentaire est requis" |
| `COMMENTS_UPDATE_BAD_RATING` | 400 | `api/comments/comments.dto.ts:25,28` (préfixe) | PATCH commentaire : note invalide | "La note doit être un entier entre 1 et 5" |

### Favoris (`FAVORITE_*`, `FAVORITES_*`)

| Code | HTTP | Levé dans | Contexte | Message FR proposé |
| --- | --- | --- | --- | --- |
| `RECIPE_BAD_ID` | 400 | `api/favorites/favorites.dto.ts:7` | Param recette favori invalide | "L'id de recette doit être un entier positif" |
| `FAVORITE_CANNOT_BE_CREATED` | 500 | `services/favorites/favorites.service.ts:15` | Insertion favori échouée | "Impossible d'ajouter le favori" |
| `FAVORITE_CANNOT_BE_DELETED` | 500 | `services/favorites/favorites.service.ts:24` | Suppression favori échouée | "Impossible de supprimer le favori" |
| `FAVORITES_PAGINATION_BAD_QUERY` | 400 | `utils/pagination.ts:27` via `api/favorites/favorites.controller.ts:24` | Pagination favoris : query invalide | "Query invalide" |
| `FAVORITES_PAGINATION_BAD_PAGE` | 400 | `utils/pagination.ts:30` | Pagination favoris : page invalide | "Page doit être un entier positif" |
| `FAVORITES_PAGINATION_BAD_LIMIT` | 400 | `utils/pagination.ts:31` | Pagination favoris : limit invalide | "Limit doit être un entier positif" |

### Catégories, Tags, Ingrédients, Équipements (référentiels)

| Code | HTTP | Levé dans | Contexte | Message FR proposé |
| --- | --- | --- | --- | --- |
| `CATEGORIES_NOT_FOUND` | 404 | `services/category/category.service.ts:13` | Liste catégories vide | "Aucune catégorie trouvée" |
| `CATEGORY_NOT_FOUND` | 404 | `services/category/category.service.ts:22` | Catégorie par id introuvable | "Catégorie introuvable" |
| `CATEGORY_BAD_ID` | 400 | `api/category/category.dto.ts:7` | Param catégorie id invalide | "L'id de catégorie doit être un entier positif" |
| `TAGS_NOT_FOUND` | 404 | `services/tag/tags.service.ts:13` | Liste tags vide | "Aucun tag trouvé" |
| `TAG_NOT_FOUND` | 404 | `services/tag/tags.service.ts:22` | Tag par id introuvable | "Tag introuvable" |
| `TAG_BAD_ID` | 400 | `api/tag/tags.dto.ts:7` | Param tag id invalide | "L'id de tag doit être un entier positif" |
| `INGREDIENTS_NOT_FOUND` | 404 | `services/ingredients/ingredients.service.ts:13` | Liste ingrédients vide | "Aucun ingrédient trouvé" |
| `INGREDIENT_NOT_FOUND` | 404 | `services/ingredients/ingredients.service.ts:22` | Ingrédient par id introuvable | "Ingrédient introuvable" |
| `INGREDIENT_BAD_ID` | 400 | `api/ingredients/ingredients.dto.ts:7` | Param ingrédient id invalide | "L'id d'ingrédient doit être un entier positif" |
| `EQUIPMENTS_NOT_FOUND` | 404 | `services/equipments/equipments.service.ts:13` | Liste équipements vide | "Aucun équipement trouvé" |
| `EQUIPMENT_NOT_FOUND` | 404 | `services/equipments/equipments.service.ts:22` | Équipement par id introuvable | "Équipement introuvable" |
| `EQUIPMENT_BAD_ID` | 400 | `api/equipments/equipments.dto.ts:7` | Param équipement id invalide | "L'id d'équipement doit être un entier positif" |

### Contact (`CONTACT_*`)

| Code | HTTP | Levé dans | Contexte | Message FR proposé |
| --- | --- | --- | --- | --- |
| `CONTACT_MISSING_FIELDS` | 400 | `api/contact/contact.dto.ts:20,25` | POST contact : champs absents | "Champs manquants" |
| `CONTACT_INVALID_EMAIL` | 400 | `api/contact/contact.dto.ts:51` | POST contact : email invalide | "Email invalide" |
| `CONTACT_<FIELD>_TOO_SHORT` | 400 | `api/contact/contact.dto.ts:32` (dynamique) | POST contact : champ trop court | "<Champ> trop court" |
| `CONTACT_<FIELD>_TOO_LONG` | 400 | `api/contact/contact.dto.ts:35` (dynamique) | POST contact : champ trop long | "<Champ> trop long" |
| `CONTACT_SEND_FAILED` | 500 | `services/mail/mail.service.ts:170` | Échec d'envoi du message de contact | "Impossible d'envoyer le message de contact" |

> `<FIELD>` = upper-case du nom de champ envoyé (ex. `CONTACT_NAME_TOO_SHORT`, `CONTACT_MESSAGE_TOO_LONG`). Voir `api/contact/contact.dto.ts` pour la liste exacte.

### Pagination générique (`PAGINATION_*`)

| Code | HTTP | Levé dans | Contexte | Message FR proposé |
| --- | --- | --- | --- | --- |
| `PAGINATION_BAD_QUERY` | 400 | `utils/pagination.ts:27` | Pagination par défaut : query invalide | "Query invalide" |
| `PAGINATION_BAD_PAGE` | 400 | `utils/pagination.ts:30` | Pagination par défaut : page invalide | "Page doit être un entier positif" |
| `PAGINATION_BAD_LIMIT` | 400 | `utils/pagination.ts:31,55` | Pagination par défaut : limit invalide | "Limit doit être un entier positif" |
| `PAGINATION_BAD_OFFSET` | 400 | `utils/pagination.ts:56` | Offset SQL hors plage | "Offset doit être un entier positif ou nul" |

> Le préfixe est paramétrable. Valeurs observées : `PAGINATION` (défaut), `RECIPES_PAGINATION`, `FAVORITES_PAGINATION`.

### Infrastructure HTTP (`RATE_LIMIT`, `ROUTE_NOT_FOUND`, `INTERNAL_ERROR`)

| Code | HTTP | Levé dans | Contexte | Message FR proposé |
| --- | --- | --- | --- | --- |
| `ROUTE_NOT_FOUND` | 404 | `middlewares/not-found.ts:4-8` | Aucune route Express ne matche | "Route introuvable" |
| `RATE_LIMIT` | 429 | `middlewares/rate-limiter.ts:41-46` | Quota du `rateLimiter` dépassé | "Trop de requêtes" |
| `INTERNAL_ERROR` | 500 | défaut `error-handler.ts:21` | Erreur sans `code` explicite | "Erreur interne du serveur" |

### Mail (`MAIL_*`)

| Code | HTTP | Levé dans | Contexte | Message FR proposé |
| --- | --- | --- | --- | --- |
| `MAIL_SEND_FAILED` | 500 | `services/mail/mail.service.ts:86,96` | Échec d'envoi générique (transport SMTP) | "Impossible d'envoyer l'email" |

### Persistence (DB) — codes MySQL relayés

`DbError` (`src/db/errors.ts`) n'ajoute **pas** de statut HTTP. Le handler renvoie donc 500 avec, dans le champ `code`, **le code MySQL natif** copié depuis l'erreur du driver. Codes susceptibles d'apparaître :

| Code MySQL | HTTP effectif | Cause typique |
| --- | --- | --- |
| `ER_DUP_ENTRY` | 500 | Violation d'index unique (devrait idéalement remonter en 409) |
| `ER_NO_REFERENCED_ROW_2` | 500 | Insertion référençant une clé étrangère inexistante |
| `ER_ROW_IS_REFERENCED_2` | 500 | Suppression d'une ligne référencée |
| `ER_BAD_FIELD_ERROR` | 500 | Bug SQL côté backend |
| `ER_PARSE_ERROR` | 500 | Bug SQL côté backend |
| `ER_LOCK_DEADLOCK` | 500 | Deadlock InnoDB (à retry si idempotent) |
| `PROTOCOL_CONNECTION_LOST`, `ECONNREFUSED`, `ETIMEDOUT` | 500 | Indisponibilité MySQL |

Voir "Inconsistances" : aucune traduction `ER_DUP_ENTRY → conflict()` n'est faite aujourd'hui.

## Mapping HTTP (récap)

| HTTP | Codes |
| --- | --- |
| 400 | Tous les `*_BAD_*`, `*_MISSING_*`, `*_WEAK_*`, `*_INVALID_*`, `*_TOO_SHORT`, `*_TOO_LONG`, `*_SAME_*`, `AUTH_MISSING_FIELDS`, `AUTH_INVALID_EMAIL`, `AUTH_EMAIL_VALIDATION_*`, `AUTH_RESET_PASSWORD_*`, `AUTH_FORGOT_PASSWORD_INVALID_EMAIL`, `AUTH_VALIDATION_RESEND_*`, `AUTH_ROLE_NOT_FOUND`, `CONTACT_*`, `COMMENTS_CREATE_REPLY_WITH_RATING`, `COMMENTS_CREATE_NESTED_REPLY` |
| 401 | `AUTH_UNAUTHORIZED`, `AUTH_NO_TOKEN`, `AUTH_BAD_TOKEN`, `AUTH_INVALID_CREDENTIALS`, `EMAIL_NOT_VALIDATED`, `USER_BANNED` (login), `USERS_UPDATE_EMAIL_BAD_PASSWORD`, `USERS_UPDATE_PASSWORD_BAD_CURRENT`, `USERS_UPDATE_USERNAME_BAD_PASSWORD` |
| 403 | `ADMIN_ACCESS_REQUIRED`, `ADMIN_USERS_BAN_SELF_FORBIDDEN`, `USER_BANNED` (validation email), `RECIPES_ACCESS_DENIED`, `RECIPES_ARCHIVE_FORBIDDEN`, `RECIPES_EDIT_FORBIDDEN`, `RECIPES_MODERATE_FORBIDDEN`, `COMMENT_ACCESS_DENIED` |
| 404 | `ROUTE_NOT_FOUND`, `NOT_FOUND`, `USER_NOT_FOUND`, `RECIPES_NOT_FOUND`, `COMMENT_NOT_FOUND`, `COMMENTS_NOT_FOUND`, `COMMENTS_PARENT_NOT_FOUND`, `CATEGORIES_NOT_FOUND`, `CATEGORY_NOT_FOUND`, `TAGS_NOT_FOUND`, `TAG_NOT_FOUND`, `INGREDIENTS_NOT_FOUND`, `INGREDIENT_NOT_FOUND`, `EQUIPMENTS_NOT_FOUND`, `EQUIPMENT_NOT_FOUND` |
| 409 | `AUTH_EMAIL_TAKEN`, `AUTH_USERNAME_TAKEN`, `USERS_UPDATE_EMAIL_ALREADY_USED`, `USERS_UPDATE_USERNAME_ALREADY_USED` |
| 429 | `RATE_LIMIT` |
| 500 | `INTERNAL_ERROR` (défaut), `COMMENT_CANNOT_BE_CREATED`, `FAVORITE_CANNOT_BE_CREATED`, `FAVORITE_CANNOT_BE_DELETED`, `MAIL_SEND_FAILED`, `CONTACT_SEND_FAILED`, tous les codes MySQL relayés via `DbError` |

## Convention de nommage

Observée dans le code :

- **Casse** : `SCREAMING_SNAKE_CASE` uniquement, séparateur `_`.
- **Préfixe par domaine** : `AUTH_`, `USERS_`, `USER_` (entité singulière), `RECIPES_`, `RECIPE_`, `COMMENTS_`, `COMMENT_`, `ADMIN_<DOMAIN>_`, `CONTACT_`, `MAIL_`, `PAGINATION_`, `<DOMAIN>_PAGINATION_`, `<RÉFÉRENTIEL>_` (`CATEGORY_`, `TAG_`, `INGREDIENT_`, `EQUIPMENT_`).
- **Verbe / action** : `<DOMAIN>_<ACTION>_<RAISON>` (ex. `USERS_UPDATE_EMAIL_BAD_PASSWORD`, `ADMIN_USERS_BAN_REASON_TOO_SHORT`).
- **Raisons standardisées** :
  - `BAD_<CHAMP>` : type / format invalide.
  - `MISSING_<CHAMP>` : champ requis absent.
  - `WEAK_<CHAMP>` : règle de robustesse non satisfaite (password, username, title).
  - `INVALID_<CHAMP>` : valeur sémantiquement incorrecte.
  - `<X>_TOO_SHORT` / `<X>_TOO_LONG` : violations de longueur.
  - `<X>_NOT_FOUND` : ressource absente (404).
  - `<X>_ACCESS_DENIED` / `<X>_FORBIDDEN` : autorisation refusée (403).
  - `<X>_ALREADY_USED` / `<X>_TAKEN` : conflit d'unicité (409).
  - `<X>_CANNOT_BE_<VERB>` : échec persistance (500).
  - `<X>_SEND_FAILED` : échec I/O externe (500).
- **Singulier vs pluriel** : le code respecte le nombre d'instances ciblées (`USER_NOT_FOUND` pour un user vs `CATEGORIES_NOT_FOUND` pour une liste vide).

## Inconsistances

1. **Pluralisation incohérente sur "not found"** :
   - `USER_NOT_FOUND` (singulier) mais utilisé pour la cible d'une action admin sur **un** utilisateur — OK.
   - `RECIPES_NOT_FOUND` est utilisé pour des recherches d'**une** recette (`services/recipes/recipes.services.ts:61,92,104`). Devrait être `RECIPE_NOT_FOUND`. Idem `COMMENTS_NOT_FOUND` côté admin sur des actions ciblant **un** commentaire (`services/admin/admin.comments.services.ts` 8 occurrences).
   - Inversement, `COMMENT_NOT_FOUND` (singulier) est utilisé côté user pour la même sémantique (`services/comments/comments.service.ts:25,42`) — divergence user/admin.

2. **`NOT_FOUND` non préfixé** : `api/recipes/recipes.controller.ts:63` renvoie `NOT_FOUND` (au lieu de `RECIPES_NOT_FOUND`). Cassé du naming, et conflit potentiel avec un futur code global.

3. **`USER_BANNED` produit deux statuts HTTP différents** :
   - 401 dans `services/auth/auth.service.ts:89` (login).
   - 403 dans `services/auth/email-validation.service.ts:66` (validation email d'un banni).
   Choix conscient ou bug ? À clarifier — un bannissement à la validation devrait sans doute être 403 partout (ou 401 partout).

4. **Codes MySQL fuités en HTTP 500** : `DbError` ne traduit aucun code MySQL en `HttpError`. Un `ER_DUP_ENTRY` sur un insert (ex. course condition register / favori) renvoie 500 + code MySQL au client, alors qu'un 409 + `*_ALREADY_USED` serait attendu. Recommandation : ajouter un mapping dans `db/errors.ts` ou un wrapper côté service.

5. **Messages 100 % en anglais** : tous les `message` levés sont en anglais. Le frontend devra traduire via i18n (clé = `code`), ou bien centraliser les traductions backend-side. Une colonne "Message FR proposé" est donnée ci-dessus pour anticiper ce travail.

6. **Codes "dynamiques" via préfixe** : les codes contact `CONTACT_<FIELD>_TOO_SHORT|TOO_LONG` (`api/contact/contact.dto.ts:32,35`) sont concaténés à partir du nom de champ runtime. Risque : un champ inconnu produira un code non documenté. Mieux vaut figer la liste des champs admissibles.

7. **Codes auth dupliqués entre DTO et service** : `AUTH_MISSING_FIELDS`, `AUTH_INVALID_EMAIL`, `AUTH_WEAK_USERNAME`, `AUTH_WEAK_PASSWORD`, `USERS_UPDATE_EMAIL_MISSING_EMAIL`, `AUTH_EMAIL_VALIDATION_MISSING_TOKEN`, `AUTH_VALIDATION_RESEND_MISSING_EMAIL` sont levés à la fois dans le DTO et dans le service. Cohérent fonctionnellement (double rempart) mais à documenter pour le frontend.

8. **Pas d'`AppError`** : la note initiale mentionnait `AppError`. La classe n'existe pas — c'est `HttpError` (+ `DbError`). Le naming `HttpError` est plus exact (couplé HTTP), à valider lors du futur portage si une autre couche de transport est envisagée.

9. **`asyncHandler` + court-circuits manuels** : plusieurs controllers (`auth.controller.ts`, `users.controller.ts`, `recipes.controller.ts`) renvoient `res.status(...).json(...)` au lieu de `throw unauthorized(...)`. Comportement équivalent côté wire, mais contourne le logging centralisé et complique le suivi des codes.
