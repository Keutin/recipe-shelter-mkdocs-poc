# G2 — Pipeline middleware Express

Ce diagramme detaille le parcours d'une requete HTTP authentifiee a travers la chaine de middlewares Express, telle que cablee dans `backend/src/app.ts` (lignes 70 a 150). Les branches conditionnelles indiquent les points ou la requete peut etre court-circuitee (erreurs 401, 403, 429, 404) avant d'atteindre le controller.

```mermaid
flowchart TD
    Req[Requete HTTP entrante]
    Cors[cors<br/>verifie origin vs corsAllowedOrigins<br/>credentials: true]
    CookieP[cookieParser<br/>peuple req.cookies]
    Json[express.json<br/>parse le body JSON]
    Route{Router matching<br/>/api/v1/...}

    RateLim{rateLimiter ?<br/>endpoints sensibles<br/>login, forgot, contact}
    Auth{requireAuth ?<br/>lit cookie auth_token<br/>verifie JWT<br/>charge user via repo}
    Admin{requireAdmin ?<br/>roleId === 1}
    Ctrl[Controller<br/>wrap asyncHandler]
    Svc[Service]
    Repo[Repository]
    DB[(MySQL pool)]
    Mapper[Mapper row vers DTO]
    Resp[Reponse JSON<br/>200 / 201 / 204]

    NotFound[notFound middleware]
    ErrHandler[errorHandler<br/>JSON error code message]

    R401[401 AUTH_NO_TOKEN<br/>ou AUTH_BAD_TOKEN]
    R403[403 ADMIN_ACCESS_REQUIRED]
    R429[429 RATE_LIMIT]
    R404[404 ROUTE_NOT_FOUND]

    Req --> Cors --> CookieP --> Json --> Route
    Route -- match --> RateLim
    Route -- aucune route --> NotFound --> ErrHandler

    RateLim -- ok --> Auth
    RateLim -- quota depasse --> R429

    Auth -- token valide --> Admin
    Auth -- token absent ou invalide --> R401 --> ErrHandler
    Auth -- route publique, skip --> Admin

    Admin -- ok ou route non-admin --> Ctrl
    Admin -- roleId different --> R403 --> ErrHandler

    Ctrl --> Svc --> Repo --> DB
    DB --> Mapper --> Resp

    Ctrl -. throw .-> ErrHandler
    Svc -. throw .-> ErrHandler
    Repo -. throw .-> ErrHandler
```

## Legende

- **Losanges** : decisions ou middlewares optionnels (montes par route, pas globalement). Un endpoint public bypasse `requireAuth` ; un endpoint non-admin bypasse `requireAdmin` ; seuls les endpoints sensibles ont un `rateLimiter`.
- **Fleches pointillees** : tout `throw` (ou `next(err)`) dans un controller, service ou repository est capture par `asyncHandler` puis route vers `errorHandler` qui renvoie un JSON normalise `{ error: { message, code } }`.
- **Ordre global fixe** dans `app.ts` : `cors` -> `cookieParser` -> `express.json` -> routers -> `notFound` -> `errorHandler`. L'ordre des middlewares **par route** (`rateLimiter`, `requireAuth`, `requireAdmin`) est defini dans chaque fichier `*.routes.ts`.

## Table des middlewares

| Nom | Fichier | Responsabilite |
| --- | --- | --- |
| `cors` | dependance npm, configure dans `src/app.ts` | Verifie l'`Origin` contre la whitelist `env.http.corsAllowedOrigins`, autorise les credentials (cookie). Refuse si `*` est dans la liste. |
| `cookieParser` | dependance npm, configure dans `src/app.ts` | Parse l'en-tete `Cookie` et expose `req.cookies`. Necessaire pour lire `auth_token`. |
| `express.json` | Express 5 builtin | Parse le body JSON et expose `req.body`. |
| `rateLimiter(max, windowMs)` | `src/middlewares/rate-limiter.ts` | Limite en memoire par cle `ip:method:baseUrl:path`. Renvoie 429 avec en-tetes `RateLimit-*` et `Retry-After`. Utilise sur auth (`auth.routes.ts`) et contact (`contact.router.ts`). |
| `requireAuth` | `src/middlewares/require-auth.ts` | Lit le cookie de session, verifie le JWT (`jwt.verify`), recharge l'utilisateur via `authUserRepository.findById` et verifie qu'il est `active`. Echec -> `unauthorized(...)` -> 401. |
| `optionalAuth` | `src/middlewares/require-auth.ts` | Variante non bloquante : peuple `req.auth` si le token est valide, sinon laisse passer sans erreur. Utile pour les routes publiques qui personnalisent leur reponse pour les membres connectes. |
| `requireAdmin` | `src/middlewares/require-admin.ts` | Verifie `req.auth?.roleId === 1`. Echec -> `forbidden(...)` -> 403. Doit etre monte apres `requireAuth`. |
| `asyncHandler` | `src/api/http/async-handler.ts` | Wrap un handler async pour propager les rejets vers `next(err)`, donc vers `errorHandler`. Sans lui, un `throw` async serait silencieusement avale par Express. |
| `notFound` | `src/middlewares/not-found.ts` | Catch-all monte en dernier avant `errorHandler`. Cree une erreur 404 `ROUTE_NOT_FOUND` et passe la main. |
| `errorHandler` | `src/middlewares/error-handler.ts` | Middleware d'erreur final. Lit `statusCode` (defaut 500), log les 5xx, renvoie `{ error: { message, code } }`. |
