# ADR-008 — TypeScript strict + Angular strict templates

## Statut

Accepté · 2026-05-25

## Contexte

Recipe Shelter côté frontend est une application Angular 21 d'une taille
non négligeable : environ 50 routes, plusieurs pages d'admin avec
tableaux interactifs, des formulaires longs, un état d'authentification
consommé partout, du SSR sélectif par route (cf. ADR-007). Sur ce
volume, deux risques se matérialisent vite :

- **La dérive silencieuse**. Un champ optionnel devient requis dans un
  DTO, un input de composant change de signature, une route ajoute un
  paramètre — sans typage strict, le bug ne sort qu'à l'exécution, et
  pas toujours sur le chemin que le développeur teste.
- **Le refactor risqué**. Renommer un champ ou changer la forme d'un
  modèle doit se propager partout. Si le compilateur ne lève pas la
  main, c'est l'utilisateur final qui découvre l'erreur.

TypeScript propose plusieurs niveaux de strictness, qui ne se résument
pas au flag `strict: true` :

1. `strict: true` est un *umbrella* qui active huit flags
   (`strictNullChecks`, `strictFunctionTypes`, `noImplicitAny`,
   `strictBindCallApply`, `strictPropertyInitialization`,
   `alwaysStrict`, `useUnknownInCatchVariables`, `noImplicitThis`).
2. Au-delà, TypeScript expose des flags supplémentaires qui ne sont
   *pas* inclus dans `strict` :
   `noPropertyAccessFromIndexSignature`, `noImplicitOverride`,
   `noImplicitReturns`, `noFallthroughCasesInSwitch`,
   `noUncheckedIndexedAccess`, etc.
3. Angular ajoute une troisième couche, **les flags du compilateur
   Angular** (`angularCompilerOptions`), dont le plus important est
   `strictTemplates`. Sans lui, le code TypeScript est strict mais les
   templates HTML restent opaques au type-checker.

La décision porte donc sur trois axes simultanés : flags TypeScript
umbrella, flags TypeScript additionnels, flags Angular compilateur.

## Décision

Activer **le mode strict le plus complet supporté par la stack** :

- **`strict: true`** côté TypeScript, plus quatre flags additionnels
  qui sortent de l'umbrella : `noImplicitOverride`,
  `noPropertyAccessFromIndexSignature`, `noImplicitReturns`,
  `noFallthroughCasesInSwitch`
  (`frontend/tsconfig.json:6-10`).
- **Tous les flags Angular strict activés** :
  `strictInjectionParameters`, `strictInputAccessModifiers`,
  `strictTemplates` (`frontend/tsconfig.json:18-22`). Ces trois flags
  sont ceux que la CLI Angular pose par défaut pour une nouvelle app
  v17+, on les conserve plutôt que de les désactiver pour s'en sortir
  plus vite.
- **`isolatedModules: true`** pour rester compatible avec les builders
  modernes (esbuild, swc) qui compilent fichier-par-fichier
  (`frontend/tsconfig.json:12`).
- **ESLint avec presets stricts**. La config étend
  `tseslint.configs.recommended` + `tseslint.configs.stylistic` +
  `angular.configs.tsRecommended` (`frontend/eslint.config.mjs:9-13`).
  Le preset `recommended` de typescript-eslint enforce déjà
  `@typescript-eslint/no-explicit-any`, `no-unused-vars`,
  `ban-ts-comment` (qui interdit `// @ts-ignore` sans description).
  Aucune dérogation locale n'est ajoutée — on prend le preset tel
  quel.

Conséquence pratique : **un commit qui introduit un `any` implicite,
un `// @ts-ignore`, ou un binding template invalide ne passe ni
`ng build` ni `ng lint`**.

## Alternatives considérées

### `strict: false` + activation progressive flag par flag

- **Pour** : onboarding plus doux, dette payée au rythme du
  développeur, pas de blocage initial sur du code legacy.
- **Contre** : sur un projet greenfield, c'est sans objet — il n'y a
  pas de legacy à composer avec. Et en pratique, l'activation
  « progressive » ne se fait jamais : une fois que du code non-strict
  existe, le passage à `strict: true` devient un chantier transverse
  qu'on repousse indéfiniment.
- **Risque** : dette technique permanente, et culture « on activera
  plus tard » qui ne se concrétise pas.

### `strict: true` seul, sans `strictTemplates`

- **Pour** : c'est la baseline TypeScript que beaucoup de projets
  considèrent comme « assez strict ». Code TS type-checké, templates
  laissés à la liberté du développeur.
- **Contre** : c'est précisément la moitié du problème côté Angular.
  Les templates HTML contiennent une part énorme de la logique de
  rendu : conditions `@if`, boucles `@for`, bindings `[value]`,
  événements `(click)`. Sans `strictTemplates`, une faute de frappe
  sur un nom de propriété (`user.nmae` au lieu de `user.name`) compile
  sans broncher et rend une chaîne vide en production.
- **Risque** : avoir l'illusion d'un projet strict alors que la
  surface la plus exposée — le template — n'est pas couverte.

### TypeScript strict mais `// @ts-ignore` toléré pour les cas tordus

- **Pour** : flexibilité ponctuelle quand une lib tierce mal typée
  bloque un appel.
- **Contre** : c'est l'archétype de la fenêtre cassée. Un `@ts-ignore`
  autorisé devient dix, puis devient une politique tacite « si ça ne
  compile pas, ignore ». Le règle ESLint `ban-ts-comment` impose au
  minimum une description (`// @ts-expect-error: raison`), ce qui
  force à motiver l'exception et à laisser une trace en revue.
- **Risque** : érosion garantie de la garantie de typage, sans signal
  visible au reviewer pressé.

### `noUncheckedIndexedAccess` en plus

- **Pour** : flag TypeScript supplémentaire qui rend `arr[i]` de type
  `T | undefined`, ce qui force à narrower avant usage. Plus sûr en
  théorie.
- **Contre** : sur un projet Angular qui consomme beaucoup de tableaux
  paginés et de `Record<string, T>` venant de l'API, ce flag impose
  un pattern de garde (`if (x === undefined)`) sur quasiment chaque
  accès. Le ratio bruit/valeur n'a pas paru justifié pour ce projet.
  Non retenu pour rester ergonomique, à reconsidérer si des bugs
  d'accès indexé apparaissent.

## Conséquences

### Positives

- **Erreurs détectées au build, templates inclus**. Une faute de frappe
  dans un binding (`[value]="user.nmae"`), un `@for` qui itère sur un
  type qui n'est pas itérable, un événement `(click)="hadnleClick()"`
  inexistant : tous remontés par `strictTemplates` au moment du
  `ng build`. C'est la garantie qui change le plus la vie au
  quotidien — sans elle, ces erreurs sortent uniquement quand on
  navigue manuellement sur la page concernée.
- **Refactor sûr**. Renommer un champ d'un modèle (`Recipe.title` →
  `Recipe.name`) propage la rupture partout : services, composants,
  templates. Le compilateur produit la liste exhaustive des sites à
  mettre à jour, on ne progresse plus à coup de grep.
- **DI typée**. `strictInjectionParameters` impose que tous les
  paramètres de constructeur ou d'`inject()` aient un type explicite
  ou un token explicite. Un composant qui appelle
  `inject(SessionService)` rate la compilation si le service n'est pas
  fourni ou si son type a changé — pas de découverte runtime.
- **Inputs verrouillés**. `strictInputAccessModifiers` empêche
  qu'un `@Input()` ou un `input()` soit déclaré dans une visibilité
  incohérente avec son usage template. Combiné avec `input.required<T>()`
  (cf. ADR-006), on a la garantie qu'un composant enfant ne sera pas
  instancié sans ses inputs requis.
- **Documentation Angular alignée**. Toute la documentation Angular
  moderne (a.dev) assume strict + standalone + signals. Coller à ce
  défaut signifie que les exemples qu'Arthur lit dans la doc
  fonctionnent tels quels dans le projet, sans adaptation.

### Négatives

- **Premier jet souvent plus verbeux**. Il faut parfois annoter
  explicitement (`as const`, type assertion sur `JSON.parse`,
  narrowing manuel `if (typeof x === 'string')`). Le surcoût est réel
  les premiers jours, négligeable ensuite.
- **`noPropertyAccessFromIndexSignature` peut surprendre**. Sur un
  objet typé `Record<string, T>` ou avec une index signature, l'accès
  `obj.key` est refusé : il faut écrire `obj['key']`. C'est volontaire
  (la notation pointée suggère un champ connu du type, ce qui est faux
  pour une clé dynamique), mais ça déroute la première fois.
- **Compile time légèrement plus long**. `strictTemplates` parse et
  type-check les templates HTML, ce qui ajoute un coût au build. À
  l'échelle de Recipe Shelter, l'impact reste sous la seconde et n'a
  pas justifié de le désactiver.
- **Onboarding plus exigeant**. Un développeur qui découvre Angular
  va se heurter au type-checker plus tôt que sur un projet relâché.
  C'est aussi un atout : il apprend les bons réflexes dès le départ.

### À surveiller

- **Tentation du `as any`**. ESLint via `tseslint.configs.recommended`
  signale `@typescript-eslint/no-explicit-any` — la règle est active,
  un `as any` se voit en revue. À surveiller : la tentation de
  contourner avec `as unknown as T` (qui n'est pas couverte par la
  règle) reste possible et doit être traquée en code review.
- **Libs tierces mal typées**. Si une lib JS pure (sans types) doit
  être consommée, la bonne pratique est d'écrire un wrapper minimal
  qui expose une API typée, plutôt que d'éparpiller des `any` dans
  le code applicatif. Aucune lib de ce genre n'est utilisée
  aujourd'hui.
- **`noUncheckedIndexedAccess`**. À reconsidérer si des bugs d'accès
  indexé apparaissent (typiquement sur des `Map`-like ou sur
  `params['id']`). L'activer plus tard est faisable, c'est un flag
  additif qui ne casse que les sites qu'il faut effectivement
  corriger.
- **Cohérence entre `tsconfig.app.json` et `tsconfig.spec.json`**. Les
  deux étendent `tsconfig.json`
  (`frontend/tsconfig.app.json:4`, `frontend/tsconfig.spec.json:4`),
  donc les flags strict sont hérités. Toute future divergence
  (relâcher strict dans les tests, par exemple) devrait être un choix
  explicite et documenté — pas une dérive.

## Références code

- `frontend/tsconfig.json:5-17` — bloc `compilerOptions` :
  `strict: true` + `noImplicitOverride`, `noPropertyAccessFromIndexSignature`,
  `noImplicitReturns`, `noFallthroughCasesInSwitch`, plus
  `skipLibCheck`, `isolatedModules`, `importHelpers`, cible ES2022.
- `frontend/tsconfig.json:18-22` — bloc `angularCompilerOptions` :
  `strictInjectionParameters`, `strictInputAccessModifiers`,
  `strictTemplates`, tous à `true`.
- `frontend/tsconfig.app.json:4` — la config app étend la base, donc
  hérite de tous les flags strict.
- `frontend/tsconfig.spec.json:4` — idem pour les tests : les specs
  Vitest tournent avec la même strictness que le code de production.
- `frontend/eslint.config.mjs:9-13` — extensions ESLint :
  `eslint.configs.recommended`, `tseslint.configs.recommended`,
  `tseslint.configs.stylistic`, `angular.configs.tsRecommended`. Le
  preset `tseslint.configs.recommended` inclut
  `@typescript-eslint/no-explicit-any` et
  `@typescript-eslint/ban-ts-comment`.
- `frontend/eslint.config.mjs:28-34` — bloc séparé pour les fichiers
  `.html` : `angular.configs.templateRecommended` +
  `templateAccessibility`. Les templates sont lintés en plus d'être
  type-checkés par `strictTemplates`.
- `frontend/angular.json:91-99` — cible `lint` configurée pour lint
  à la fois les `.ts` et les `.html`, garantissant que la CI rejette
  les deux familles d'erreurs.
