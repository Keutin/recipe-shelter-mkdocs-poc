# G6 — Pipeline d'erreur

Ce diagramme detaille le **chemin d'une erreur** depuis un endpoint Express jusqu'a la reponse HTTP serialisee. Il complete G2 (pipeline middleware) en zoomant sur les branches : validation DTO, regle metier, erreur base de donnees, et erreur JS non typee.

L'enjeu est double : (1) montrer comment l'API normalise toutes les sortes d'echecs en un format JSON unique `{ error: { message, code } }`, et (2) signaler honnetement les failles connues — en particulier le risque de fuite d'une `DbError` non mappee en reponse 500.

```mermaid
flowchart TB
    REQ([Requete HTTP /api/v1/...]):::entry

    REQ --> ROUTE[Router Express]
    ROUTE --> MW[Middlewares amont<br/>cookie / cors / json / requireAuth]
    MW --> HANDLER[Controller / api layer]

    HANDLER -->|1 - validation DTO| BR_A{DTO valide ?}
    BR_A -- non --> ERR_HTTP_400[throw badRequest<br/>HttpError 400<br/>code: VALIDATION_ERROR]:::http
    BR_A -- oui --> SERVICE[Service layer]

    SERVICE -->|2 - regle metier| BR_B{Regle OK ?}
    BR_B -- ressource absente --> ERR_HTTP_404[throw notFound<br/>HttpError 404<br/>code: NOT_FOUND_xxx]:::http
    BR_B -- droit refuse --> ERR_HTTP_403[throw forbidden<br/>HttpError 403<br/>code: FORBIDDEN_xxx]:::http
    BR_B -- conflit etat --> ERR_HTTP_409[throw conflict<br/>HttpError 409<br/>code: CONFLICT_xxx]:::http
    BR_B -- ok --> REPO[Repository]

    REPO -->|3 - acces DB| QUERY[db/query.ts<br/>execute SQL via pool]
    QUERY -->|mysql2 throws| CATCH_DB[try/catch dans query]
    CATCH_DB --> TODB[toDbError err, sql]
    TODB --> ERR_DB[throw DbError<br/>code: ER_xxx, sqlState, errno]:::db

    ERR_DB --> SVC_CATCH{Service catche<br/>et mappe ?}
    SVC_CATCH -- oui --> ERR_HTTP_MAPPED[re-throw HttpError<br/>ex: 409 RECIPE_SLUG_TAKEN]:::http
    SVC_CATCH -- non - faille --> LEAK[(DbError remonte<br/>au middleware tel quel)]:::leak

    HANDLER -.->|4 - erreur JS standard| ERR_JS[Error non typee<br/>TypeError, ReferenceError, etc.]:::generic

    ERR_HTTP_400  --> EH
    ERR_HTTP_404  --> EH
    ERR_HTTP_403  --> EH
    ERR_HTTP_409  --> EH
    ERR_HTTP_MAPPED --> EH
    LEAK --> EH
    ERR_JS --> EH

    EH[middlewares/error-handler.ts<br/>errorHandler err, req, res, next]:::handler
    EH --> READ[statusCode = err.statusCode<br/>?? err.status ?? 500]
    READ --> COND{statusCode >= 500 ?}
    COND -- oui --> LOG[logger.error log serveur]
    COND -- non --> SKIP[pas de log]
    LOG --> SERIAL
    SKIP --> SERIAL
    SERIAL[res.status statusCode .json<br/>{ error: { message, code } }]
    SERIAL --> RES([Reponse HTTP au client]):::exit

    classDef entry fill:#e8f4ff,stroke:#1f6feb,color:#0a2540
    classDef exit  fill:#e8ffe8,stroke:#2e7d32,color:#0a2540
    classDef http  fill:#fff5d6,stroke:#b88600,color:#3d2c00
    classDef db    fill:#ffe0cc,stroke:#d04a00,color:#3d1a00
    classDef generic fill:#eeeeee,stroke:#666,color:#222
    classDef leak  fill:#ffd6d6,stroke:#c0392b,color:#5a0000,stroke-dasharray: 5 3
    classDef handler fill:#e6e0ff,stroke:#5e3bdb,color:#1a0f55
```

## Legende des branches

| # | Branche | Origine | Type d'erreur leve | Statut HTTP final |
|---|---------|---------|---------------------|--------------------|
| 1 | Validation DTO | `api/*.dto.ts` (zod / parsing manuel) | `HttpError(400, ..., 'VALIDATION_ERROR')` | 400 |
| 2 | Regle metier | `services/*.service.ts` | `HttpError` 4xx via helpers `notFound` / `forbidden` / `conflict` / `badRequest` | 4xx mappe |
| 3 | Erreur DB | `db/query.ts` catch `mysql2` | `DbError` (puis ideallement re-mappe en `HttpError` par le service) | 4xx si mappe, 500 sinon (faille) |
| 4 | Erreur JS standard | n'importe ou (bug code) | `Error` brute (`TypeError`, ...) | 500 |

Toutes les branches convergent vers le **middleware `errorHandler`** (`backend/src/middlewares/error-handler.ts`), qui est enregistre en dernier dans la chaine Express.

## Types d'erreur

### `HttpError` — la classe canonique exposable

Definie dans `backend/src/utils/errors.ts`.

```ts
class HttpError extends Error {
  constructor(
    public readonly status: number,
    message: string,
    public readonly code?: string
  ) { ... }
  get statusCode(): number { return this.status; }
}
```

Helpers exportes : `badRequest`, `unauthorized`, `forbidden`, `notFound`, `conflict`, `internalError`.

- **Qui le jette** : api layer (validation), service layer (regles metier), middlewares d'auth (`requireAuth`, ...).
- **Format** : `status` numerique + `code` symbolique (ex: `NOT_FOUND_RECIPE`, `FORBIDDEN_NOT_OWNER`). Catalogue complet dans `../errors.md` (~140 codes).
- **Visible client** : oui, c'est le seul type d'erreur dont le `message` et le `code` arrivent intacts au frontend.

### `DbError` — l'erreur interne d'infrastructure

Definie dans `backend/src/db/errors.ts`. Produite par `toDbError(err, sql)` dans `db/query.ts` quand `mysql2` leve.

```ts
class DbError extends Error {
  code?: string;     // ER_DUP_ENTRY, ER_NO_REFERENCED_ROW_2, ...
  sqlState?: string; // 23000, 23503, ...
  errno?: number;
}
```

- **Qui le jette** : uniquement `db/query.ts` (centralise).
- **Convention attendue** : un service qui appelle un repository doit `try/catch` les `DbError` et les **re-emettre en `HttpError`** approprie (ex: `ER_DUP_ENTRY` -> `HttpError(409, ..., 'RECIPE_SLUG_TAKEN')`).
- **Probleme** : `DbError` n'a pas de propriete `statusCode` ni `status`. Si un service oublie le mapping, l'erreur remonte au middleware -> `statusCode = 500` par defaut, le `code` `ER_xxx` n'est pas serialise (pas de `err.code` recopie tel quel en surface utile), et le client recoit `{ error: { message: "...", code: "ER_DUP_ENTRY" } }` avec details SQL dans `message` (concatene par `toDbError`).

### `Error` JS standard — le fallback

Tout `throw new Error(...)`, `TypeError`, `ReferenceError`, promesse rejected non geree (traitee par Express 5 async error propagation) tombe sur la branche 4.

- Pas de `statusCode` -> par defaut `500`.
- Message expose au client tel quel, **ce qui peut leaker des details internes**.

## Le middleware `errorHandler`

Code integral (`backend/src/middlewares/error-handler.ts`) :

```ts
type AppError = {
  status?: number;
  statusCode?: number;
  message?: string;
  code?: string;
};

export function errorHandler(err: AppError, _req, res, _next) {
  const statusCode = err.statusCode ?? err.status ?? 500;
  if (statusCode >= 500) logger.error('[http] Internal error', err);
  res.status(statusCode).json({
    error: {
      message: err.message ?? 'Internal server error',
      code: err.code ?? 'INTERNAL_ERROR'
    }
  });
}
```

### Note de reconciliation `HttpError` vs `AppError`

La Vague 1 (`errors.md`) parle de `HttpError`. Le middleware utilise un type **local** `AppError`. **Ce ne sont pas deux types coexistants** : `AppError` est un simple **type structurel** (duck-typing) qui decrit la **forme** acceptee par le handler (`{ status?, statusCode?, message?, code? }`). Il accepte aussi bien :

- une instance de `HttpError` (qui expose `statusCode` via le getter),
- une `DbError` (qui n'a ni `status` ni `statusCode` -> fallback 500),
- une `Error` JS brute,
- voire un objet errone non-`Error` (anti-pattern, mais le typage le tolere).

C'est un choix de souplesse cote middleware, pas un second hierarchie d'erreurs. **A harmoniser quand meme** : faire de `errorHandler` un consommateur explicite de `HttpError | DbError | Error` rendrait l'intention plus lisible et permettrait un mapping centralise des `DbError` vers HTTP (cf. failles ci-dessous).

## Failles identifiees (rappel Vague 1 — `securite.md`)

1. **DbError fuite en 500 si non mappee par le service.** Exemple : si `recipes.service.ts` ne `try/catch` pas un `INSERT` susceptible de violer `recipes_slug_UK`, le `ER_DUP_ENTRY` remonte tel quel. Le client recoit un 500 avec un message qui contient le SQL et le code mysql. **Correctif recommande** : soit mapping defensif dans chaque service, soit un middleware intermediaire `dbErrorAdapter` avant `errorHandler` qui transforme `DbError` en `HttpError` standard.
2. **Pas de stack trace masking en prod.** Le middleware loggue `err` complet en `statusCode >= 500` (avec stack si presente) mais ne distingue pas dev/prod cote reponse. Le `message` JS brut peut leaker chemins serveur, requetes SQL, noms de variables. **Correctif** : si `NODE_ENV === 'production'` et `statusCode >= 500`, remplacer `err.message` par `'Internal server error'` cote serialisation.
3. **`code` par defaut trop generique.** `'INTERNAL_ERROR'` couvre tous les fallback ; le frontend ne peut pas differencier "bug serveur" et "panne DB" sans inspecter le message. **Correctif** : code dedie `'DB_ERROR'` quand `err instanceof DbError`.

## Cross-references

- `../errors.md` — catalogue des codes d'erreur (~140), conventions de nommage, mapping codes -> statuts
- `../architecture.md` §11 — Gestion d'erreurs (philosophie, conventions par couche)
- `../securite.md` (Vague 1) — exposition d'information sensible via les messages d'erreur
- `g2-pipeline-middleware.md` — chaine Express complete, ordre des middlewares
