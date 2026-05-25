# Findings détaillés — audit statique frontend

Inventaire technique brut, par thématique, des observations faites en lisant le code source. Chaque finding est ancré sur `file:line` pour pouvoir être vérifié ou corrigé directement.

> Ce fichier sert de **base technique** pour la grille RGAA (`02-rgaa-grille.md`) et le plan de remédiation (`04-plan-remediation.md`). Il peut rester dans `_draft_bloc1_audit/` et **ne pas être copié** dans le repo `documentation` si on préfère un livrable plus synthétique côté jury.

Périmètre : `frontend/src/` (Angular 21, SSR, standalone, signals, Bootstrap 5 partiel).

---

## 1. HTML sémantique

### Bon
- `<main class="rs-main">` autour de `<router-outlet>` — `app/app.html:3-5`.
- `<rs-header>` utilise `<header>` + `<nav class="navbar" aria-label="En tête">` — `app/layouts/header/header.html:1-2`.
- `<footer aria-label="Pied de page">` + `<nav aria-label="Liens légaux">` imbriqué — `app/layouts/footer/footer.html:1,7`.
- Fiche recette utilise `<article>`, `<header>`, `<aside>`, `<section>` proprement — `app/pages/recipes/detail/recipe-detail.html:17,18,79,80,111`.
- Commentaires : chaque commentaire = `<article>` avec `<header>` — `app/pages/recipes/detail/recipe-detail.html:144,239`.
- Listes : `<ul>` / `<ol>` pour ingrédients et étapes — `app/pages/recipes/detail/recipe-detail.html:83,114`.
- Définition list (`<dl>/<dt>/<dd>`) pour profil utilisateur — `app/pages/admin/users/user-detail/user-detail.html:40-60`.
- Un seul `<h1>` par page, `<h2>` de section — `app/pages/home/home.html:5,17,45`, `app/pages/recipes/detail/recipe-detail.html:25,81,96,112,129`, `app/pages/profile/profile.html:5,34,83,130`, `app/pages/about/about.html:2,7,14,20`.
- `aria-labelledby` qui lie une section à son titre — `app/pages/home/home.html:15,42`, `app/pages/recipes/submit/recipe-form.html:41,76,94,112`, `app/pages/users/profile/profile.html:35`.

### Manquant / risqué
- **`<main>` imbriqué dans `<main>`** — `app/pages/admin/users/user-detail/user-detail.html:36` et `app/pages/admin/review/review.html:33` ajoutent un `<main class="rs-user-content">` à l'intérieur du `<main>` du layout (`app/app.html:3`). RGAA 9.1 → un seul `<main>` par page. **Fix** : passer en `<section>` ou `<div role="region" aria-label="...">`.
- **`role="list"` / `role="listitem"` sur `<div>`** — `app/pages/contact/contact.html:12-29`. Antipattern, utiliser un vrai `<ul>/<li>`.
- **Cartes de recettes en `<h2>`** sous une section dont le titre est aussi `<h2>` — `app/shared/recipe-card/recipe-card.html:14` vs `app/pages/home/home.html:45`. Idéalement `<h3>` pour les cartes.
- **Boutons modération sans `aria-label`** — « Modérer », « Supprimer » au lecteur d'écran : sans contexte du commentaire associé. `app/pages/recipes/detail/recipe-detail.html:198-215, 277-289`. **Fix** : `[attr.aria-label]="'Supprimer le commentaire de ' + comment.author.username"`.

---

## 2. Images et alt

### Bon
- Hero correctement décoratif : `alt=""` + `width="1920" height="1280" loading="eager" fetchpriority="high" decoding="async"` — `app/pages/home/home.html:3`.
- Cartes : `alt=""` (décoratif) + `width/height/loading` — `app/shared/recipe-card/recipe-card.html:5,7`.
- Fiche recette : `alt = titre` — `app/pages/recipes/detail/recipe-detail.html:72`.
- Aperçu image dans le formulaire avec alt descriptif — `app/pages/recipes/submit/recipe-cover-image-form.html:14`.
- Préchargement hero : `<link rel="preload" as="image" href="/assets/images/home/home-hero.webp" type="image/webp" fetchpriority="high">` — `src/index.html:11`.
- Toutes les `<img>` ont `width` et `height` → pas de CLS.

### Manquant / risqué
- **`NgOptimizedImage` (`ngSrc`) jamais utilisé** (grep zero). À brancher au minimum sur `home.html:3`, `recipe-detail.html:72`, `recipe-card.html:5,7`, `review.html:51` → srcset auto, lazy auto, LCP detection.
- **Image admin review sans dimensions** — `app/pages/admin/review/review.html:51` : `<img [src]="recipe.coverImageUrl" [alt]="recipe.title">`. **Fix** : ajouter `width`, `height`, `loading="lazy"`.
- **Aperçu formulaire sans dimensions** — `app/pages/recipes/submit/recipe-cover-image-form.html:14`.
- **Typo** : `"Apercu"` (sans accent) — `recipe-cover-image-form.html:14`. **Fix** : `"Aperçu de l'image de couverture"`.

---

## 3. Formulaires

### Bon
- Tous les inputs sondés ont un `<label for="...">` correctement lié.
- `autocomplete` correct : `email`, `current-password`, `new-password`, `name` — `sign-in.html:8,21`, `sign-up.html:8,17,30,43`, `contact.html:46-99`, `profile.html:45-96`, `reset-password.html:14-28`.
- IDs dynamiques pour les form arrays — `recipe-ingredients-form.html:3-4,9-10,15-16,26-27` (`[attr.for]` + `[id]`).
- Astérisque + mention « champs marqués * obligatoires » — `contact.html:41`, `profile.html:7`.
- `<form role="search">` + label visually-hidden dans le header — `header.html:6-7,11`.
- `inputmode` adapté : `email`, `numeric`, `decimal` (`contact.html:64`, `recipe-form.html:50,58,63,68`, `recipe-ingredients-form.html:4`).
- `novalidate` pour laisser Angular gérer la validation — `contact.html:43`, `recipe-form.html:34`, `profile.html:33`.
- Erreurs `role="alert"` (`sign-up.html:72`, `contact.html:119`, `reset-password.html:40`), succès `role="status" aria-live="polite"` (`sign-in.html:29`, `contact.html:115`).
- `role="group"` + `aria-labelledby` pour les tags — `recipe-tags-form.html:10`.
- Style invalide tied à `.ng-touched.ng-invalid` — `shared/styles/auth-form.css:106-109`.

### Manquant / risqué
- **`aria-invalid` jamais positionné** (grep zero). **Fix généralisé** :
  ```html
  [attr.aria-invalid]="control.touched && control.invalid ? 'true' : null"
  ```
  à appliquer sur tous les inputs validés.
- **`aria-describedby` → message d'erreur** quasi absent. Seul `sign-up.html:56` le fait pour la checkbox CGU avec `aria-describedby="termsAccepted-error"` lié à `<p id="termsAccepted-error" ...>` (`sign-up.html:67`). **Fix généralisé** : donner un `id` aux `<p class="rs-field-error">` et référencer depuis l'input quand l'erreur est visible.
- **`required` sans `aria-required`** sur les inputs — mineur, `required` suffit dans la plupart des AT.
- **Forgot-password : erreur en dehors du `.rs-field`** — `forgot-password.html:9-13`.
- **`inputmode="name"` invalide** — `profile.html:96`. Valeurs valides : `none|text|tel|url|email|numeric|decimal|search`. **Fix** : `inputmode="text"` ou retirer.
- **Sélect multiple sans helper** — `search.html:30,42` pas de hint sur Ctrl-clic, considérer un widget chip multi-select.
- **Champ temps total sans `aria-describedby`** pour l'unité « minute(s) » — `search.html:54`.

---

## 4. Navigation clavier et focus

### Bon
- Global : `:focus-visible { outline: 3px solid var(--rs-focus-ring); outline-offset: 3px; }` — `src/styles.scss:51-54`.
- Focus renforcé sur inputs auth — `shared/styles/auth-form.css:62-67`.
- Focus visible sur submit search header — `layouts/header/header.css:79-82`.
- **Aucun `outline:none` / `outline:0`** dans le code (grep zero).
- Boutons natifs partout pour les actions cliquables.
- Toggle mobile menu expose son état : `[attr.aria-expanded]`, `aria-controls="main-navbar"`, `[attr.aria-label]` qui bascule entre « Afficher / Fermer le menu » — `header.html:20`.
- Dropdown user : `[attr.aria-expanded]` + `aria-controls="user-menu"` — `header.html:27`.

### Manquant / risqué
- **Pas de skip link** (`grep skip` zero). **Fix** dans `app/app.html` :
  ```html
  <a class="visually-hidden-focusable" href="#main-content">Aller au contenu principal</a>
  ```
  + `id="main-content"` sur `<main>`, et `tabindex="-1"` pour permettre le focus.
- **Pas de gestion de focus sur changement de route** — `app.config.ts:15-18` scroll en haut mais ne déplace pas le focus. **Fix** : injecter `Router` + écouter `NavigationEnd` + `setTimeout(() => document.querySelector('h1')?.focus())` (avec `tabindex="-1"` sur h1).
- **Dropdown user menu non accessible clavier** — `header.html:26-39` : pas de `role="menu"`/`menuitem`, pas d'Escape pour fermer, pas de flèches haut/bas, pas de retour focus sur le toggle. **Fix** : `(keydown.escape)="closeMenu()"` + `@angular/cdk/menu` ou implémentation manuelle.
- **Pas de focus trap** sur dropdown — neutre car pas de modal, mais à prévoir si modal ajouté.
- **Focus pas déplacé dans les formulaires d'édition/réponse de commentaire** — `recipe-detail.html:164-191, 218-233`. À l'ouverture, le textarea n'est pas auto-focus.

---

## 5. ARIA

### Bon
- 139 attributs `aria-*` répartis sur 33 fichiers (bonne adoption).
- `aria-live="polite"` sur chargements asynchrones — `home.html:21,23,25,39,50,52,54`, `pagination-controls.html:3`, formulaires.
- `role="alert"` pour erreurs critiques.
- `role="status"` pour succès, paired avec `aria-live="polite"`.
- `aria-labelledby` qui lie `<section>` à son titre.
- `aria-label` sur boutons icône — favori, recherche submit, user menu toggle.
- `ariaCurrentWhenActive="page"` sur les nav links — `header.html:42,50,53,56`.
- `aria-hidden="true"` + `focusable="false"` sur SVGs décoratifs — `header.html:8,13`, `recipe-card.html:41,45`, `recipe-detail.html:29,33`, `shared/category-icon/category-icon.html:1-2`.
- `aria-busy="true"` sur skeletons — `recipe-list-shell.html:18`, `recipe-detail.html:5`.

### Manquant / risqué
- **Live regions dupliquées sur la même page** — `dashboard.html:25,31,43,49,61,67,79,85` cumule 5 `aria-live` + 5 `role="alert"`. Risque d'annonces simultanées. **Fix** : une seule live region partagée par zone, swap du message.
- **Dropdown sans `role="menu"`/`menuitem`** — `header.html:26-39`. Acceptable si on garde `<ul>/<li>/<a>` natifs, mais alors clavier doit fonctionner naturellement.

---

## 6. Contraste et visuel

### Bon
- Design tokens dans `:root` — `src/styles.scss:24-49`. Primaire `#384688`, dark `#1e1e1e`, cream `#f8f5f2`, accent jaune `#fcc417`.
- Focus ring primary-dark `#2c376d` à 3px outline + 3px offset — fortement visible — `src/styles.scss:51-54`.
- Inputs : hover + focus avec bord + shadow — `shared/styles/auth-form.css:58-67`.
- Aucun `outline:none`.
- Erreur rouge `#b42318` ≈ 4.97:1 sur blanc, succès vert `#067647` ≈ 5.03:1, warning brun `#8a5400` ≈ 5.34:1 — tous WCAG AA — `shared/styles/auth-form.css:96,103,107`.
- Placeholder forcé en `opacity:1` pour neutraliser Firefox — `src/styles.scss:56-59`.

### Manquant / risqué
- **Bouton CTA jaune sur photo brightness(.75)** — `app/pages/home/home.html:8` + `pages/home/home.css:16`. Contraste runtime variable selon les pixels de l'image. **Fix** : scrim solide sombre derrière les CTA.
- **Hero texte blanc avec `text-shadow: 0 4px 20px rgba(0,0,0,.45)`** — `home.css:43-44`. WCAG ne dispense pas du 4.5:1 grâce à un shadow. **Fix** : gradient sombre plus marqué.
- **Texte muted en `rgb(--rs-dark-rgb / 0.62)` ≈ 4.4:1** — limite — `src/styles.scss:57`, `auth-form.css:15,28,45,78`. **Fix** : passer à 0.7+ ou couleur solide `#5a5a5a`.
- **Placeholder à 62% opacité** ≈ < 4.5:1 sur blanc.
- **`.text-muted` Bootstrap** sur about/privacy ≈ 4.69:1 — passe de justesse.
- **`.rs-btn:disabled` à 46% opacité** ≈ 3:1 — exempté par WCAG (disabled), mais UX faible.

---

## 7. Langue et localisation

### Bon
- `<html lang="fr">` — `src/index.html:2`.
- UI en français, ARIA labels en français, messages d'erreur en français.
- Dates via `date: 'dd/MM/yyyy'` (format locale-agnostic).
- Tri français explicite : `localeCompare(right.name, 'fr')` — `core/services/recipe-reference-data.service.ts:53,62`.

### Manquant / risqué
- **Pas de `LOCALE_ID = 'fr-FR'`** dans `app.config.ts`. Sans `registerLocaleData(localeFr)` + `{ provide: LOCALE_ID, useValue: 'fr-FR' }`, le pipe `date` utilise par défaut `en-US`. OK en pratique pour `dd/MM/yyyy` mais cassé si on utilise `MMMM`. **Fix** :
  ```ts
  import { LOCALE_ID } from '@angular/core';
  import { registerLocaleData } from '@angular/common';
  import localeFr from '@angular/common/locales/fr';
  registerLocaleData(localeFr);
  // providers: { provide: LOCALE_ID, useValue: 'fr-FR' }
  ```
- **Apostrophe typographique vs droite** incohérente — `resend-validation-email.html:3` utilise `’` alors que le reste utilise `'`. Mineur.

---

## 8. SEO

### Bon
- `<title>` statique — `src/index.html:6`.
- `<meta name="description">` — `src/index.html:7`.
- `<meta name="robots" content="index, follow">` — `src/index.html:8`.
- `<base href="/">` + viewport — `src/index.html:9,10`.
- `public/robots.txt` présent, structuré (disallow `/admin/`, `/auth/`, `/api/`, tracking params).
- SSR pour les pages publiques — `app.routes.server.ts:60-67`. Auth/profile/admin en `RenderMode.Client`.

### Manquant / risqué
- **Angular `Title`/`Meta` jamais utilisés** (grep usage zero hors imports). Tous les onglets affichent le même `Recipe Shelter — Cook. Share. Discover.`. **Fix** : injecter `Title`/`Meta` dans `recipe-detail.ts`, `users/profile/profile.ts`, `home.ts`, `search.ts`, etc. `title.setTitle(...)` + `meta.updateTag(...)` au chargement des données.
- **Pas d'OpenGraph / Twitter Card** dans `index.html`. **Fix** : ajouter `og:title`, `og:description`, `og:image`, `og:type` (statique + dynamique côté pages détail).
- **Pas de `sitemap.xml`** dans `public/`. **Fix** : générer à la build SSR avec la liste des slugs prerendered.
- **Pas de JSON-LD `Recipe`** — manque rich results Google. **Fix** : `<script type="application/ld+json">` dans `recipe-detail.html` avec `@type: Recipe`, ingredients, steps, prep/cook time, ratings.
- **Pas de `<link rel="canonical">`** — risque de duplicate content.
- **Robots.txt bloque `/assets/`** — à vérifier (risque pour Google Images).

---

## 9. Performance

### Bon
- Lazy routes partout — `app/app.routes.ts:12-54`.
- SSR + hydratation client avec event replay — `app.config.ts:20`.
- HTTP Fetch API (pas XHR) — `app.config.ts:22-24`.
- Hero préchargé WebP + width/height + `fetchpriority="high"` + `decoding="async"` — `src/index.html:11`, `home.html:3`.
- Première carte de la liste en `loading="eager"`, suivantes en `lazy`, `fetchpriority="high"` seulement sur la première — `recipe-list.html:3`, `recipe-card.html:5,7`.
- Skeletons pendant chargement — `recipe-detail.html:5-11`, `recipe-list-shell.html:18-22`.
- Prerender + SSR + CSR mix par route — `app.routes.server.ts`.
- Bootstrap cherry-picked (nav, navbar, dropdown, buttons, helpers seulement) — `src/styles.scss:1-19`.

### Manquant / risqué
- **`ChangeDetectionStrategy.OnPush` jamais utilisé** (grep zero). Avec signals déjà en place, OnPush est gratuit. **Fix** : ajouter à chaque composant standalone.
- **`@defer` jamais utilisé** (grep zero). **Fix** : envelopper la section commentaires de la fiche recette en `@defer (on viewport) { ... } @placeholder { ... }` — `recipe-detail.html:126`.
- **Pas de `NgOptimizedImage`** — voir §2.
- **Pas de `<link rel="preconnect">`** vers l'API — `src/index.html`. **Fix** : `<link rel="preconnect" href="https://api.recipe-shelter.fr">`.
- **Liste de commentaires non virtualisée** — acceptable à scale faible, mais penser à `@defer (on interaction)` pour les replies.

---

## 10. Responsive

### Bon
- Viewport : `<meta name="viewport" content="width=device-width, initial-scale=1">` — pas de `maximum-scale` (a11y respectée).
- 38 `@media` répartis sur 26 fichiers.
- Typographie fluide via `clamp()` — `home.css:31,37`, `auth-form.css:3`, `styles.scss:64`.
- Navbar collapse < 992px — `header.css:138`.

### Manquant / risqué
- **`min-height: 520px` du hero** — `home.css:5`. Sur petit landscape, plus haut que le viewport. **Fix** : `min-height: clamp(360px, 60vh, 520px)`.
- **Hero overlay `width: 50%`** — `home.css:22`. Probablement override mobile, à vérifier.
- **Pas de `prefers-reduced-motion`** — transitions partout. **Fix** : wrap dans `@media (prefers-reduced-motion: no-preference)`.
- **Pas de `prefers-color-scheme: dark`** — choix de design assumable, mais variables dark déjà importées sans être utilisées (`styles.scss:4`).
- **Touch target submit search header 32×32** — `header.css:68,71`. WCAG 2.5.5 AAA = 44×44, AA = 24×24 → passe AA, sous AAA.

---

## 11. RGPD / consentement

### Bon
- Politique de confidentialité — `pages/legal/privacy/privacy.html`.
- Inscription nécessite checkbox consentement CGU + privacy — `sign-up.html:54-69`.
- Texte usage des données inline — `sign-up.html:58-62`.
- Hint contact « email pour répondre » — `contact.html:126`.
- CGU/Privacy/Mentions Légales liés dans footer — `footer.html:8-10`.
- Privacy mentionne cookies (`privacy.html:80`) et responsable de traitement (`privacy.html:14-20`).

### Manquant / risqué
- **Pas de bannière consentement** (grep `cookie|consent|cnil|tarteaucitron|axeptio` → seulement texte statique). Si analytics/tracking en place : non-conforme CNIL. **Fix** :
  - Soit déclarer dans privacy que SEUL le cookie session auth est utilisé (pas de bannière requise).
  - Soit implémenter une bannière (ngx-cookie-consent ou custom).
- **Contradiction** : privacy parle de cookies « à des fins de mesure d'audience » (`privacy.html:81`) sans bannière. **Fix urgent** : aligner le texte avec la réalité (probablement retirer la mention « mesure d'audience »).
- **Pas de bouton « Supprimer mon compte »** visible — `profile.html` change email/username/password mais pas de delete. **Fix** : ajouter le flux suppression.
- **Pas de log/horodatage de consentement** côté inscription (seulement booléen). Côté backend : tracer version CGU + timestamp.

---

## 12. Gestion d'erreurs

### Bon
- Erreurs inline en FR sur tous les inputs.
- `role="alert"` pour erreurs API, `role="status" aria-live="polite"` pour succès.
- Skeletons pendant chargement.
- États vides dédiés — `recipe-list-shell.html:27-32`, `recipe-detail.html:346-350`, `users/profile.html:41-44`.
- Recette introuvable avec `<h1>` + explication — `recipe-detail.html:346-350`.
- Reset-password gère token manquant gracieusement — `reset-password.html:5-10`.
- Validate-email gère `loading | success | error` avec retry — `validate-email.html:5-15`.
- Submit buttons disable + label « Envoi... » — anti double-soumission.

### Manquant / risqué
- **Pas de page 404 globale** — `app.routes.ts:57` fait `redirectTo: ''` → home affichée à la place + statut HTTP 200. Mauvais SEO + UX. **Fix** :
  - Créer `pages/not-found/not-found.ts`.
  - Remplacer `{ path: '**', redirectTo: '' }` par `{ path: '**', loadComponent: ... }`.
  - En SSR, retourner HTTP 404.
- **Pas d'intercepteur HTTP global pour erreurs** — `app.config.ts:23` ne charge que `authInterceptor`. Erreurs réseau bubbles par page. **Fix** : ajouter un error interceptor → toast global ou signal.
- **Pas de bouton « Réessayer »** sur erreurs de liste — `recipe-list-shell.html:25`.
- **`provideBrowserGlobalErrorListeners()`** présent — `app.config.ts:12` — mais pas de surface UI pour erreurs capturées.
- **Pas de gestion offline / network failure** — pas de PWA / service worker (grep `service-worker` zero). Optionnel pour le scope cert.

---

## Notes méthodologiques

- **Audit statique** : pas d'exécution réelle, pas de Lighthouse, pas de NVDA/VoiceOver. Certaines règles (contraste exact d'images de fond, performance réelle) demandent l'app en marche.
- **Périmètre** : `frontend/src/` complet, échantillonné par feature (1–2 templates par zone fonctionnelle + global styles + layout + composants partagés clés).
- **Outils** : grep / read de fichier source uniquement. Pas d'eslint a11y plugin, pas d'`axe`, pas de `pa11y`.
- **Sources des règles** : RGAA 4.1.2 (https://accessibilite.numerique.gouv.fr/), WCAG 2.1 AA, Angular best practices (https://angular.dev), Lighthouse v11 weighting.
