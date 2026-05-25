---
project_name: recipe-shelter-backend
generated_for: AI agents context (LLM-optimized)
generated_by: GPC-equivalent manual scan (Claude)
date: 2026-05-25
source_repo: C:\DEV\ARTHUR\RECETTES\backend
sections_completed:
  - technology_stack
  - language_rules
  - framework_rules
  - testing_rules
  - quality_rules
  - workflow_rules
  - dont_miss_rules
---

# Project Context — Recipe Shelter Backend

_Critical rules and patterns AI agents must follow when implementing code in this repo. Focus on unobvious details that LLMs would otherwise miss._

---

## Technology Stack & Versions

**Runtime**
- Node.js LTS (no engine pin in `package.json`)
- TypeScript `5.9.3` — `target: ES2022`, `module: NodeNext`, `strict: true`, ESM (`"type": "module"`)
- Build: `tsc` → `dist/`. Dev: `tsx watch src/server.ts`. Tests: `node --import tsx --test`

**HTTP & web**
- Express `^5.2.1` (Express **5**, not 4 — handler error-propagation differs)
- `cookie-parser ^1.4.7`, `cors ^2.8.6` (`credentials: true`, explicit origins required)

**Persistence**
- MySQL 8.x via `mysql2 ^3.18.2` (uses `pool` exported from `src/db/pool.js`)
- Collation `utf8mb4_0900_ai_ci` (MySQL 8 only — MariaDB breaks)
- SQL seed/reset scripts in `database/reset.sql` and `database/reset_demo.sql`

**Auth**
- `jsonwebtoken ^9.0.3` — JWT in HttpOnly cookie `rs_session`, **never** returned in JSON body
- `bcrypt ^6.0.0` — default cost `12` (configurable via `BCRYPT_COST`)
- SameSite `lax` by default; switching to `none` requires adding CSRF protection

**Mail**
- `nodemailer ^8.0.7` via `SmtpMailService` — used for email validation, password reset, contact form

**Tooling**
- `eslint ^9.39.4` + `typescript-eslint ^8.57.1` + `eslint-plugin-import` (flat config in `eslint.config.js`)
- `husky ^9.1.7` + `lint-staged ^16.4.0` (auto `eslint --fix` on staged `src/**/*.ts`)

**Absent on purpose** — no ORM, no DI framework, no test runner package (uses native `node:test`), no scaffolding generator. This is the **"from scratch"** point Arthur defends in front of the jury — do not propose Prisma/TypeORM/NestJS as "improvements" without flagging it as a major architectural change.

---

## Critical Implementation Rules

### Language-Specific Rules (TypeScript ESM)

- **Always use `.js` extensions in import paths**, even for `.ts` source files (NodeNext module resolution requires it). Ex: `import { foo } from './bar.js';` when the file is `bar.ts`.
- **Type-only imports must use the `type` keyword inline** — ESLint rule `@typescript-eslint/consistent-type-imports` with `fixStyle: 'inline-type-imports'`. Ex: `import { type NextFunction, type Request } from 'express';`
- `strict: true` is on — no implicit any, no implicit return undefined. Don't silence with `// @ts-ignore`; fix the type.
- Unused vars: prefix with `_` to silence (`argsIgnorePattern: '^_'`).
- ESM only — no `require()`, no `__dirname`/`__filename` (use `import.meta.url`).
- Import order is **enforced** by `import/order`: `builtin` → `external` → `internal` → `parent/sibling/index` → `type`, with blank lines between groups and alphabetical case-insensitive sort.

### Framework-Specific Rules (Express 5)

- **All wiring lives in `src/app.ts`**. Repositories, services, controllers, routers are instantiated by hand (manual DI). When adding a new feature, follow the established order in `createApp()`:
  1. Instantiate the repository: `new XxxRepositoryMysql(pool)`
  2. Instantiate the service: `new XxxService(repo, ...deps)`
  3. Instantiate the controller via factory: `createXxxController(service)`
  4. Mount the router on `/api/v1/<resource>`
- **Routes are created via factory functions** (`createXxxRouter(controller)`) — not exported as module-level `Router` instances. Same for controllers (`createXxxController(service)`). Preserve this pattern for testability — services can be swapped without monkey-patching.
- **Repository pattern is interface + Mysql implementation**: e.g. `RecipeRepository` interface + `RecipeRepositoryMysql` concrete class. New persistence code must follow the split.
- **API prefix is `/api/v1`**. All routes go under it. Versioning is in the URL, not in headers.
- **Auth flow**: middleware `require-auth.ts` reads the `rs_session` cookie, decodes JWT, and loads the user via `configureAuthUserRepository(userRepository)` (called once in `app.ts`). For admin routes, chain `require-admin.ts` after.
- **Error handling**: throw errors that match the `AppError` shape (`{ statusCode, message, code }`); the global `errorHandler` middleware (`src/middlewares/error-handler.ts`) serializes them into `{ error: { message, code } }`. Status ≥ 500 are logged via `utils/logger`. Don't `res.json()` errors directly.
- **404 handler** (`not-found.ts`) is mounted last, **before** the error handler. Order matters.
- **CORS**: `CORS_ALLOWED_ORIGINS` is parsed at boot; `*` is **rejected** because `credentials: true`. Add origins to the env var, don't bypass.

### Testing Rules

- **Test runner is `node:test` (native)** — no Jest, no Vitest, no Mocha. Use `import { test, describe } from 'node:test'` and `import assert from 'node:assert/strict'`.
- Run with `npm run test`. Test files live under `tests/` mirroring `src/` layout (`tests/services/`, `tests/repositories/`, `tests/middlewares/`, `tests/api/`, `tests/utils/`).
- Test files end in `.test.ts`. Use `tsx` loader (`--import tsx`).
- `tests/http/` contains `.http` files for manual REST testing — **not** automated tests.
- Repositories are tested against patterns; services are tested by injecting mock repositories (the interface/impl split makes this trivial).
- Don't add a mocking library — hand-roll plain objects matching the repository interface.

### Code Quality & Style Rules

- ESLint is the source of truth. Run `npm run lint:fix` before committing (husky hook will auto-fix anyway via lint-staged).
- File naming: **kebab-case** (`recipe-slug.service.ts`, `admin.users.repository.mysql.ts`).
- Class naming: **PascalCase** (`RecipeService`, `RecipeRepositoryMysql`).
- Factory function naming: **camelCase prefixed with `create`** (`createAuthController`, `createRecipesRouter`).
- Folder organization mirrors domain: `api/<feature>/`, `services/<feature>/`, `repositories/<feature>/`. Sub-features get nested folders (`api/admin/`, `services/auth/`).

### Development Workflow Rules

- Branch naming: `<type>/<short-name>` per `documentation/GIT_CONVENTION.md` in the documentation repo (e.g. `docs/uml-diagrams`, `feat/new-endpoint`).
- Commit messages: **Conventional Commits** (`feat:`, `fix:`, `docs:`, `chore:`, etc.).
- Pre-commit hook runs ESLint via lint-staged on staged `src/**/*.ts`. Don't `--no-verify`.
- No CI/CD configured yet — no `.github/workflows/`. Deploy is manual (see [`_draft_deployment/02-plan-railway.md`](../../deployment/02-plan-railway.md)).

### Critical Don't-Miss Rules

- 🚨 **Never return the JWT in the JSON body** — only set the `rs_session` HttpOnly cookie. The cookie name is configurable via `AUTH_SESSION_COOKIE_NAME`; reference `env.auth.sessionCookieName`, don't hardcode `'rs_session'`.
- 🚨 **Never use `*` in `CORS_ALLOWED_ORIGINS`** — the boot check throws. Explicit origins only.
- 🚨 **Slug generation is two-phase** (insert with temporary slug → resolve final slug from id) — see `RecipeSlugService`. Don't shortcut with a single-phase slug, it breaks uniqueness under concurrency. See [`_draft_adr/adr-005`](../../soutenance/backend/adr/adr-005-slug-en-deux-phases.md).
- 🚨 **Moderation is soft-delete + log** — never `DELETE FROM recipes` for moderation actions. See [`_draft_adr/adr-004`](../../soutenance/backend/adr/adr-004-soft-delete-et-log-moderation.md).
- 🚨 **`.js` extension in imports** — forgetting it gives a runtime `ERR_MODULE_NOT_FOUND` that's invisible at compile time.
- 🚨 **Don't add an ORM** — the from-scratch SQL+mysql2 stack is the cert defense story. See [`_draft_adr/00-narration-from-scratch.md`](../../soutenance/backend/adr/00-narration-from-scratch.md).
- ⚠️ Rate limiter middleware exists (`middlewares/rate-limiter.ts`) — apply on auth routes (`AUTH_RATE_LIMIT_*` env vars). Don't roll a new one.
- ⚠️ Password policy lives in `services/auth/password-policy.ts` — don't duplicate validation logic in controllers.
- ⚠️ Mail sending failures throw `MAIL_SEND_FAILED` / `CONTACT_SEND_FAILED` — auth routes that send email (register, resend-validation, forgot-password) will fail without valid SMTP config. Skip those routes when developing without SMTP, or wire a dev SMTP catcher.

---

## Source Tree (high level)

```
backend/
├── database/                      # SQL seed/reset scripts (MySQL 8 only)
├── src/
│   ├── server.ts                  # bootstrap, listens on PORT
│   ├── app.ts                     # createApp() — manual DI for all features
│   ├── api/                       # controllers + routers, one folder per feature
│   │   ├── admin/                 # admin.recipes, admin.users, admin.comments
│   │   ├── auth/                  # login, register, logout, validate, reset
│   │   ├── category/ comments/ contact/ equipments/ favorites/
│   │   ├── health/ ingredients/ recipes/ tag/ users/
│   ├── db/
│   │   └── pool.ts                # mysql2 pool (singleton)
│   ├── middlewares/
│   │   ├── error-handler.ts       # global JSON error serializer
│   │   ├── not-found.ts           # 404 — mount before error-handler
│   │   ├── rate-limiter.ts        # apply on auth routes
│   │   ├── require-auth.ts        # reads rs_session cookie, loads user
│   │   └── require-admin.ts       # chains after require-auth
│   ├── repositories/              # interface + xxx.repository.mysql.ts
│   │   └── (one folder per feature, mirrors api/)
│   ├── services/                  # business logic, accepts repos via constructor
│   │   ├── auth/                  # auth, email-validation, password-policy, password-reset
│   │   ├── mail/                  # SmtpMailService
│   │   ├── recipes/               # RecipeService + RecipeSlugService
│   │   └── (one folder per feature)
│   ├── types/                     # shared TS types
│   └── utils/
│       ├── env.ts                 # env loader + typed accessors
│       ├── errors.ts              # AppError helpers
│       ├── logger.ts              # logger wrapper
│       ├── pagination.ts string.ts array.ts
│       ├── security/              # crypto, hashing helpers
│       └── session-cookie.ts      # cookie options builder
└── tests/                         # mirrors src/ — uses node:test runner
    ├── api/ middlewares/ repositories/ services/ utils/
    ├── http/                      # manual .http requests (not automated)
    └── tsconfig.json              # test-specific TS config
```

---

## How to use this file

- Drop into a fresh Claude session (or any LLM) before asking it to write/modify backend code.
- Update **only** when stack, conventions, or architectural rules change — not for every commit.
- This file is **not** for jury defense (see `_draft_adr/` and `documentation/soutenance/` for that).

---

## Related context

- [[project-recipe-shelter]] — overall project memory
- [[project-certification-brief]] — RNCP context
- [`_draft_adr/`](../../soutenance/backend/adr/README.md) — architecture decision records (jury defense)
- [UML diagrams](../../soutenance/backend/uml/README.md) — UML diagrams
- [`_draft_deployment/`](../../deployment/README.md) — deployment plan (Railway)
- `CAHIER_DES_CHARGES.md` — cert spec at workspace root
