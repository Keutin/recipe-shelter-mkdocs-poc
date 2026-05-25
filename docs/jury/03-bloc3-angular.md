# 03 — Bloc 3 : Framework (Angular)

Le Bloc 3 consomme l'API du Bloc 2 et propose une SPA Angular 21 avec SSR. Le jury vérifie ici la maîtrise du framework choisi : structure, état, routing, performance, tests.

---

## Q1. Pourquoi avoir choisi Angular plutôt que React, Vue ou Svelte ?

**Intention jury** — Choix conscient ? Connaissance de l'écosystème ?

**Ossature de réponse**

1. **Framework batteries-included** : Angular fournit routing, formulaires, HTTP client, tests, CLI, build, SSR — pas besoin d'assembler 10 librairies comme avec React.
2. **TypeScript natif** : Angular pousse TS, et la sécurité de typage est cohérente avec le back TS.
3. **Conventions fortes** → moins de bikeshedding sur l'archi, plus de focus sur la fonctionnalité.
4. **Angular 21 moderne** : Signals (réactivité native), composants standalone (pas besoin de NgModule), SSR intégré.
5. **Choix vs React** : React aurait été plus léger mais aurait demandé d'assembler la stack (Redux/Zustand, React Router, Vite, Vitest…). Pour un projet seul à but pédagogique, le confort d'Angular l'emporte.

**Pièges**
- ❌ « Angular c'était dans la formation » → manque d'autonomie.
- ❌ Critiquer React → contre-productif.

---

## Q2. Vous utilisez les Signals. Qu'est-ce que c'est et pourquoi ?

**Intention jury** — Maîtrise des features modernes d'Angular.

**Ossature de réponse**

1. **Définition** : un Signal est une **valeur réactive** ; quand elle change, les composants qui la lisent sont automatiquement re-rendus.
2. **Différence avec RxJS** :
   - RxJS = flux d'événements dans le temps (`Observable`), parfait pour HTTP, debounce, combineLatest…
   - Signals = état courant synchrone, lecture/écriture simple, intégration directe dans les templates.
3. **Avantages** : moins de `subscribe`/`async pipe`, meilleur change detection (Angular peut être plus chirurgical), API plus simple.
4. **Usage dans le projet** : *(à compléter avec Arthur)* — typiquement pour l'état local des composants (liste de recettes, état d'un formulaire, état d'auth utilisateur).
5. **Coexistence avec RxJS** : on garde RxJS pour les appels HTTP via `HttpClient`, mais on peut convertir un Observable en Signal avec `toSignal()`.

**Pièges**
- ❌ Confondre Signals et `Subject` RxJS.
- ❌ Ne pas savoir donner un exemple concret dans le code.

---

## Q3. Composants standalone, qu'est-ce que ça change ?

**Intention jury** — Adoption des features modernes (depuis Angular 14+, défaut 17+).

**Ossature de réponse**

1. **Avant** : chaque composant devait être déclaré dans un `NgModule` (boilerplate, cycles d'imports, lazy-loading verbeux).
2. **Maintenant** : un composant standalone déclare ses propres imports dans le décorateur :
   ```ts
   @Component({
     standalone: true,
     imports: [CommonModule, ReactiveFormsModule, RecipeCardComponent],
     ...
   })
   ```
3. **Avantages** :
   - Pas de NgModule à maintenir.
   - Lazy-loading natif via `loadComponent` dans le router.
   - Tree-shaking plus efficace.
4. **Bundle plus petit** car on importe seulement ce qu'on utilise.

**Ancres** — un composant standalone du projet.

---

## Q4. Qu'est-ce que le SSR et pourquoi l'avez-vous activé ?

**Intention jury** — Comprendre Server-Side Rendering vs Client-Side Rendering.

**Ossature de réponse**

1. **CSR par défaut** : le navigateur reçoit un HTML quasi vide, charge le JS, puis rend l'app. Inconvénients : SEO faible, premier rendu lent, page blanche pendant le chargement.
2. **SSR** : Angular rend l'HTML côté serveur (Node) pour la première requête → le navigateur affiche tout de suite la page rendue, puis le JS « hydrate » l'app pour la rendre interactive.
3. **Avantages** :
   - **SEO** : les bots voient le contenu rendu (essentiel pour un site de recettes qui doit être indexé).
   - **First Contentful Paint** plus rapide → meilleure UX.
   - **Accessibilité** : utile pour les user-agents sans JS.
4. **Coût** : un serveur Node pour rendre, complexité accrue (différence `window`/`document` selon contexte).

**Ancres** — `frontend/angular.json` (config SSR), `frontend/server.ts` ou équivalent.

**Pièges**
- ❌ Confondre SSR et SSG (Static Site Generation = pré-rendu au build).

---

## Q5. Comment gérez-vous l'état de l'application ?

**Intention jury** — State management.

**Ossature de réponse**

1. **Pas de Redux / NgRx** — pour un projet de cette taille, ça aurait été du sur-engineering.
2. **State local** : Signals dans les composants pour leur état UI.
3. **State partagé** : Services injectables avec Signals (ex : `AuthService` expose `currentUser = signal(null)`), partagés en singleton via le DI Angular.
4. **State serveur** : récupéré au besoin via `HttpClient`, pas mis en cache global (à part éventuellement pour la liste de recettes — *à confirmer*).

**Ancres** — un service partagé (`AuthService` ou `RecipeService` du front).

**Pièges**
- ❌ Inventer un store global qui n'existe pas.

---

## Q6. Comment fonctionne votre routing ?

**Intention jury** — Maîtrise du Router.

**Ossature de réponse**

1. **Configuration des routes** : tableau `Routes` qui mappe `path` → composant ou `loadComponent`.
2. **Lazy-loading** : `loadComponent: () => import('./recipe-detail.component').then(m => m.RecipeDetailComponent)` → composant chargé à la demande, bundle initial plus petit.
3. **Routes typiques** :
   - `/` → page d'accueil
   - `/recipes` → liste avec filtres
   - `/recipes/:slug` → détail
   - `/recipes/new` → formulaire création (protégé)
   - `/login`, `/register`, `/profile`
   - `/admin/...` → routes admin (protégées par guard)
4. **Guards** :
   - `authGuard` : redirige vers `/login` si pas connecté.
   - `adminGuard` : 403 si pas admin.

**Ancres** — `app.routes.ts` ou équivalent.

**Pièges**
- ❌ Ne pas savoir où sont définies les routes.

---

## Q7. Comment communiquez-vous avec l'API back ?

**Intention jury** — `HttpClient`, intercepteurs, gestion d'erreur.

**Ossature de réponse**

1. **`HttpClient`** Angular dans des services dédiés (`RecipeService`, `AuthService`, …).
2. **Base URL** dans `environment.ts` (dev / prod différentes).
3. **Cookies HttpOnly** : `withCredentials: true` pour que le navigateur envoie le cookie d'auth automatiquement.
4. **Intercepteur HTTP** :
   - Capter les 401 → rediriger vers `/login`.
   - Capter les 5xx → toast erreur générique.
   - Ajouter éventuellement des headers communs.

**Ancres** — un service HTTP + un intercepteur.

---

## Q8. Comment gérez-vous les formulaires ?

**Intention jury** — Reactive Forms vs Template-driven.

**Ossature de réponse**

1. **Reactive Forms** (`FormGroup`, `FormControl`) — choisis pour les formulaires non-triviaux car :
   - Validations programmatiques claires.
   - Testabilité (on peut piloter le form en TS).
   - Pas de magie du two-way binding.
2. **Validators** : built-in (`required`, `email`, `minLength`) + validators custom si besoin (ex : confirmation mot de passe).
3. **Affichage des erreurs** : `*ngIf="form.get('email').touched && form.get('email').invalid"` + messages contextualisés.
4. **Soumission** : `form.invalid → bloqué` ; `form.valid → envoi HTTP + désactivation du bouton pendant la requête + gestion du retour.

**Ancres** — formulaire d'inscription ou de création de recette.

---

## Q9. Quelle est la stratégie de tests côté Angular ?

**Intention jury** — Cahier des charges : « Jasmine/Karma pour Angular ».

**Ossature de réponse**

1. **Runner** : Jasmine + Karma (configuration par défaut générée par `ng new`).
2. **Tests unitaires** : sur les services (auth, recipe), sur les composants critiques (formulaires, guards).
3. **TestBed** : configure le module de test, fournit mocks pour les services HTTP.
4. **Pas de tests E2E** (limite assumée — j'aurais ajouté Playwright en évolution).

**Pièges**
- ❌ Ne pas savoir lancer les tests (`ng test`).

---

## Q10. Comment optimisez-vous la taille du bundle ?

**Intention jury** — Performance.

**Ossature de réponse**

1. **Build de prod** (`ng build --configuration production`) → minification, tree-shaking, AOT.
2. **Lazy-loading** des routes → chaque page lourde est dans son chunk.
3. **Composants standalone** → tree-shaking plus efficace que les NgModule.
4. **OnPush change detection** sur les composants où c'est sûr → moins de re-renders.
5. **Analyse** : `ng build --stats-json` + `source-map-explorer` pour identifier les gros modules.

**Ancres** — `angular.json` (config de build).

---

## Q11. Le site est-il responsive en Angular ?

**Intention jury** — Lien avec le Bloc 1.

**Ossature de réponse** — Voir aussi [`01-bloc1-frontend.md`](01-bloc1-frontend.md).

- **Bootstrap 5** pour la grille responsive (rows / cols, breakpoints `sm/md/lg/xl/xxl`).
- **Media queries CSS** custom pour les composants spécifiques.
- **Test sur Chrome devtools responsive + BrowserStack**.

---

## Q12. Avez-vous géré l'accessibilité côté Angular ?

**Intention jury** — Lien avec le Bloc 1, mais à valider aussi sur Angular.

**Ossature de réponse**

1. **Sémantique HTML5 native** dans les templates.
2. **`@angular/cdk/a11y`** : `FocusTrap` pour les modales, `LiveAnnouncer` pour les notifs.
3. **Bindings ARIA** : `[attr.aria-label]`, `[attr.aria-invalid]`, `[attr.aria-describedby]` selon le contexte.
4. **Navigation clavier** validée sur les flows critiques.

**Pièges**
- ❌ Ne pas savoir que le CDK Angular a un module accessibilité.

---

## Q13. Comment gérez-vous l'internationalisation ? (si applicable)

**Intention jury** — Pas forcément attendu, mais bonne question optionnelle.

**Ossature de réponse**

- *(Si non implémenté)* « Le site est en français uniquement. Pour de l'i18n, Angular fournit `@angular/localize` qui permet de marquer les textes avec `i18n` dans les templates, d'extraire un fichier XLIFF, et de servir des bundles par langue. C'est une évolution potentielle. »

---

## Q14. Quelle est la structure de votre projet Angular ?

**Intention jury** — Organisation des dossiers.

**Ossature de réponse** — Décrire l'arborescence :

```
src/app/
  core/          → services singleton (auth, http, interceptors)
  shared/        → composants réutilisés, pipes, directives
  features/
    recipes/     → composants liés aux recettes
    auth/        → login, register, forgot-password
    admin/       → routes admin (moderation, users)
  app.routes.ts  → routes globales
  app.config.ts  → providers, intercepteurs
```

*(À ajuster selon la vraie structure du projet d'Arthur.)*

**Pièges**
- ❌ Ne pas connaître l'arborescence par cœur.

---

## Q15. Quels sont les liens entre votre Angular et votre back-end ?

**Intention jury** — Question d'intégration entre les blocs.

**Ossature de réponse**

1. **Contrat API REST** documenté côté back (les endpoints sont définis dans le router Express).
2. **Types TypeScript dupliqués** dans le front : on a une `Recipe` côté back (domain object) et une `Recipe` côté front (DTO de transfert). Ils sont alignés à la main. *(Évolution possible : générer les types depuis OpenAPI.)*
3. **Auth via cookie HttpOnly** : Angular n'a pas accès au token, juste un endpoint `/auth/me` pour savoir si l'utilisateur est connecté.
4. **CORS** activé côté back avec `credentials: true` et l'origine du front en whitelist.

**Pièges**
- ❌ Avoir le JWT en localStorage côté front → contradiction avec ADR-003.
