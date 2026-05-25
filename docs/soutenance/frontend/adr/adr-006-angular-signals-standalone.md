# ADR-006 — Angular Signals + composants standalone

## Statut

Accepté · 2026-05-25

## Contexte

Recipe Shelter est construit sur Angular 21
(`frontend/package.json:39`). Le projet doit gérer un état applicatif
relativement classique : l'utilisateur authentifié (lu par la quasi-totalité
des pages), des listes paginées de recettes, des filtres de recherche, des
formulaires longs (création/édition de recette), un panneau d'admin avec
plusieurs tableaux interactifs. Aucun besoin d'un store global complexe
(pas d'undo/redo, pas de time-travel, pas de synchronisation multi-onglet).

Angular a profondément évolué depuis la version 14. Trois grandes options
coexistaient au démarrage :

1. **L'historique** : NgModules + services basés sur `BehaviorSubject`,
   consommation via le pipe `async` dans les templates. C'est encore le
   modèle dominant dans les tutoriels d'avant 2024 et dans la plupart
   des exemples Stack Overflow.
2. **Le moderne** : composants `standalone` (GA Angular 15-16),
   bootstrap via `bootstrapApplication` (pas d'`AppModule`), état local
   via `signal()` / `computed()` / `effect()` (stable depuis Angular 17),
   services-as-store qui exposent des signals.
3. **Le formel** : NgRx (ou Akita, Elf), avec actions, reducers,
   sélecteurs et effects. Approche Redux complète.

Le contexte pédagogique compte : ce projet est défendu devant un jury
RNCP. Le jury peut très bien connaître surtout le legacy Angular. Le
choix doit donc être à la fois techniquement justifié *et* défendable
sans tomber dans le « parce que c'est nouveau ».

## Décision

Adopter 100 % la pile moderne :

- **Zéro `NgModule`** dans tout le projet. Tous les composants sont
  `standalone: true` (ou implicitement standalone, qui est le défaut
  depuis Angular 19). Bootstrap via `bootstrapApplication(App, appConfig)`
  (`frontend/src/main.ts:5`).
- **Signals pour tout l'état UI local** : chaque composant déclare ses
  états en `signal<T>()` ou `computed()`. Pas de `BehaviorSubject`
  exposé aux templates.
- **Pattern service-as-store** pour l'état partagé : `SessionService`
  est la seule source de vérité pour l'utilisateur authentifié. Il
  expose des `computed()` en lecture (`user`, `isAuthenticated`,
  `isAdmin`) et garde le signal mutable privé
  (`frontend/src/app/core/services/session.service.ts:16`).
- **RxJS conservé pour le HTTP uniquement** : `HttpClient` renvoie des
  `Observable`, on les bridge vers des signals via
  `takeUntilDestroyed(destroyRef)` + `.subscribe(value => signal.set(value))`.

Pas de NgRx, pas d'Akita, pas de NgRx Signal Store pour l'instant.

## Alternatives considérées

### NgModules + BehaviorSubject + pipe async

- **Pour** : familier pour le jury, écosystème massif, tutoriels
  abondants.
- **Contre** : verbeux dans les templates (`user$ | async`), nécessite
  une gestion explicite des souscriptions, lifecycle plus difficile à
  raisonner. Direction officielle Angular : NgModules ne sont plus
  recommandés depuis la v17, et la doc 21 documente d'abord standalone.
- **Risque** : produire du code qui paraîtra daté dans deux ans, et
  qui se mariera mal avec les évolutions à venir (zoneless, signal
  inputs).

### NgRx (ou un store Redux-like)

- **Pour** : structure formelle, devtools, traçabilité de chaque
  mutation, scalable.
- **Contre** : boilerplate massif (actions, reducers, selectors,
  effects, feature modules) pour un projet d'environ 50 routes. Les
  besoins métier ne justifient pas ce coût : pas de undo, pas de cache
  cross-composant complexe, pas de synchronisation multi-onglet.
- **Risque** : passer plus de temps à câbler le store qu'à écrire la
  feature.

### Composants standalone mais Observables partout

- **Pour** : transition douce depuis le legacy, RxJS reste l'outil
  central, on évite de mixer deux modèles.
- **Contre** : RxJS est puissant pour les *flux* (HTTP, websockets,
  événements DOM) mais lourd pour de l'*état* (souscriptions à gérer,
  pipe async dans chaque template, valeurs initiales à fournir). Les
  signals sont conçus pour ce cas-là et s'intègrent au cycle de
  détection de changement d'Angular.
- **Risque** : du code plus verbeux qu'il n'a besoin de l'être, et
  contre-courant de la direction officielle du framework.

## Conséquences

### Positives

- **Code de page lisible**. Dans le template, `user()` au lieu de
  `user$ | async`, `isLoading()` au lieu de `isLoading$ | async`. Le
  lecteur voit immédiatement qu'il s'agit d'une lecture, sans avoir à
  raisonner sur un flux.
- **Source de vérité explicite** pour l'auth. `SessionService` est un
  petit fichier (~50 lignes) qui dit tout : un signal privé `_user`,
  trois `computed()` publics, des setters typés
  (`frontend/src/app/core/services/session.service.ts:16-49`). Le jury
  peut le lire en trente secondes et comprendre comment l'auth circule
  dans l'application.
- **Moins de fuites mémoire**. Les signals n'ont pas besoin d'être
  désabonnés. Quand un `Observable` doit être souscrit (typiquement un
  appel HTTP), on utilise systématiquement
  `takeUntilDestroyed(this.destroyRef)` pour le couper au démontage du
  composant (`frontend/src/app/pages/home/home.ts:83`,
  `frontend/src/app/pages/search/search.ts:191`).
- **Composants à l'API moderne** : les composants enfants utilisent
  `input.required<T>()` et `output<T>()` plutôt que les anciens
  décorateurs `@Input()` / `@Output()`, plus type-safe et plus
  ergonomiques
  (`frontend/src/app/pages/admin/comments/admin-comments-list/admin-comments-list.ts:22-26`).
- **`effect()` pour les synchronisations transverses**. Dans la page
  recherche, trois `effect()` désactivent/activent automatiquement des
  contrôles de formulaire en fonction du chargement des données de
  référence, sans `subscribe` ni `combineLatest`
  (`frontend/src/app/pages/search/search.ts:71-94`).
- **Prêt pour zoneless**. Signals + standalone est le combo recommandé
  pour migrer vers la détection de changement sans `zone.js` quand
  elle sera GA partout.

### Négatives

- **Doc Angular en transition**. La doc officielle elle-même mélange
  encore exemples Signals et exemples Observables. Les recettes Stack
  Overflow d'avant 2024 sont la plupart du temps obsolètes pour notre
  pile, ce qui a obligé Arthur à privilégier la doc officielle et le
  code généré par `ng generate`.
- **Deux modèles à comprendre**. On garde RxJS pour HTTP (parce que
  `HttpClient` retourne des `Observable`) et on utilise Signals pour
  l'état. Le pattern de pont est récurrent et explicite :
  `service.call().pipe(takeUntilDestroyed(this.destroyRef)).subscribe(value => mySignal.set(value))`.
  Ce n'est pas idéal d'avoir deux abstractions, mais c'est la
  situation actuelle de l'écosystème.
- **Pas de devtools dédiés** aux signals (contrairement à NgRx).
  L'inspection en debug se fait à la main dans la console.

### À surveiller

- **Croissance de l'application**. Si l'état partagé entre pages
  devient plus complexe (ex. cache de recettes consultées, panier
  cross-page), envisager **NgRx Signal Store** qui formalise le
  pattern service-as-store sans imposer le boilerplate Redux.
- **Migration zoneless**. Surveiller la stabilité dans Angular 22+ et
  vérifier que les dépendances tierces (formulaires, router) ne
  reposent plus sur `zone.js`.
- **API signal inputs/outputs**. Vérifier au fil des montées de
  version qu'aucune migration n'est requise (les `@Input` historiques
  restent supportés, mais on a déjà standardisé sur `input()` partout).

## Références code

- `frontend/src/main.ts:5` — bootstrap standalone via
  `bootstrapApplication(App, appConfig)`, aucun `AppModule`.
- `frontend/src/app/app.config.ts:10-26` — configuration applicative
  fournie sous forme de providers (route, HTTP, hydration), pas de
  module.
- `frontend/src/app/core/services/session.service.ts:16-24` — pattern
  service-as-store : signal privé, `computed()` publics.
- `frontend/src/app/pages/home/home.ts:30-37` — état de page entièrement
  en signals (`recentRecipes`, `categoriesLoading`, `favoriteError`…).
- `frontend/src/app/pages/home/home.ts:82-93` — pont RxJS → signal via
  `takeUntilDestroyed(this.destroyRef)`.
- `frontend/src/app/pages/search/search.ts:71-94` — trois `effect()`
  qui synchronisent l'état d'activation des contrôles de formulaire
  avec les signals de chargement.
- `frontend/src/app/pages/recipes/list/recipe-list.ts:30-34` — état de
  liste paginée en signals (`recipes`, `pagination`, `isLoading`).
- `frontend/src/app/pages/admin/comments/admin-comments-list/admin-comments-list.ts:22-26`
  — composant enfant avec API signal `input.required()` / `output()`.
