# Matrice de conformité — Recipe Shelter ⇄ Cahier des charges

> Une ligne = une exigence atomique extraite du [cahier des charges](../CAHIER_DES_CHARGES.md).
> Évidences pointées vers les chemins réels (repos Arthur ou drafts `_draft_*/`).
> Date du snapshot : 2026-05-25.

## Légende statut

| Code | Signification |
| ---- | ------------- |
| 🟢 | OK — committé et vérifié |
| 🟡 | Ready — draft prêt + walkthrough, en attente de la copie côté Arthur |
| 🟠 | Partial — partiellement traité, manque éléments |
| 🔴 | Manque — rien produit |
| ⚫ | N/A — non applicable au périmètre doc (justifié) |

---

## Bloc 1 — Front-End

| #   | Exigence | Cahier | Statut | Évidence | Gap / Action |
|-----|----------|--------|--------|----------|--------------|
| 1.1 | HTML5 / CSS3 / JS | l.19 | 🟢 | Angular 21 (TS compile → JS), SCSS, templates HTML — couvert par Bloc 3 | Discours soutenance : Bloc 1 réalisé via Angular plutôt que vanilla, à expliquer au jury |
| 1.2 | Framework CSS (Bootstrap) + ARIA | l.20 | 🟢 | [`frontend/src/styles.scss`](../frontend/src/styles.scss) (Bootstrap 5 partial, 16 modules SCSS) + ARIA `role="status"` `aria-live` etc. dans templates | — |
| 1.3 | Outils Lighthouse + BrowserStack | l.21 | 🟠 | Lighthouse : template prêt [`_draft_bloc1_audit/01-lighthouse-template.md`](../soutenance/frontend/audit/01-lighthouse-template.md) ; BrowserStack : non couvert | Exécuter Lighthouse ; pour BrowserStack défense alternative (DevTools mobile + captures `Responsive/`) |
| 1.4 | Affichage recettes (img/titre/desc/ingrédients/étapes) | l.24 | 🟢 | `frontend/src/app/pages/recipes/{list,detail}/`, `recipe-form/`, modèle [`recipe.model.ts`](../frontend/src/app/core/models/recipe.model.ts) | — |
| 1.5 | Formulaire soumission/modification | l.25 | 🟢 | [`pages/recipes/recipe-form/`](../frontend/src/app/pages/recipes/recipe-form/) | — |
| 1.6 | Recherche avancée (ingrédients/type/temps) | l.26 | 🟢 | [`pages/search/`](../frontend/src/app/pages/search/) consommant `/recipes/search` backend | — |
| 1.7 | Commentaires | l.27 | 🟢 | Bloc commentaires dans `pages/recipes/detail/` + `CommentsService` | — |
| 1.8 | Responsivité (media queries) | l.28 | 🟢 | 18 captures responsive (1440×900 / 768×1024 / 375×812) dans [`documentation/soutenance/frontend/Responsive/`](../documentation/soutenance/frontend/Responsive/) + Bootstrap grid | — |
| 1.9 | Accessibilité ARIA + WCAG | l.29 | 🟡 | Grille RGAA pré-remplie [`_draft_bloc1_audit/02-rgaa-grille.md`](../soutenance/frontend/audit/02-rgaa-grille.md) — 45 conformes / 16 partiels / 3 NC / 17 N/A ≈ 70 % | Copier dans `documentation/soutenance/frontend/audit/` ; appliquer P1 (~1h20) du [plan de remédiation](../soutenance/frontend/audit/04-plan-remediation.md) avant soutenance |
| 1.10 | Livrable code source HTML/CSS/JS commenté | l.32 | 🟢 | `frontend/src/` (150 TS, 64 specs) | — |
| 1.11 | Doc conception/responsivité/accessibilité | l.33 | 🟡 | À copier : RGAA grid, audit findings, frontend architecture (1018 l.) [`_draft_frontend_docs/frontend-architecture.md`](../soutenance/frontend/architecture.md), ADRs 006-010 | Exécuter handoffs ; walkthrough frontend-arch dispo dans `_draft_frontend_docs/README.md` côté workspace |
| 1.12 | **Démo en ligne** | l.34 | 🔴 | **AUCUNE** — `nslookup recipe-shelter.fr` → NXDOMAIN. Pas de Dockerfile, fly.toml, render.yaml, .github/workflows/ | **🚨 ÉLIMINATOIRE.** Exécuter [`_draft_deployment/02-plan-railway.md`](../deployment/02-plan-railway.md) (~3h30) ou plan B tunnel ngrok |
| 1.13 | Tests responsivité BrowserStack | l.37 | 🔴 | Pas de compte ni de scripts BrowserStack | Défense : « Tests réalisés via Chrome DevTools mobile/desktop et captures matérielles iPhone/iPad — voir `Responsive/` (3 breakpoints × 9 features) » |
| 1.14 | Tests accessibilité Lighthouse | l.38 | 🟡 | Template [`_draft_bloc1_audit/01-lighthouse-template.md`](../soutenance/frontend/audit/01-lighthouse-template.md) à 5 URLs × mobile+desktop | Arthur exécute `npm run build && http-server` puis DevTools → Lighthouse, capture scores, remplit le template |
| 1.15 | Tests performance | l.39 | 🟠 | Couvert partiellement par Lighthouse (Core Web Vitals dans le template) | Inclure scores LCP/CLS/INP dans la doc audit |

## Bloc 2 — Back-End from scratch

| #   | Exigence | Cahier | Statut | Évidence | Gap / Action |
|-----|----------|--------|--------|----------|--------------|
| 2.1 | Langages Node.js / SQL | l.46 | 🟢 | Node 20+ / TypeScript / mysql2 [`backend/package.json`](../backend/package.json) | — |
| 2.2 | Base MySQL | l.47 | 🟢 | mysql2 driver + schema 16 tables [`backend/database/migrations/1_create_schema.sql`](../backend/database/migrations/1_create_schema.sql) | — |
| 2.3 | Outil de modélisation UML | l.48 | 🟢 | 5 diagrammes Mermaid committed [`documentation/soutenance/backend/uml/`](../documentation/soutenance/backend/uml/) sur branche `docs/uml-diagrams` | Push de la branche par Arthur (commit `1acdf99` non poussé — re-attribution patch déjà documentée) |
| 2.4 | **From scratch — sans frameworks/librairies** | l.53 | 🟠 | Express, bcrypt, jsonwebtoken, cookie-parser, cors, nodemailer, mysql2 présents dans `package.json` | **Argumentation préparée** dans [`_draft_adr/00-narration-from-scratch.md`](../soutenance/backend/adr/00-narration-from-scratch.md) : pas de framework MVC (Nest/AdonisJS), pas d'ORM (Prisma/TypeORM), pas de DI, pas de scaffolding ; Express = couche HTTP minimale, libs utilisées sont des **primitives** OS/crypto/protocole. Le câblage manuel dans [`app.ts`](../backend/src/app.ts) est l'illustration cœur. À défendre frontalement |
| 2.5 | Contrôleurs / modèles / vues | l.54 | 🟢 | `src/api/<domaine>/*.controller.ts`, `services/`, `repositories/` (vues = JSON DTO) | — |
| 2.6 | Programmation Orientée Objet | l.55 | 🟢 | Classes TS, pattern Repository (interface + impl Mysql) [`adr-002`](../soutenance/backend/adr/adr-002-pattern-repository-interface-impl.md) | — |
| 2.7 | Gestion recettes/catégories/commentaires/utilisateurs | l.56 | 🟢 | 11 domaines API : `recipes`, `categories`, `comments`, `users`, `auth`, `admin/{recipes,users,comments}`, etc. | — |
| 2.8 | Comptes utilisateurs + rôles (admin/user) | l.59 | 🟢 | Table `users` + `roles`, `roleId` dans JWT, [`admin.guard.ts`](../frontend/src/app/core/guards/admin.guard.ts) | — |
| 2.9 | Inscription/connexion/profil/récup pwd | l.60 | 🟢 | `auth.routes.ts` : register, login, logout, me, forgot/reset password, validate-email. Tokens opaques SHA-256, TTL 30 min/24 h | — |
| 2.10 | Permissions / rôles | l.61 | 🟢 | `require-auth` middleware + `require-role` + re-check `user.status === 'active'` en base [`require-auth.ts:57-67`](../backend/src/middlewares/require-auth.ts) | — |
| 2.11 | Ajout/modif/suppr recettes | l.64 | 🟢 | CRUD complet + workflow draft → pending → published → rejected/archived | — |
| 2.12 | Catégories / tags | l.65 | 🟢 | Tables `categories`, `tags`, `recipe_tags`, services dédiés | — |
| 2.13 | Affichage par catégories/tags | l.66 | 🟢 | Endpoints `/recipes?categoryId=…&tagId=…` | — |
| 2.14 | Recherche avancée | l.67 | 🟢 | Endpoint `/recipes/search` + index `FULLTEXT` (cf. [G5 ERD](../soutenance/backend/diagrams/g5-erd.md)) | — |
| 2.15 | Commentaires | l.70 | 🟢 | `comments.routes.ts` + `recipes/:id/comments` | — |
| 2.16 | Modération commentaires | l.71 | 🟢 | `admin/comments/{moderated,soft-deleted}` + workflow soft-delete | — |
| 2.17 | Favoris | l.72 | 🟢 | `favorites.routes.ts` | — |
| 2.18 | MySQL gestion données | l.75 | 🟢 | Pool mysql2 [`db/pool.ts`](../backend/src/db/pool.ts) + transactions | — |
| 2.19 | CRUD complet | l.76 | 🟢 | 53 endpoints OpenAPI [`_draft_backend_docs/openapi/openapi.yaml`](../soutenance/backend/openapi/openapi.yaml) | — |
| 2.20 | Optim SQL | l.77 | 🟢 | Index FULLTEXT, pagination LIMIT/OFFSET, projection des colonnes, JOIN ciblés | — |
| 2.21 | Code source backend | l.80 | 🟢 | `backend/src/` (136 TS) | — |
| 2.22 | Schémas fonctionnels + modèles données | l.81 | 🟡 | À copier : [`_draft_backend_docs/data-dictionary.md`](../soutenance/backend/data-dictionary.md) (16 tables) + [G5 ERD Mermaid](../soutenance/backend/diagrams/g5-erd.md). Schémas fonctionnels : UML déjà committed | Suivre walkthrough [`_draft_backend_docs/README.md`](../soutenance/backend/README.md) (branche `docs/backend-soutenance-vague1` + `docs/backend-soutenance-vague2`) |
| 2.23 | Doc tech + UML + description fonctionnalités | l.82 | 🟡 | [`_draft_backend_docs/architecture.md`](../soutenance/backend/architecture.md) (820 l., 12 sections) + [`errors.md`](../soutenance/backend/errors.md) (~140 codes) + [`tests.md`](../soutenance/backend/tests.md) + [`environment.md`](../soutenance/backend/environment.md) + diagrams G1–G8 + ADRs 001-005 | Handoffs en attente — branches `docs/backend-soutenance-vague1/2` + `docs/adr-architecture-decisions` |
| 2.24 | Script SQL création/gestion BDD | l.83 | 🟢 | [`backend/database/migrations/1_create_schema.sql`](../backend/database/migrations/1_create_schema.sql) + [`seed.sql`](../backend/database/seed.sql) + [`seed_demo.sql`](../backend/database/seed_demo.sql) + `reset.sql` | — |
| 2.25 | Tests unitaires | l.86 | 🟢 | 29 tests `node:test` natif (DTOs, services, middlewares, mappers) [`backend/tests/`](../backend/tests/) — détail [`_draft_backend_docs/tests.md`](../soutenance/backend/tests.md) | — |
| 2.26 | Tests d'intégration | l.87 | 🟠 | Tests sont surtout unitaires (avec mocks). 11 fichiers `.http` pour test manuel + 12 collections Postman [`documentation/soutenance/backend/postman/`](../documentation/soutenance/backend/postman/) | Défense : « Tests intégration via Postman collections automatisables + 11 scripts `.http` versionnés » |
| 2.27 | Revues de code par pairs | l.88 | ⚫ | N/A — projet solo, pas de pair | À mentionner : revue par `eslint`/`prettier`/`husky` lint-staged + auto-revue formalisée par ADRs |

## Bloc 3 — Framework (Angular)

| #   | Exigence | Cahier | Statut | Évidence | Gap / Action |
|-----|----------|--------|--------|----------|--------------|
| 3.1 | Framework au choix (Angular) | l.95 | 🟢 | Angular 21.1.0 SSR — [`frontend/angular.json`](../frontend/angular.json) | — |
| 3.2 | Outils dev (Webpack/Babel/TS/Vite) | l.96 | 🟢 | TypeScript 5.9 strict, esbuild (via `@angular/build`), Vitest 4 | — |
| 3.3 | Affichage dynamique recettes via API | l.99 | 🟢 | `RecipesService` + signals + lazy routes | — |
| 3.4 | Gestion état (hooks/services) | l.100 | 🟢 | Pattern service-as-store [`session.service.ts`](../frontend/src/app/core/services/session.service.ts) (signal privé + 3 computed) — cf. [ADR-006](../soutenance/frontend/adr/adr-006-angular-signals-standalone.md) | — |
| 3.5 | Routage | l.101 | 🟢 | `app.routes.ts` (9 groupes lazy) + `app.routes.server.ts` (RenderMode par route) | — |
| 3.6 | Interactivité (AJAX sans rechargement) | l.102 | 🟢 | `HttpClient` + `withFetch()` + interceptor `credentials: 'include'` | — |
| 3.7 | Code source bien documenté/structuré | l.105 | 🟢 | 150 fichiers TS, 5 dossiers fonctionnels (`core/layouts/pages/shared/environments`), conventions documentées [`RecipeShelterNamingConvention.md`](../frontend/RecipeShelterNamingConvention.md) | — |
| 3.8 | Guide utilisateur + doc tech choix techno | l.106 | 🟡 | Guide user prêt [`_draft_user_guide/`](../manuel-utilisateur/README.md) (5 fichiers) ; doc tech prête [`_draft_frontend_docs/frontend-architecture.md`](../soutenance/frontend/architecture.md) (1018 l.) + ADRs 006-010 [`_draft_adr_frontend/`](../soutenance/frontend/adr/README.md) | Handoffs : guide → `documentation/manuel-utilisateur/` ; arch + ADRs → `documentation/soutenance/frontend/{architecture.md,adr/}` (créer README de handoff pour `_draft_frontend_docs/`) |
| 3.9 | **Démo en ligne** | l.107 | 🔴 | **AUCUNE** (cf. 1.12) | **🚨 ÉLIMINATOIRE** — voir 1.12 |
| 3.10 | Tests unitaires (Jasmine/Karma/Jest) | l.110 | 🟢 | **64 specs Vitest** via builder officiel `@angular/build:unit-test` (Angular 20+ natif) — cf. [ADR-009](../soutenance/frontend/adr/adr-009-vitest.md) | Défense : « Vitest = équivalent moderne de Karma/Jasmine, builder officiel Angular » |
| 3.11 | Tests fonctionnels | l.111 | 🟠 | Pas de Playwright/Cypress dans le périmètre frontend. Dossier `e2e/` mentionné mais hors `npm test` | Défense : « Tests fonctionnels via scénario démo manuel [`scenario-demo.md`](../documentation/soutenance/demo/scenario-demo.md) + comptes test [`comptes-test.md`](../documentation/soutenance/demo/comptes-test.md) » |
| 3.12 | Tests performance (bundle/loading) | l.112 | 🟡 | Budgets `angular.json` : 500 kB initial / 4 kB anyComponentStyle ; à confirmer par Lighthouse | Inclure dans audit Bloc 1 (1.14/1.15) |

## Transverse & Modalités RNCP

| #   | Exigence | Cahier | Statut | Évidence | Gap / Action |
|-----|----------|--------|--------|----------|--------------|
| T.1 | Soutenance jury 2 pros ≥3 ans XP | l.120 | ⚫ | N/A doc — action Arthur | — |
| T.2 | Argumentation choix conception/dev/implém | l.120 | 🟡 | Slides Marp [`_draft_slides/`](../slides/README.md) (28 slides, ~20 min) + ADRs (10 au total) + narration from-scratch | Personnaliser slides ([À COMPLÉTER] : date, jurés, scores Lighthouse, URLs) ; entraînement |
| T.3 | Sécurité / fiabilité / efficacité | l.120 | 🟡 | [`_draft_backend_docs/securite.md`](../soutenance/backend/securite.md) (440 l., OWASP Top 10 mapping) | Handoff en attente |
| T.4 | Modifier code en temps réel jury | l.122 | ⚫ | N/A doc — action Arthur | Entraînement : revue ADRs + code à jour |
| T.5 | Force de proposition (innovations) | l.122 | 🟡 | [`_draft_jury/06-ameliorations-innovantes.md`](../jury/06-ameliorations-innovantes.md) (10 questions amélioration sécu/perf/UX/IA) | Reste local (banque entraînement, pas un livrable) |
| T.6 | RGPD (sous-entendu cert RNCP) | — | 🟠 | Securite.md identifie trou : pas de `DELETE /users/me` en code. Mentions légales committed dans frontend `pages/legal/` | Code change Arthur : exposer `DELETE /users/me` ; ou défense « purge admin sur demande utilisateur » |
| T.7 | Stage avec tuteur | l.124 | ⚫ | Hors périmètre projet | — |

---

## Synthèse des comptages

| Statut | Bloc 1 | Bloc 2 | Bloc 3 | Trans. | **Total** |
|--------|-------:|-------:|-------:|-------:|----------:|
| 🟢 OK  | 8      | 22     | 8      | 0      | **38**    |
| 🟡 Ready | 3    | 2      | 2      | 3      | **10**    |
| 🟠 Partial | 2  | 2      | 1      | 1      | **6**     |
| 🔴 Manque | 2   | 0      | 1      | 0      | **3**     |
| ⚫ N/A | 0      | 1      | 0      | 3      | **4**     |
| **Total ligne** | **15** | **27** | **12** | **7** | **61** |

**Taux brut OK** : 38 / 61 = **62 %**
**Taux livrable** (OK + Ready, hors N/A) : 48 / 57 = **84 %** si Arthur exécute tous les handoffs
**Taux soutenable** (OK + Ready + Partial avec défense, hors N/A) : 54 / 57 = **95 %**, modulo le bloc éliminatoire déploiement (3 🔴)

---

## Note d'audit — `_draft_frontend_docs/frontend-architecture.md`

Vérification factuelle de 9 affirmations contre le code Angular réel :

| # | Affirmation | Résultat |
|---|-------------|----------|
| 1 | Services sous `core/services/` (pas `core/auth/`) | ✅ PASS |
| 2 | 3 guards avec `if (isPlatformServer) return true;` | ✅ PASS |
| 3 | Interceptor pose `credentials: 'include'` | ✅ PASS |
| 4 | Builder de test `@angular/build:unit-test` | ✅ PASS |
| 5 | Pas de `vitest.config.ts` | ✅ PASS |
| 6 | Bootstrap importé partiellement | ⚠️ **PARTIAL** — le doc dit « 15 modules » (l.38 et l.712), réel = **16** (`functions, variables, variables-dark, maps, mixins, utilities, root, reboot, containers, transitions, nav, navbar, dropdown, buttons, helpers, utilities/api`). Correction d'1 ligne |
| 7 | Préfixe `rs-` enforcé | ✅ PASS |
| 8 | Chemin recipe list `pages/recipes/list/recipe-list.ts` | ✅ PASS |
| 9 | `auth-form.css` : pas dans `styles.scss` mais importé depuis 6 pages auth | ✅ PASS |

**Verdict** : doc précis à 8/9. Corriger « 15 modules » → « 16 modules » en deux endroits (lignes 38 et 712 de [`soutenance/frontend/architecture.md`](../soutenance/frontend/architecture.md)) — et idéalement [ADR-010](../soutenance/frontend/adr/adr-010-bootstrap-partial-scss.md) si même chiffre cité.
