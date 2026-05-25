---
project_name: recipe-shelter-frontend
generated_for: AI agents context (LLM-optimized)
generated_by: GPC-equivalent manual scan (Claude)
date: 2026-05-25
source_repo: C:\DEV\ARTHUR\RECETTES\frontend
sections_completed:
  - technology_stack
  - language_rules
  - framework_rules
  - testing_rules
  - quality_rules
  - workflow_rules
  - dont_miss_rules
---

# Project Context — Recipe Shelter Frontend

_Critical rules and patterns AI agents must follow when implementing code in this repo. Focus on unobvious details that LLMs would otherwise miss._

---

## Technology Stack & Versions

**Runtime & framework**
- Angular `^21.1.0` — **standalone components only** (no `NgModule`), Signals, new control flow (`@if` / `@for` / `@switch`)
- TypeScript `~5.9.2` — `target: ES2022`, `module: preserve`, strict ++ (see Language Rules)
- Node.js (no engine pin) — used for SSR runtime and tooling

**SSR**
- `@angular/ssr ^21.1.1` + Express `^5.1.0` server in `src/server.ts`
- Output mode: `server` (Angular Application Builder, `@angular/build:application`)
- Hydration: `provideClientHydration(withEventReplay())` in `app.config.ts`
- Per-route render mode declared in `src/app/app.routes.server.ts` (`RenderMode.Server` / `Client` / `Prerender`)

**HTTP & state**
- `provideHttpClient(withFetch(), withInterceptors([authInterceptor]))`
- RxJS `~7.8.0` — used for HTTP and async flows; **Signals for state** (no NgRx / Akita / Redux)
- `inject()` everywhere — no constructor injection

**UI**
- Bootstrap `^5.3.8` via **partial SCSS imports** in `src/styles.scss` (only modules actually used: `nav`, `navbar`, `dropdown`, `buttons`, `helpers`, etc.). Do not `@import "bootstrap/scss/bootstrap"` — bundle budget is tight (500 kB initial warning / 1 MB error).
- CSS variables prefixed `--rs-*` (`--rs-primary`, `--rs-accent`, …) defined at `:root` in `styles.scss`.

**Tooling**
- `eslint ^10` flat config (`eslint.config.mjs`) + `typescript-eslint 8.56.1` + `angular-eslint 21.3.1`
- `stylelint ^17` with `stylelint-config-standard` + `stylelint-config-standard-scss`
- `husky ^9.1.7` + `lint-staged` (auto `eslint --fix` on `*.{ts,html}`, `stylelint --fix` on `*.{css,scss}`)
- Prettier embedded in `package.json` (`printWidth: 100`, `singleQuote: true`, `parser: angular` for `*.html`)

**Tests**
- `vitest ^4.0.8` + `jsdom` via `@angular/build:unit-test` (no Karma, no Jasmine, but `describe/it/expect` syntax stays the same). 64 `*.spec.ts` files at the time of writing.
- HTTP tested with `provideHttpClient()` + `provideHttpClientTesting()` + `HttpTestingController` from `@angular/common/http/testing`.

**Absent on purpose** — no state-management lib (Signals do the job), no UI kit beyond Bootstrap, no form lib (Reactive Forms), no Karma, no NgModule, no constructor injection, no eager `loadChildren`. Do not propose these as "improvements" without flagging it as a major architectural change Arthur will have to defend in front of the jury.

---

## Critical Implementation Rules

### Language-Specific Rules (TypeScript strict ++)

- `strict: true` **plus** `noImplicitOverride`, `noPropertyAccessFromIndexSignature`, `noImplicitReturns`, `noFallthroughCasesInSwitch`, `isolatedModules`.
- Angular compiler: `strictTemplates`, `strictInjectionParameters`, `strictInputAccessModifiers` are **all on** — template expressions must type-check, inputs must respect access modifiers.
- `noPropertyAccessFromIndexSignature` is enabled — use bracket notation for index-signature props (`params['page']`, not `params.page`).
- `module: preserve` — keep ESM-style imports (`import { x } from './y'`), bundler resolves. **Do not** add `.js` extensions to imports (the backend does, the frontend does not).
- Don't silence types with `any` or `// @ts-ignore`; fix the typing. ESLint will flag `@typescript-eslint/no-explicit-any`.
- Path aliases are not configured — use relative imports only.

### Framework-Specific Rules (Angular 21)

- **Standalone is mandatory**. Every component, directive, pipe declares its own `imports: [...]`. No `NgModule`, no `declarations`, no `entryComponents`.
- **Selectors**: components use `rs-` prefix in **kebab-case** (`rs-recipe-card`), directives use `rs` prefix in **camelCase** (`rsAutofocus`). Enforced by ESLint (`@angular-eslint/component-selector`, `@angular-eslint/directive-selector`).
- **`inject()` over constructor DI**. All deps are field-level `private readonly x = inject(XService)`. Don't write constructors unless you really need one (rare).
- **Services are `providedIn: 'root'`** unless explicitly scoped. New services go under `src/app/core/services/` with the `Service` suffix.
- **State = Signals**. Use `signal()` for mutable state, `computed()` for derived state. RxJS stays for HTTP and event streams; bridge with `toSignal()` / `takeUntilDestroyed()` when needed. Don't store async state in a `BehaviorSubject` — use a signal.
- **Routing is fully lazy** via `loadComponent` / `loadChildren`. Each domain has its own `*.routes.ts` (`auth.routes.ts`, `recipes.routes.ts`, …) loaded from `app.routes.ts`. Never import a page component eagerly.
- **Layout wraps all routes**: top-level route uses `component: Layout` with children. Page components render inside `<router-outlet>`.
- **SSR per route is declared in `app.routes.server.ts`** — every new route must be added there with the right `RenderMode`:
  - `RenderMode.Server` → public dynamic pages (`recipes/:slug`, `users/:username`)
  - `RenderMode.Client` → authenticated/admin pages (`/profile`, `/me/**`, `/admin/**`, `/auth/validate-email`, `/auth/resend-validation-email`)
  - `RenderMode.Prerender` → static catch-all (`**`)
- **Guard SSR explicitly** in guards/services that touch browser APIs:
  ```ts
  if (isPlatformServer(platformId)) return true;   // guards
  if (!isPlatformBrowser(platformId)) return;      // app.ts ngOnInit
  ```
- **HTTP**: every API call uses `environment.apiBaseUrl` (never hardcode URLs). The `authInterceptor` adds `credentials: 'include'` and clears the session on `401`. New services follow the pattern in `core/services/recipes.service.ts`.
- **Auth flow**: `App` calls `AuthService.me()` once on `ngOnInit` (browser only), feeds `SessionService.setAuthUser(auth)`. Guards (`authGuard`, `adminGuard`, `guestGuard`) re-check via `AuthService.me()` and redirect to `/sign-in?redirectTo=...` with the error code as `status` query param when relevant.
- **Cookies are HttpOnly** — the front **never reads or writes `rs_session`**. Backend sets/clears it; front only reacts to `401` to clear local session state.
- **Templates use new control flow** (`@if`, `@for`, `@switch`). Don't use `*ngIf` / `*ngFor` in new code; ESLint may not flag it but Angular 21 is the standard target.

### Testing Rules

- Test runner is **vitest** via `@angular/build:unit-test` (configured in `angular.json`, no standalone `vitest.config.ts`). Run with `npm test`.
- Spec files live **next to the code** (`auth.service.ts` + `auth.service.spec.ts`). Use the existing `kebab-case.spec.ts` pattern.
- For HTTP: use `provideHttpClient()` + `provideHttpClientTesting()` + `HttpTestingController` — **never** mock `HttpClient` manually.
- Standard pattern:
  ```ts
  TestBed.configureTestingModule({
    providers: [SomeService, provideHttpClient(), provideHttpClientTesting()]
  });
  const req = TestBed.inject(HttpTestingController).expectOne(`${environment.apiBaseUrl}/...`);
  req.flush(payload);
  TestBed.inject(HttpTestingController).verify();
  ```
- For components depending on services with HTTP, inject mocks via `{ provide: XService, useValue: { ... } }`. No mocking lib in the project.
- `tsconfig.spec.json` adds `vitest/globals` types — `describe/it/expect` are globally typed; don't import them.
- No coverage threshold enforced. Don't introduce one without asking Arthur.

### Code Quality & Style Rules

- ESLint is the source of truth. Run `npm run lint` before committing (husky/lint-staged auto-fixes on staged files).
- **File & folder naming: `kebab-case`** (`recipe-card.ts`, `admin-comments.service.ts`).
- **Classes / interfaces / types: `PascalCase`** (`RecipeCard`, `AuthSuccessResponse`).
- **Component class names**: short for layouts/simple pages (`Home`, `Profile`, `Header`); explicit for domain or shared components (`RecipeForm`, `PaginationControls`). Suffix `Component` is **not** systematic — match existing siblings.
- **Component file set**: one folder per component, files named after it: `<name>.ts`, `<name>.html`, `<name>.css`, `<name>.spec.ts`.
- **Folder layout** (enforced by convention — see `RecipeShelterNamingConvention.md`):
  ```
  src/app/
    core/         guards, interceptors, models, services, utils (cross-cutting)
    layouts/      header, footer, layout (global shell)
    pages/        one folder per routed feature
    shared/       reusable components, layouts, styles, validators
  ```
- **Routes (URL)**: kebab-case, business-oriented; user-scoped routes prefixed `/me/...`, admin routes prefixed `/admin/...`, dynamic segments explicit (`:slug`, `:username`, `:id`).
- **CSS classes**: applicative classes prefixed `rs-` (`.rs-btn`, `.rs-main`, `.rs-auth-card`). Component-local styles in the component's `.css`. Shared styles under `src/app/shared/styles/`.
- **No global styles for local needs** — keep selectors scoped.
- **Prettier**: `printWidth: 100`, single quotes, Angular parser for HTML.
- **Comments**: only when the *why* is non-obvious. No JSDoc on obvious methods.

### Development Workflow Rules

- Branch naming: `<type>/<short-name>` per `documentation/GIT_CONVENTION.md` in the documentation repo (`feat/recipe-card-skeleton`, `fix/auth-redirect-loop`).
- Commit messages: **Conventional Commits** (`feat:`, `fix:`, `docs:`, `style:`, `refactor:`, `test:`, `chore:`).
- Pre-commit hook runs ESLint + Stylelint via lint-staged. Don't `--no-verify`.
- No CI/CD configured yet — no `.github/workflows/`. Deploy is manual (see [`_draft_deployment/02-plan-railway.md`](../../deployment/02-plan-railway.md)). Frontend will be served via the SSR Node server on the same Railway service or a separate one — TBD by Arthur.
- `environment.prod.ts` currently points to `https://api.recipe-shelter.fr/api/v1` (domain does not yet exist). Update with the real Railway URL before prod build.

### Critical Don't-Miss Rules

- 🚨 **Never read or write the auth cookie from the front.** `rs_session` is HttpOnly — the only signal is HTTP `401`, which the interceptor catches to call `session.clear()`. Don't try `document.cookie`.
- 🚨 **Never call the API without going through `environment.apiBaseUrl`.** No hardcoded `localhost:3000` or `recipe-shelter.fr`. The interceptor's `isApiRequest()` check relies on this prefix.
- 🚨 **SSR safety**: any code reading `window`, `document`, `localStorage`, `navigator`, `IntersectionObserver`, etc. must be guarded with `isPlatformBrowser(platformId)`. Otherwise the SSR build crashes silently and the page falls back to client-only rendering.
- 🚨 **New route ⇒ update `app.routes.server.ts`** with the right `RenderMode`. Forgetting it means the route inherits `Prerender` from `**` and breaks for dynamic/auth pages.
- 🚨 **Don't import the full Bootstrap SCSS** (`@import "bootstrap/scss/bootstrap";`). The current partial import strategy keeps the initial bundle under 500 kB. Adding a new Bootstrap module = add the matching `@import` in `styles.scss`, nothing more.
- 🚨 **Don't reintroduce `NgModule`** or constructor injection. The "standalone + Signals + inject()" stack is the Bloc 3 defense story for the jury.
- 🚨 **Don't add a state-management lib** (NgRx, Akita, ngxs). Signals + services are deliberate. See [[project-certification-brief]] — Bloc 3 emphasizes mastery of modern Angular primitives.
- ⚠️ **Lazy loading must stay lazy** — never `import { Home } from './pages/home/home'` in `app.routes.ts`. Always `loadComponent: () => import('...').then(m => m.Home)`.
- ⚠️ **Form/API conversions** stay close to the service or component (see `RecipesService.toRecipeBody`) — no separate "mapper" layer.
- ⚠️ **`@angular-eslint` template a11y rules are on** (`templateAccessibility`). Fix the warnings rather than silencing them — the Bloc 1 audit ([`_draft_bloc1_audit/`](../../soutenance/frontend/audit/README.md)) tracks accessibility status for the jury.
- ⚠️ **Bundle budgets**: `initial` 500 kB warn / 1 MB error, `anyComponentStyle` 4 kB warn / 8 kB error. Crossing them fails the production build.
- ⚠️ **`takeUntilDestroyed()`** is the unsubscribe pattern (no manual `Subject` + `takeUntil` boilerplate). It must be called in an injection context (constructor/field initializer) or with an explicit `DestroyRef`.

---

## Source Tree (high level)

```
frontend/
├── angular.json                       # build/serve/test/lint targets
├── eslint.config.mjs                  # flat config, ng-eslint + ts-eslint
├── .stylelintrc.json
├── tsconfig.json / .app.json / .spec.json
├── public/                            # static assets (copied as-is)
└── src/
    ├── index.html                     # SSR entry HTML
    ├── main.ts                        # browser bootstrap
    ├── main.server.ts                 # SSR bootstrap
    ├── server.ts                      # Express server (SSR runtime, listens on PORT|4000)
    ├── styles.scss                    # global styles + Bootstrap partial imports + --rs-* vars
    ├── environments/
    │   ├── environment.ts             # apiBaseUrl: http://localhost:3000/api/v1
    │   └── environment.prod.ts        # apiBaseUrl: https://api.recipe-shelter.fr/api/v1 (domain TBD)
    ├── assets/
    └── app/
        ├── app.ts                     # root component (rs-root) + auth init
        ├── app.config.ts              # browser providers (router, hydration, http, interceptors)
        ├── app.config.server.ts       # server providers (provideServerRendering + routes)
        ├── app.routes.ts              # top-level routes (lazy) wrapped by Layout
        ├── app.routes.server.ts       # RenderMode per path (Server/Client/Prerender)
        ├── core/
        │   ├── guards/                # auth.guard, admin.guard, guest.guard
        │   ├── interceptors/          # auth.interceptor (credentials: include + 401 handling)
        │   ├── models/                # API DTOs (LoginRequest, RecipeSummary, …)
        │   ├── services/              # one *.service.ts per domain (Auth, Recipes, Favorites, Admin*, …)
        │   └── utils/                 # auth-errors, recipe-status, helpers
        ├── layouts/
        │   ├── header/ footer/ layout/
        ├── pages/                     # one folder per routed feature
        │   ├── about/ admin/ auth/ contact/ favorite/
        │   ├── home/ legal/ profile/ recipes/ search/ users/
        └── shared/
            ├── article/ auth-shell/ category-icon/ recipe-card/
            ├── components/            # pagination-controls, …
            ├── layouts/               # recipe-list-shell, …
            ├── styles/components/     # buttons.css + shared CSS
            └── validators/
```

---

## How to use this file

- Drop into a fresh Claude session (or any LLM) before asking it to write/modify frontend code.
- Update **only** when stack, conventions, or architectural rules change — not for every commit.
- This file is **not** for jury defense (see [`_draft_adr/`](../../soutenance/backend/adr/README.md), [`_draft_bloc1_audit/`](../../soutenance/frontend/audit/README.md) and `documentation/soutenance/` for that).

---

## Related context

- [[project-recipe-shelter]] — overall project memory
- [[project-certification-brief]] — RNCP context (Bloc 1 = frontend audit, Bloc 3 = Angular)
- [`backend-context.md`](./backend-context.md) — sibling context for the backend repo
- [`_draft_bloc1_audit/`](../../soutenance/frontend/audit/README.md) — Lighthouse + RGAA findings (file:line) — useful when fixing a11y/perf
- [UML diagrams](../../soutenance/backend/uml/README.md) — UML diagrams (some apply to the frontend client flow)
- [`_draft_deployment/`](../../deployment/README.md) — deployment plan (Railway) — affects `environment.prod.ts`
- `CAHIER_DES_CHARGES.md` — cert spec at workspace root
- `frontend/RecipeShelterNamingConvention.md` — authoritative naming convention reference (kept in sync with this file)
