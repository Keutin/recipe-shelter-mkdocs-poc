# ADR-009 — Vitest comme runner de tests frontend

## Statut

Accepté · 2026-05-25

## Contexte

Recipe Shelter est construit sur Angular 21 (`frontend/package.json:39`).
Le frontend doit être couvert par une suite de tests unitaires qui
valide les services (clients HTTP, parsing des erreurs, session),
les guards de routing, les interceptors et les composants de page.
À ce jour, 64 fichiers `*.spec.ts` sont colocalisés avec le code testé
(un `xxx.ts` est systématiquement accompagné d'un `xxx.spec.ts` dans
le même répertoire).

Historiquement, Angular CLI génère tout nouveau projet avec
**Karma + Jasmine** : un navigateur réel (Chromium headless) est lancé,
les specs sont compilées puis exécutées dans ce navigateur, Jasmine
fournit l'API `describe`/`it`/`expect`. Cette stack a été le standard
de fait pendant dix ans.

Le paysage a changé fin 2024 :

- L'équipe Angular a annoncé la **dépréciation de Karma** et invite
  les projets à migrer vers un autre runner. Les options recommandées
  par la doc officielle sont **Jest** et **Vitest**.
- Angular 20+ introduit un nouveau builder de test, `@angular/build:unit-test`,
  qui s'intègre nativement à Vite et expose Vitest comme moteur
  d'exécution.
- Vitest gagne rapidement du terrain : DX moderne, watch ultra-rapide
  via Vite, API quasi-compatible Jest (`describe`/`it`/`expect`,
  matchers identiques), un seul outil pour le bundling de test et le
  bundling de l'app (déjà sur Vite via `@angular/build`).

Trois options crédibles se posaient pour Recipe Shelter :

1. **Karma + Jasmine** (le défaut historique de la CLI).
2. **Jest** (l'écosystème massif, mature côté Angular via
   `jest-preset-angular`).
3. **Vitest** (la voie moderne, désormais intégrée à `@angular/build`).

Note sur la cohérence avec le backend : côté Node, le projet utilise
le runner natif `node:test` (cf. ADR backend). Ce parti pris « from
scratch » est cohérent avec le cahier des charges du Bloc 2 qui
demande de bâtir le serveur sans framework. **Le frontend est dans une
situation différente** : Angular impose déjà une toolchain lourde
(compilateur, ZoneJS, DI, TestBed, jsdom ou navigateur). Tenter
d'écrire un runner « from scratch » côté front aurait été réinventer
la roue sans valeur pédagogique, et serait incompatible avec
`TestBed`, `ComponentFixture`, `HttpTestingController` qui sont les
primitives standard du framework.

## Décision

Adopter **Vitest 4** comme runner de tests unitaires frontend, via le
builder Angular natif `@angular/build:unit-test`, avec **jsdom** comme
environnement DOM.

- Le target `test` de `angular.json` déclare simplement
  `"builder": "@angular/build:unit-test"`
  (`frontend/angular.json:88-90`). Aucune configuration Karma, aucun
  fichier `karma.conf.js`, aucun `test.ts` de bootstrap : Angular CLI
  pilote Vitest directement.
- `vitest` et `jsdom` figurent en `devDependencies` du `package.json`
  (`frontend/package.json:60`, `frontend/package.json:67`). Aucune
  dépendance Karma / Jasmine n'est installée.
- Les tests sont **colocalisés** : chaque module `xxx.ts` a son
  voisin `xxx.spec.ts` dans le même dossier. 64 specs au total.
- Pattern de test standard Angular : `TestBed.configureTestingModule(...)`
  pour le DI, `HttpTestingController` pour valider les requêtes HTTP,
  `ComponentFixture` pour piloter les composants.
- API de mock : `vi.fn()` (équivalent Vitest de `jest.fn()`) pour les
  doubles, `mockReturnValue`/`mockReset` pour le contrôle.

Aucun navigateur réel n'est lancé pour les tests unitaires. La couche
visuelle et les interactions de bout en bout sont prévues
séparément (Cypress / Playwright), pas dans ce périmètre.

## Alternatives considérées

### Karma + Jasmine (l'historique CLI)

- **Pour** : ce que `ng new` génère par défaut depuis dix ans,
  beaucoup de tutoriels, beaucoup de réponses Stack Overflow.
- **Contre** : déprécié par l'équipe Angular fin 2024, lent (chaque
  run lance un navigateur headless), parallélisation limitée, watch
  mode peu réactif. Continuer sur Karma aurait signifié bâtir sur un
  socle officiellement en fin de vie.
- **Risque** : maintenance descendante. Les plugins Karma ne sont
  plus mis à jour au rythme du reste de l'écosystème, et un
  projet neuf en 2026 qui démarre sur Karma part avec une dette
  technique immédiate.

### Jest (avec `jest-preset-angular`)

- **Pour** : écosystème massif, mature, énormément de matchers et de
  plugins. C'était l'alternative privilégiée pendant la transition
  Karma → autre chose en 2023-2024.
- **Contre** : l'intégration Angular est plus indirecte
  (`jest-preset-angular` + Babel/ts-jest + transformers). La config
  TS est notoirement délicate (mapping ESM/CJS, `transformIgnorePatterns`
  pour les paquets Angular en ESM). Watch mode plus lent que Vitest
  parce que pas adossé à Vite. Et surtout : pas d'intégration native
  dans `@angular/build`.
- **Risque** : passer du temps en config plutôt qu'en tests, et
  diverger du toolchain officiel Angular 21+ qui pousse Vitest.

### `node:test` natif (cohérence avec le backend)

- **Pour** : zéro dépendance supplémentaire, cohérence philosophique
  avec le back (`node:test` partout).
- **Contre** : `node:test` n'a pas d'environnement DOM. Il faudrait
  monter jsdom à la main, écrire les bridges avec `TestBed`, gérer
  ZoneJS, simuler `ComponentFixture`. C'est exactement la roue que
  Vitest + `@angular/build:unit-test` ne réinvente pas. Le parti pris
  « from scratch » du backend ne s'applique pas tel quel au front :
  Angular impose déjà sa propre toolchain de test, et tenter de la
  contourner avec `node:test` aurait coûté des semaines de plomberie
  sans bénéfice pédagogique.
- **Risque** : produire un harness fragile et non-standard qui aurait
  fait fuir tout futur contributeur.

## Conséquences

### Positives

- **Watch mode instantané**. Vitest s'appuie sur Vite : modification
  d'un fichier, seuls les specs concernés ré-exécutent, en
  millisecondes. La boucle TDD est fluide.
- **API Jest-compatible**. `describe`/`it`/`expect`, matchers
  classiques, `vi.fn()` pour les doubles. Un développeur qui connaît
  Jest est productif immédiatement (cf. l'utilisation de `vi.fn()`
  dans `frontend/src/app/layouts/header/header.spec.ts:13-21`).
- **jsdom isolé, pas de navigateur**. Les tests tournent dans un DOM
  simulé en mémoire, parallèles, déterministes. Pas de pop-up
  Chromium, pas de variabilité réseau, pas de port à libérer.
- **Specs colocalisés**. Le `*.spec.ts` est dans le même dossier que
  le `*.ts` qu'il teste. Navigation immédiate dans l'éditeur,
  visibilité directe de la couverture (un fichier sans spec voisin
  saute aux yeux dans l'arborescence).
- **Pattern TestBed standard**. Les tests services utilisent
  `TestBed.configureTestingModule` + `provideHttpClient()` +
  `provideHttpClientTesting()` + `HttpTestingController` pour valider
  méthode, URL, body et flusher la réponse
  (`frontend/src/app/core/services/auth.service.spec.ts:8-21`). Les
  tests de composants utilisent `TestBed.createComponent` +
  `ComponentFixture` avec providers mockés
  (`frontend/src/app/layouts/header/header.spec.ts:31-45`). Pattern
  100 % aligné sur la doc Angular, aucune abstraction propre au
  projet à apprendre.
- **Un seul bundler**. Vite assemble déjà l'app en dev (via
  `@angular/build:dev-server`) et le SSR. Avec Vitest, c'est le même
  outil qui assemble les tests. Cohérence totale de la toolchain.
- **Intégration officielle**. `@angular/build:unit-test` est le
  builder de test officiel à partir d'Angular 20. Pas de
  configuration custom : Angular CLI sait quoi faire.

### Négatives

- **Écosystème Vitest pour Angular plus récent que Jest**. Le couple
  Vitest + Angular est en train de se stabiliser. Quelques recettes
  exotiques (matchers spécialisés, plugins de coverage très fins) ont
  davantage d'antériorité sur Jest. Aucun blocage rencontré jusqu'ici.
- **Pas de navigateur réel**. jsdom couvre 95 % des cas mais ne
  reproduit pas certaines API natives (`ResizeObserver` précis,
  `IntersectionObserver`, layout réel, animations CSS, capture
  d'écran). Pour ces cas, l'option future est Cypress ou Playwright
  en tests E2E, hors périmètre de la suite unitaire.
- **Configuration moins « magique » que Karma**. Là où Karma piloté
  par Angular CLI marchait sans intervention, Vitest peut nécessiter
  un `vitest.config.ts` si on veut personnaliser l'environnement, le
  coverage ou des transformers spécifiques. Aujourd'hui le projet
  s'en passe : le builder `@angular/build:unit-test` fournit la
  configuration par défaut suffisante.
- **Vitest 4 est très récent**. La v4 est sortie courant 2025 ; il
  faut suivre les changelogs de près en cas de montée mineure
  pour ne pas se laisser surprendre par un changement d'API
  (notamment côté `vi.mock`).

### À surveiller

- **Coverage**. Aujourd'hui non activé en CI. À introduire via
  `vitest --coverage` (provider v8 par défaut, sinon istanbul), avec
  seuil minimal défini dans `vitest.config.ts` quand l'outil sera
  jugé utile pour le pilotage.
- **Performance à l'échelle**. 64 specs aujourd'hui, tout est
  instantané. À 500+ specs, mesurer le temps total, et envisager le
  mode `--shard` ou `--no-isolate` si nécessaire.
- **Montées de version Vitest**. La v4 est récente, les majeures
  peuvent introduire des breaking changes (ex. signature de `vi.mock`,
  default de `globals`). Lire les release notes avant chaque bump.
- **Migration éventuelle vers le browser mode**. Vitest propose un
  mode navigateur expérimental (`@vitest/browser`) qui pourrait
  remplacer jsdom le jour où on aurait besoin de tester des
  comportements purement navigateur sans passer à Cypress. À évaluer
  uniquement si un besoin réel apparaît.

## Références code

- `frontend/package.json:60` — `jsdom` en devDependency
  (`"jsdom": "^27.1.0"`), environnement DOM utilisé par Vitest.
- `frontend/package.json:67` — `vitest` en devDependency
  (`"vitest": "^4.0.8"`). Aucune entrée `karma`, `jasmine`,
  `karma-chrome-launcher`, `karma-jasmine` ni
  `@types/jasmine` dans le `package.json`.
- `frontend/package.json:17` — script `"test": "ng test"` : le runner
  est piloté par Angular CLI, pas par un binaire Vitest direct.
- `frontend/angular.json:88-90` — target `test` configuré avec
  `"builder": "@angular/build:unit-test"`, le builder officiel
  Angular qui adosse Vitest. Aucune option custom : on s'appuie
  sur les défauts.
- Absence de `frontend/karma.conf.js`, `frontend/karma.conf.ts`,
  `frontend/src/test.ts` : Vitest n'a pas besoin de bootstrap manuel,
  et Karma n'est pas du tout installé.
- `frontend/src/app/core/services/session.service.spec.ts:1-28` —
  spec service minimaliste : `TestBed.configureTestingModule({})`,
  injection du service, vérification des signals (`user()`,
  `isAuthenticated()`, `isAdmin()`) et des transitions
  (`setAuthUser` → `updateUser` → `clear`).
- `frontend/src/app/core/services/auth.service.spec.ts:1-21` —
  spec service HTTP : `provideHttpClient()` +
  `provideHttpClientTesting()`, appel de la méthode, récupération
  via `HttpTestingController.expectOne()`, assertions sur
  `req.request.method` / `req.request.body`, `req.flush(...)` pour
  simuler la réponse, `verify()` final.
- `frontend/src/app/layouts/header/header.spec.ts:13-45` — spec
  composant : doubles via `vi.fn()` (API Vitest, équivalente à
  `jest.fn()`), injection de mocks `AuthService` et `SessionService`,
  `provideRouter([])` pour le routing, `TestBed.createComponent` +
  `ComponentFixture` pour piloter la vue.
- 64 fichiers `*.spec.ts` au total dans `frontend/src/app/**`,
  systématiquement colocalisés avec le code qu'ils testent.
