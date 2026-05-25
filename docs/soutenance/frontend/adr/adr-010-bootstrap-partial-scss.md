# ADR-010 — Bootstrap 5 importé partiellement en SCSS

## Statut

Accepté · 2026-05-25

## Contexte

Recipe Shelter est une application Angular 21 défendue devant un jury
RNCP. Le Bloc 1 demande un travail sérieux sur HTML/CSS et une mise en
forme cohérente, mobile-first, sur une cinquantaine de routes : la home,
la recherche, les fiches recette, l'espace utilisateur `me/**`, le
panneau d'administration `admin/**`, les pages auth. Le branding visé
est sobre (palette bleu nuit `#384688` + accent jaune `#fcc417` +
arrière-plan crème `#f8f5f2`), il n'y a pas de directeur artistique
dans l'équipe et le temps imparti ne permet pas d'écrire un design
system maison.

Le build Angular est configuré avec des budgets de bundle stricts qu'il
ne faut pas faire sauter (`frontend/angular.json:54-58`) :

- `initial` warning à 500 kB, error à 1 MB
- `anyComponentStyle` warning à 4 kB, error à 8 kB

Bootstrap 5 (`bootstrap@^5.3.8`,
`frontend/package.json:45`) est un candidat naturel : grid mature,
utilities riches, composants accessibles, vocabulaire que le jury
connaît. Mais le bundle CSS complet pèse environ 250 kB minifié ; ajouté
au JS Angular, on flirte avec le warning de 500 kB avant même d'avoir
écrit une ligne de feature.

Quatre grandes options se présentaient :

1. **CDN Bootstrap full** : `<link>` vers `bootstrap.min.css`.
2. **Bootstrap full SCSS** : `@import "bootstrap/scss/bootstrap"`.
3. **Bootstrap partiel SCSS** : sélectionner uniquement les modules
   réellement utilisés.
4. **Framework alternatif** : Tailwind CSS, Angular Material.
5. **CSS pur custom** : tout réécrire (grid, reset, utilities).

## Décision

Importer Bootstrap 5 **partiellement** via SCSS dans
`frontend/src/styles.scss` (`frontend/src/styles.scss:1-19`). Quinze
modules sont importés à la main, regroupés en deux blocs :

- **Couches de base obligatoires** : `functions`, `variables`,
  `variables-dark`, `maps`, `mixins`, `utilities`
  (`frontend/src/styles.scss:2-7`). Elles n'émettent quasiment pas de
  CSS mais fournissent les variables Sass et l'API utilities sur
  laquelle reposent tous les autres modules.
- **Modules effectivement consommés** : `root`, `reboot`, `containers`,
  `transitions`, `nav`, `navbar`, `dropdown`, `buttons`, `helpers`,
  `utilities/api` (`frontend/src/styles.scss:10-19`). C'est ce qui
  produit le CSS final livré au navigateur.

Le branding est posé en CSS custom properties sous `:root`, toutes
préfixées `--rs-*` (`frontend/src/styles.scss:24-49`) : couleurs
primaire / accent / cream / dark, focus ring, hauteur de header. Les
variantes RGB de chaque couleur sont exposées séparément
(`--rs-primary-rgb: 56 70 136`) pour pouvoir composer avec `rgb(... /
alpha)` sans manipuler de hex.

Le préfixe des sélecteurs Angular est `rs` (`frontend/angular.json:17`),
ce qui sépare strictement nos composants (`<rs-recipe-card>`,
`.rs-main`) des classes natives Bootstrap (`.container`, `.btn`,
`.navbar`).

Les styles globaux supplémentaires (boutons custom, formulaires d'auth)
sont posés à côté des partials Bootstrap dans
`frontend/src/app/shared/styles/components/`, importés depuis
`styles.scss` (`frontend/src/styles.scss:22`).

## Alternatives considérées

### CDN Bootstrap full

- **Pour** : zéro configuration, mise en place en une ligne.
- **Contre** : sert le bundle complet (~250 kB), aucune sélection, pas
  de tree-shaking, dépendance externe au runtime (CDN down = site cassé
  visuellement). Les variables Sass ne sont pas accessibles, donc pas
  de customization au-delà des overrides CSS bruts.
- **Risque** : performance mobile dégradée, audit Lighthouse pénalisé.

### Bootstrap full SCSS (`@import "bootstrap/scss/bootstrap"`)

- **Pour** : tout est sous la main, aucun risque d'oublier un module.
- **Contre** : ~250 kB de CSS injectés alors qu'on n'utilise concrètement
  ni les carousels, ni les modals, ni les offcanvas, ni les accordions,
  ni les tooltips, ni les badges, ni les progress bars. Le warning de
  budget initial à 500 kB est dépassé presque mécaniquement.
- **Risque** : flouter le message pédagogique « on maîtrise ce qu'on
  embarque ».

### Tailwind CSS

- **Pour** : utility-first ultra-fin, JIT compilation, purge automatique
  du CSS non utilisé.
- **Contre** : vocabulaire CSS atypique (`flex items-center
  justify-between`) sur lequel le jury peut tiquer s'il n'est pas
  familier. Refactor lourd des templates HTML déjà écrits en classes
  Bootstrap. Courbe d'apprentissage à absorber en parallèle de
  l'apprentissage d'Angular.
- **Risque** : passer du temps sur l'outillage CSS plutôt que sur les
  features attendues par le cahier des charges.

### Angular Material

- **Pour** : composants Angular natifs, accessibilité de premier ordre,
  thème SCSS officiel.
- **Contre** : impose Material Design, qui est très daté visuellement
  pour un site de recettes (cartes ombrées, ripple effects, FAB). La
  customization Sass est documentée mais lourde (mixins de thème,
  tokens, palette). Bundle conséquent même en n'important que quelques
  composants.
- **Risque** : un site qui ressemble à toutes les apps Google, à
  l'opposé du branding chaleureux visé.

### CSS pur custom

- **Pour** : zéro dépendance, contrôle absolu.
- **Contre** : réécrire une grille responsive correcte, un reset, des
  utilities (`d-flex`, `gap-*`, `text-*`), une navbar accessible et un
  dropdown ARIA-conforme représente des semaines de travail sans valeur
  pédagogique pour le jury. Le Bloc 1 demande de la maîtrise HTML/CSS,
  pas « réécris ton propre Bootstrap ».
- **Risque** : retarder tout le reste du projet pour reproduire au pire
  ce que Bootstrap fait déjà.

## Conséquences

### Positives

- **Bundle CSS maîtrisé**. En ne tirant que dix modules de production,
  le bundle reste largement sous le budget warning de 500 kB
  (`frontend/angular.json:54-58`).
- **Customization fine sans fork**. Les variables Sass de Bootstrap
  sont accessibles parce qu'on importe `variables` avant tout module
  qui les consomme (`frontend/src/styles.scss:3`). Le branding métier
  est posé séparément en CSS custom properties `--rs-*`
  (`frontend/src/styles.scss:24-49`), ce qui permet de basculer un thème
  sans recompiler le Sass.
- **Grid et utilities matures, gratuits**. `container`, `row`, `col-*`,
  `d-flex`, `gap-*`, `text-center`, `mb-3` sont disponibles partout, et
  c'est précisément le socle dont une app cert RNCP a besoin pour
  produire un rendu propre sans réinventer le wheel.
- **Vocabulaire connu du jury**. Les classes Bootstrap sont identifiables
  immédiatement (`.navbar`, `.dropdown-menu`, `.btn-primary`), ce qui
  facilite la lecture du HTML pendant la soutenance.
- **Pas de collision de classes**. Le préfixe `rs` enforcé par la CLI
  Angular (`frontend/angular.json:17`) et par les règles ESLint garantit
  que nos sélecteurs (`<rs-navbar>`, `.rs-main`) ne percutent jamais un
  sélecteur Bootstrap.
- **Accessibilité**. Le focus visible global est posé une seule fois
  avec la couleur de marque (`frontend/src/styles.scss:51-54`), pas
  laissée au défaut variable des navigateurs.

### Négatives

- **Liste d'imports à maintenir à la main**. Si on commence à utiliser
  un nouveau composant Bootstrap (par exemple `.modal` ou `.accordion`),
  il faut penser à ajouter `@import "bootstrap/scss/modal"` dans
  `styles.scss`. Oubli possible → le composant s'affiche sans styles et
  on perd du temps à diagnostiquer.
- **Migration Bootstrap 6 (à venir)** impactera potentiellement tous
  ces imports : noms de modules réorganisés, suppression de `@import`
  au profit de `@use`. À planifier comme une tâche dédiée plutôt qu'en
  passant.
- **Sass deprecation warnings sur `@import`**. Sass a déprécié `@import`
  au profit de `@use` / `@forward`. Bootstrap 5 n'a pas encore migré son
  arborescence SCSS, ce qui inonde le build de warnings. Mitigation :
  silenciage ciblé dans `frontend/angular.json:39-44` via
  `stylePreprocessorOptions.sass.silenceDeprecations: ["import"]`.
  C'est un choix conscient et limité : on silence uniquement la
  catégorie `import`, pas tous les warnings Sass.
- **Variables `variables-dark` importées mais usage à valider**. On
  tire la palette dark (`frontend/src/styles.scss:4`) parce qu'elle est
  référencée par les maps internes de Bootstrap, mais l'application n'a
  pas encore de mode sombre fonctionnel. Du CSS est embarqué pour rien
  tant que ce mode n'est pas activé.

### À surveiller

- **Croissance du bundle CSS**. Chaque nouveau `@import` Bootstrap ou
  chaque ajout dans `shared/styles/components/` doit être mis en regard
  du budget `initial` (500 kB warning, 1 MB error,
  `frontend/angular.json:54-58`) et du budget `anyComponentStyle`
  (4 kB warning par composant, `frontend/angular.json:60-64`).
- **Migration `@import` → `@use`**. Quand Sass 2.0 supprimera `@import`,
  il faudra réécrire `styles.scss` en `@use "bootstrap/scss/..." with
  (...)`. Le silenciage actuel ne sera plus suffisant. À faire avant
  cette échéance, idéalement quand Bootstrap publiera sa propre
  migration officielle.
- **Mode sombre**. Si on active un thème sombre, vérifier que
  `variables-dark` est exploité correctement et que les variables
  `--rs-*` ont leur pendant `@media (prefers-color-scheme: dark)`.
- **Drift entre branding `--rs-*` et palette Bootstrap**. Les couleurs
  `--rs-*` sont définies en custom properties CSS, indépendamment des
  variables Sass `$primary` de Bootstrap. Si on commence à utiliser
  intensivement `.btn-primary`, il faudra soit redéfinir `$primary`
  avant l'import des variables Bootstrap, soit overrider `.btn-primary`
  pour utiliser `var(--rs-primary)`. Aujourd'hui les boutons custom
  sont posés dans `shared/styles/components/buttons.css` pour éviter
  cette question.

## Références code

- `frontend/src/styles.scss:1-19` — les quinze imports Bootstrap
  sélectionnés (6 couches de base + 10 modules de rendu), avec
  commentaires séparant « modules nécessaires à l'API » et « modules
  réellement utilisés ».
- `frontend/src/styles.scss:22` — point d'entrée des styles globaux
  custom (`shared/styles/components/buttons.css`).
- `frontend/src/styles.scss:24-49` — variables de branding `--rs-*`
  posées sous `:root` (couleurs primary / accent / cream / dark, focus
  ring, hauteur de header), avec variantes RGB pour composition alpha.
- `frontend/src/styles.scss:51-65` — `:focus-visible` global,
  `::placeholder` typé sur `--rs-dark-rgb`, layout container `.rs-main`.
- `frontend/angular.json:17` — `"prefix": "rs"` enforcé pour tous les
  composants Angular, garantit la séparation avec les classes Bootstrap.
- `frontend/angular.json:35-37` — déclaration de `src/styles.scss`
  comme seule feuille de style globale du build.
- `frontend/angular.json:39-44` — `stylePreprocessorOptions.sass.silenceDeprecations:
  ["import"]` pour silencier les warnings Sass liés à `@import`
  Bootstrap.
- `frontend/angular.json:54-58` — budgets de bundle `initial`
  (500 kB warning, 1 MB error) qui dimensionnent toute décision sur
  les imports CSS.
- `frontend/angular.json:60-64` — budget `anyComponentStyle` (4 kB
  warning, 8 kB error) qui empêche un composant de tirer une feuille
  de style disproportionnée.
- `frontend/package.json:45` — version `bootstrap@^5.3.8`.
- `frontend/src/app/shared/styles/components/buttons.css`,
  `frontend/src/app/shared/styles/components/auth-form.css` — styles
  globaux custom déposés à côté des partials Bootstrap.
