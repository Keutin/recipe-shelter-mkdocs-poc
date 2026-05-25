# ADR-007 — SSR Angular avec mode de rendu par route

## Statut

Accepté · 2026-05-25

## Contexte

Recipe Shelter expose deux familles de pages très différentes :

- **Des pages publiques** indexables par les moteurs (la fiche d'une
  recette `recipes/:slug`, la page d'un auteur `users/:username`, la
  liste des recettes, la home). Le SEO est un objectif du projet : on
  veut que les recettes apparaissent dans les résultats Google et que
  les aperçus sur les réseaux sociaux affichent le bon titre et la
  bonne image.
- **Des pages privées** sous authentification (espace `me/**`,
  panneau d'admin `admin/**`, validation d'email, profil). Ces pages
  ne doivent pas être indexées et leur contenu dépend de l'utilisateur
  connecté.

Angular 21 (`frontend/package.json:39-44`) fournit
`@angular/ssr` qui permet de choisir un mode de rendu par route
(`Server`, `Client`, `Prerender`) plutôt que de basculer toute
l'application en SSR ou en CSR. C'est une capacité récente
(Angular 17+) qui remplace l'ancien « Angular Universal » full-SSR.

Trois grandes options se présentaient :

1. **CSR pur** (Client-Side Rendering uniquement). Une seule
   `index.html` vide, tout est rendu par le navigateur.
2. **SSR complet** (toutes les routes rendues côté serveur).
3. **SSR sélectif par route** : chaque route choisit son mode.

## Décision

Adopter le **SSR sélectif** via `app.routes.server.ts` avec trois modes :

- **`RenderMode.Server`** pour les routes publiques dynamiques où le
  SEO importe : `recipes/:slug` (la fiche recette est le cœur métier)
  et `users/:username` (page auteur)
  (`frontend/src/app/app.routes.server.ts:25-27`,
  `frontend/src/app/app.routes.server.ts:60-63`).
- **`RenderMode.Client`** pour toutes les routes auth et admin :
  `profile`, `me/recipes/**`, `me/favorites`, `admin/**`,
  `auth/validate-email`, `auth/resend-validation-email`
  (`frontend/src/app/app.routes.server.ts:4-58`).
  Pas d'intérêt SEO, pas de pré-rendu côté serveur sur des données
  qui dépendent du cookie d'auth de l'utilisateur.
- **`RenderMode.Prerender`** pour le catch-all `**`
  (`frontend/src/app/app.routes.server.ts:64-67`). Les pages statiques
  restantes (pages publiques sans paramètre, pages d'information) sont
  générées au build et servies en fichiers plats.

Côté hydratation, on active `provideClientHydration(withEventReplay())`
dans la configuration applicative
(`frontend/src/app/app.config.ts:20`). L'event replay rejoue les
événements (clics, focus) survenus pendant la phase d'hydratation pour
éviter qu'un utilisateur rapide ne perde son interaction.

Le serveur Node minimal utilise Express comme couche statique +
dispatcher SSR (`frontend/src/server.ts`).

## Alternatives considérées

### CSR pur

- **Pour** : simplicité maximale. Pas de serveur Node, pas de
  branches `isPlatformBrowser`, pas de double-rendu. Déploiement
  trivial (n'importe quel CDN ou bucket statique).
- **Contre** : SEO médiocre sur les fiches recette. Googlebot sait
  exécuter du JS mais le rendu différé pénalise l'indexation, et tous
  les crawlers sociaux (Twitter, Facebook, WhatsApp) ne le font pas.
  Or les fiches recette sont précisément le contenu qu'on veut voir
  émerger en partage.
- **Risque** : sacrifier un objectif fonctionnel (visibilité du
  contenu) pour gagner du temps technique.

### SSR complet (toutes les routes en `RenderMode.Server`)

- **Pour** : cohérent, un seul mode à raisonner. Premier rendu rapide
  partout.
- **Contre** : charge serveur inutile sur les pages admin (consultées
  par 1-2 utilisateurs au total) et sur les pages auth où le serveur
  Node devrait de toute façon attendre la session côté API pour
  rendre quoi que ce soit d'utile. Latence ajoutée sans bénéfice.
- **Risque** : passer du temps à debugger des problèmes SSR
  (`window` indéfini, double-fetch) sur des pages où le SSR n'apporte
  rien.

### Angular Universal « à l'ancienne »

- **Pour** : précédent historique, beaucoup de doc existante.
- **Contre** : déprécié au profit de `@angular/ssr` moderne, qui est
  d'ailleurs ce que la CLI génère par défaut depuis Angular 17. Pas de
  rendu par route, configuration plus lourde.
- **Risque** : partir sur une API qui n'est plus celle recommandée
  par l'équipe Angular.

## Conséquences

### Positives

- **SEO sur le contenu métier**. Les fiches recette et les profils
  d'auteur sont servis en HTML pré-rendu côté serveur, donc indexables
  immédiatement et correctement aperçus par les crawlers sociaux.
- **First Contentful Paint amélioré** sur les pages publiques (le
  navigateur affiche du HTML utile avant que le bundle JS soit
  téléchargé et hydraté).
- **Pas de surcharge serveur sur les pages privées**. Les routes
  `admin/**` et `me/**` ne déclenchent aucun travail SSR : le serveur
  Node renvoie le shell, le navigateur hydrate, l'auth API fait son
  appel. C'est la voie qui correspond à la réalité métier de ces
  pages.
- **Event replay**. `withEventReplay()` capture les événements arrivés
  entre la première peinture et l'hydratation complète, et les rejoue
  au moment où les handlers Angular deviennent vivants. Pas de
  « clic qui ne fait rien » sur un bouton apparu visible mais pas
  encore réactif.
- **Bundle ESM moderne**. Le serveur Node utilise
  `AngularNodeAppEngine` + Express minimaliste
  (`frontend/src/server.ts:1-23`), pas de framework SSR custom à
  maintenir.

### Négatives

- **Le code doit être platform-aware**. Tout accès direct à
  `window`, `document`, `localStorage` ou aux API navigateur doit
  être gardé par `isPlatformBrowser(PLATFORM_ID)`. C'est le cas dans
  `App.ngOnInit` qui ne déclenche l'appel `authService.me()` que côté
  navigateur (`frontend/src/app/app.ts:21-26`), et dans tous les
  guards qui retournent `true` côté serveur pour laisser le client
  refaire la vérification
  (`frontend/src/app/core/guards/auth.guard.ts:16-17`,
  `frontend/src/app/core/guards/admin.guard.ts:17-18`,
  `frontend/src/app/core/guards/guest.guard.ts:14-15`).
- **Auth en SSR : on ne tente pas**. L'`authInterceptor`
  (`frontend/src/app/core/interceptors/auth.interceptor.ts:14-17`)
  ajoute `credentials: 'include'` pour transmettre le cookie
  `HttpOnly`. Côté serveur Node, le cookie du visiteur n'est pas
  automatiquement propagé aux appels HTTP sortants. On a tranché en
  faisant *court-circuiter les guards côté serveur* et *ne pas
  appeler `me()` côté serveur* (`frontend/src/app/app.ts:21-26`).
  Conséquence : les pages SSR sont rendues en état « non connecté »
  côté serveur, puis l'état authentifié apparaît après hydratation
  client. C'est acceptable parce que les routes auth sont en
  `RenderMode.Client`, donc le SSR ne sert que des pages publiques où
  l'état de session n'altère pas le contenu indexable.
- **Déploiement plus complexe** qu'un site statique. Il faut un
  serveur Node (Express embarqué dans le bundle SSR) qui sert les
  routes `Server` et les assets. Cf. `_draft_deployment/02-plan-railway.md`
  pour la stratégie de déploiement retenue.

### À surveiller

- **Performance du serveur Node sous charge**. À mesurer en
  production. Si la latence sur `recipes/:slug` augmente trop, ajouter
  un cache HTTP (`Cache-Control: public, max-age=...`) devant les
  routes SSR, ou pré-rendre les recettes les plus populaires.
- **Cohérence Server / Client**. Aujourd'hui un appel `RecipesService`
  effectué côté serveur pour rendre la fiche est susceptible d'être
  refait côté client juste après hydratation. À terme, introduire
  `TransferState` pour passer le résultat du fetch SSR au client et
  éviter le double-appel.
- **Cookies et CORS en SSR**. Si on décide un jour de propager le
  cookie d'auth depuis la requête entrante vers les appels API côté
  serveur (pour rendre des pages personnalisées en SSR), il faudra
  installer un interceptor SSR distinct qui lit `req.headers.cookie`
  et le forward sur les appels sortants.
- **Routes oubliées**. Toute nouvelle route doit être déclarée dans
  `app.routes.server.ts` avec son `RenderMode`. Sinon elle tombe dans
  le catch-all `**` en `Prerender`, ce qui n'est pas toujours le bon
  mode pour une route dynamique nouvellement ajoutée.

## Références code

- `frontend/src/app/app.routes.server.ts:3-67` — table de routage
  serveur, un `renderMode` par chemin (Client pour `me/**`, `admin/**`,
  `profile`, `auth/**` ; Server pour `recipes/:slug`, `users/:username` ;
  Prerender pour le catch-all `**`).
- `frontend/src/app/app.config.server.ts:6-10` — configuration serveur
  via `provideServerRendering(withRoutes(serverRoutes))`, mergée avec
  la config navigateur.
- `frontend/src/main.server.ts:5-6` — point d'entrée SSR, bootstrap
  identique au navigateur mais avec la config serveur.
- `frontend/src/app/app.config.ts:20` — hydration cliente avec
  `provideClientHydration(withEventReplay())`.
- `frontend/src/server.ts:1-23` — serveur Node Express minimal :
  statique sur `../browser` + dispatcher `AngularNodeAppEngine`.
- `frontend/src/app/app.ts:21-26` — garde
  `isPlatformBrowser(PLATFORM_ID)` avant l'appel `authService.me()`
  pour ne pas tenter l'auth côté serveur.
- `frontend/src/app/core/guards/auth.guard.ts:16-17`,
  `frontend/src/app/core/guards/admin.guard.ts:17-18`,
  `frontend/src/app/core/guards/guest.guard.ts:14-15` — chaque guard
  fait `if (isPlatformServer(platformId)) return true;` pour
  court-circuiter la vérification côté serveur et laisser le client
  la refaire.
- `frontend/src/app/core/interceptors/auth.interceptor.ts:14-17` —
  l'`authInterceptor` ajoute `credentials: 'include'` (cookies envoyés
  uniquement côté navigateur, contexte du choix de ne pas rendre les
  routes auth en SSR).
- `frontend/angular.json:47-50` — config build Angular avec
  `outputMode: "server"` et `ssr.entry: "src/server.ts"`.
