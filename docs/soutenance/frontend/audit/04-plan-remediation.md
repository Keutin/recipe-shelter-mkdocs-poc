# Plan de remédiation priorisé

Corrections classées par **rapport effort / impact**. Chaque correction indique le critère RGAA concerné, le fichier exact à modifier, l'effort estimé, et un exemple de patch.

> **Stratégie soutenance** : choisir 3 à 5 corrections symboliques pour montrer une amélioration entre le constat et la défense, plutôt que de tout corriger. Le jury évalue la rigueur méthodo, pas la perfection.

## P1 — Quick wins haute visibilité (faire avant soutenance)

### P1.1 — Skip link « Aller au contenu principal »

- **Critère RGAA** : 12.7
- **Effort** : 10 min
- **Fichier** : `frontend/src/app/app.html`
- **Patch** :
  ```html
  <!-- Avant <main> -->
  <a class="visually-hidden-focusable rs-skip-link" href="#main-content">
    Aller au contenu principal
  </a>
  <main id="main-content" tabindex="-1" class="rs-main">
    <router-outlet></router-outlet>
  </main>
  ```
  Et dans `src/styles.scss` (Bootstrap fournit `.visually-hidden-focusable` mais à vérifier qu'il est importé, sinon CSS personnalisé) :
  ```scss
  .rs-skip-link {
    position: absolute;
    top: -40px;
    left: 0;
    z-index: 999;
    background: var(--rs-primary, #384688);
    color: #fff;
    padding: 0.5rem 1rem;
    text-decoration: none;
  }
  .rs-skip-link:focus { top: 0; }
  ```
- **Justification jury** : « C'est la première barre fonctionnelle qu'un utilisateur clavier rencontre. RGAA 12.7. »

### P1.2 — Titres de page dynamiques

- **Critère RGAA** : 8.5, 8.6
- **Effort** : 30 min (3-4 pages critiques)
- **Fichiers** : `pages/recipes/detail/recipe-detail.ts`, `pages/users/profile/profile.ts`, `pages/home/home.ts`, `pages/search/search.ts`
- **Patch (exemple pour recipe-detail.ts)** :
  ```ts
  import { Title, Meta } from '@angular/platform-browser';
  // …
  constructor(private title: Title, private meta: Meta) {}
  
  // dans le effect/computed qui charge la recette
  effect(() => {
    const r = this.recipe();
    if (r) {
      this.title.setTitle(`${r.title} — Recipe Shelter`);
      this.meta.updateTag({ name: 'description', content: r.shortDescription });
      this.meta.updateTag({ property: 'og:title', content: r.title });
      this.meta.updateTag({ property: 'og:image', content: r.coverImageUrl });
      this.meta.updateTag({ property: 'og:type', content: 'article' });
    }
  });
  ```
- **Justification jury** : « SEO + accessibilité — un utilisateur de lecteur d'écran annonce le titre d'onglet en changeant de page. Sans titre dynamique, il n'a aucun moyen de différencier deux recettes. »

### P1.3 — Page 404 dédiée

- **Critère RGAA** : non strict, mais qualité Lighthouse + UX
- **Effort** : 30 min
- **Fichiers** : créer `pages/not-found/not-found.ts` + `.html`, modifier `app.routes.ts:57`
- **Patch** :
  ```ts
  // app.routes.ts
  { path: '**', loadComponent: () => import('./pages/not-found/not-found').then(m => m.NotFound) }
  ```
  `not-found.html` :
  ```html
  <section class="rs-state-card rs-state-card--error">
    <h1>Page introuvable</h1>
    <p>L'adresse demandée n'existe pas ou a été déplacée.</p>
    <a routerLink="/" class="rs-btn rs-btn-primary">Retour à l'accueil</a>
    <a routerLink="/recipes" class="rs-btn rs-btn-secondary">Voir les recettes</a>
  </section>
  ```
  Pour le statut HTTP 404 côté SSR : injecter `Response` du serveur Express et `res.status(404)` dans le composant (ou via un guard).

### P1.4 — `role="list"` sur `<div>` → vrai `<ul>/<li>`

- **Critère RGAA** : 8.9, 9.3
- **Effort** : 5 min
- **Fichier** : `pages/contact/contact.html:12-29`
- **Patch** : remplacer
  ```html
  <div role="list">
    <div role="listitem">...</div>
  </div>
  ```
  par
  ```html
  <ul class="rs-info-list">
    <li>...</li>
  </ul>
  ```

### P1.5 — Enregistrer la locale `fr-FR`

- **Critère** : qualité (pas RGAA strict)
- **Effort** : 5 min
- **Fichier** : `frontend/src/app/app.config.ts`
- **Patch** :
  ```ts
  import { LOCALE_ID } from '@angular/core';
  import { registerLocaleData } from '@angular/common';
  import localeFr from '@angular/common/locales/fr';
  
  registerLocaleData(localeFr);
  
  export const appConfig: ApplicationConfig = {
    providers: [
      // … existants
      { provide: LOCALE_ID, useValue: 'fr-FR' },
    ],
  };
  ```

**Total P1 : ~1 h 20 min, 5 critères/qualités corrigés.**

---

## P2 — Corrections moyennement coûteuses (si temps avant soutenance)

### P2.1 — `aria-invalid` + `aria-describedby` sur tous les formulaires

- **Critères** : 11.10, 11.11
- **Effort** : 1–2 h (5+ formulaires)
- **Fichiers** : `pages/auth/sign-in`, `sign-up`, `forgot-password`, `reset-password`, `pages/contact`, `pages/profile`, `pages/recipes/submit/*`
- **Pattern à appliquer** :
  ```html
  <input
    id="email"
    type="email"
    formControlName="email"
    [attr.aria-invalid]="emailControl.touched && emailControl.invalid ? 'true' : null"
    [attr.aria-describedby]="emailControl.touched && emailControl.invalid ? 'email-error' : null"
  >
  <p *ngIf="emailControl.touched && emailControl.invalid" id="email-error" class="rs-field-error" role="alert">
    Veuillez saisir une adresse email valide.
  </p>
  ```
- **Astuce** : extraire dans un composant `<rs-field>` ou une directive `rsFieldError` pour DRY.

### P2.2 — `<main>` imbriqué en admin

- **Critère** : 9.1, 9.2
- **Effort** : 10 min
- **Fichiers** : `pages/admin/users/user-detail/user-detail.html:36`, `pages/admin/review/review.html:33`
- **Patch** : remplacer `<main class="rs-user-content">` par `<section class="rs-user-content" aria-label="Détails utilisateur">` (et idem pour review).

### P2.3 — `NgOptimizedImage` sur images clés

- **Critère** : qualité performance Lighthouse
- **Effort** : 30 min
- **Fichiers** : `home.html:3`, `recipe-detail.html:72`, `recipe-card.html:5,7`, `review.html:51`
- **Patch (exemple `recipe-card`)** :
  ```ts
  import { NgOptimizedImage } from '@angular/common';
  // imports: [NgOptimizedImage]
  ```
  ```html
  <img ngSrc="{{ recipe.coverImageUrl }}" width="400" height="300" priority="false" alt="">
  ```
  Pour le hero, ajouter `priority` (LCP candidate).

### P2.4 — `ChangeDetectionStrategy.OnPush` global

- **Critère** : qualité performance
- **Effort** : 20 min (find/replace)
- **Patch** : ajouter `changeDetection: ChangeDetectionStrategy.OnPush` au `@Component()` de tous les composants standalone. Compatible avec signals (déjà utilisés).

### P2.5 — `aria-describedby` corrigé sur `forgot-password.html`

- **Critère** : 11.11
- **Effort** : 2 min
- **Fichier** : `pages/auth/forgot-password/forgot-password.html:9-13`
- **Fix** : déplacer le bloc d'erreur à l'intérieur du `<div class="rs-field">`.

### P2.6 — Typo + dimensions image review

- **Critères** : 1.1, perf
- **Effort** : 2 min
- **Fichiers** : `pages/recipes/submit/recipe-cover-image-form.html:14`, `pages/admin/review/review.html:51`
- **Patch** : `"Apercu"` → `"Aperçu de l'image de couverture"` + ajouter `width="600" height="400" loading="lazy"`.

### P2.7 — `inputmode="name"` invalide

- **Critère** : 11.13
- **Effort** : 1 min
- **Fichier** : `pages/profile/profile.html:96`
- **Patch** : `inputmode="text"` ou retirer.

**Total P2 : ~3–4 h, 7 corrections impactantes.**

---

## P3 — Corrections complexes (post-soutenance)

### P3.1 — Dropdown user menu accessible clavier

- **Critère** : 7.3, 12.11
- **Effort** : 2 h
- **Fichier** : `layouts/header/header.html:26-39` + `header.ts`
- **Approche** : `@angular/cdk/menu` (pattern WAI-ARIA Menu Pattern) ou refactor manuel avec :
  - `(keydown.escape)` → ferme + focus sur toggle
  - `(keydown.arrowDown)` → focus item suivant
  - `(keydown.arrowUp)` → focus item précédent
  - Click outside → ferme

### P3.2 — Focus déplacé sur changement de route

- **Critère** : 12.8, qualité
- **Effort** : 1 h
- **Fichier** : `app.config.ts` ou guard global
- **Approche** :
  ```ts
  import { Router, NavigationEnd } from '@angular/router';
  
  // au bootstrap
  router.events.pipe(filter(e => e instanceof NavigationEnd)).subscribe(() => {
    setTimeout(() => {
      const h1 = document.querySelector('main h1') as HTMLElement | null;
      if (h1) { h1.setAttribute('tabindex', '-1'); h1.focus(); }
    });
  });
  ```

### P3.3 — JSON-LD `Recipe` schema + OpenGraph dynamique + sitemap

- **Critère** : qualité SEO (pas RGAA)
- **Effort** : 4 h
- **Approche** : générer `<script type="application/ld+json">` dans `recipe-detail.html`, étendre la build SSR pour produire `public/sitemap.xml` à partir de la liste des slugs prerendered.

### P3.4 — Cohérence contrastes (muted, hero, placeholder)

- **Critère** : 3.2, 10.5
- **Effort** : 1 h
- **Fichiers** : `src/styles.scss:56-59`, `auth-form.css:15,28,45,78`, `home.css:43-44`
- **Approche** : remplacer les `opacity 0.62` par solides `#5a5a5a` ; renforcer le gradient sombre du hero.

### P3.5 — Cohérence RGPD (bannière ou retrait du texte « mesure d'audience »)

- **Critère** : RGPD/CNIL
- **Effort** : 30 min (si on retire le texte) à 4 h (si on implémente une bannière)
- **Décision** : si aucune analytics en place réellement → modifier `privacy.html:81` pour retirer la mention. Sinon implémenter une vraie bannière.

### P3.6 — `prefers-reduced-motion`

- **Critère** : 13.8
- **Effort** : 30 min
- **Fichier** : `src/styles.scss` + composants à animations
- **Patch** :
  ```css
  @media (prefers-reduced-motion: reduce) {
    *, *::before, *::after {
      animation-duration: 0.001ms !important;
      transition-duration: 0.001ms !important;
    }
  }
  ```

### P3.7 — `@defer` sur sections lourdes

- **Critère** : qualité perf
- **Effort** : 1 h
- **Fichier** : `recipes/detail/recipe-detail.html:126` (commentaires)
- **Patch** :
  ```html
  @defer (on viewport) {
    <section aria-labelledby="comments-title">
      <!-- … commentaires existants … -->
    </section>
  } @placeholder {
    <div class="rs-skeleton">Chargement des commentaires…</div>
  }
  ```

### P3.8 — Bouton « Supprimer mon compte » RGPD

- **Critère** : RGPD
- **Effort** : variable (UI 1 h, backend déjà supposé)
- **Fichier** : `pages/profile/profile.html` (section sécurité)

**Total P3 : ~10–12 h.**

---

## Récapitulatif effort vs impact

| Lot | Effort | Critères couverts | Recommandation |
| --- | --- | --- | --- |
| **P1** | ~1 h 20 | 5 critères RGAA / qualité (8.5, 8.6, 12.7, 8.9, 9.3) + qualité locale | **À faire avant soutenance** |
| **P2** | ~3–4 h | 7 critères (11.10, 11.11, 9.1, 1.1, 11.13, perf, CD) | À faire si possible avant soutenance |
| **P3** | ~10–12 h | 8 chantiers (dropdown, focus, SEO, contraste, RGPD, motion, defer, RGPD UI) | Post-soutenance / roadmap |

## Comment présenter au jury

Slide possible (à intégrer dans `_draft_slides/soutenance-slides.md`) :

```
## Accessibilité — où on en est

État au début de l'audit : 45 critères conformes / 16 partiels / 3 non conformes / 17 N/A
                          → ~70 % de conformité sur le périmètre applicable.

Corrigé pendant l'audit :
- Skip link ajouté (RGAA 12.7)
- Titres de page dynamiques (RGAA 8.5/8.6)
- Page 404 dédiée
- Locale fr-FR enregistrée
- role="list" sur div → ul/li (RGAA 8.9)

Plan documenté pour la suite (effort estimé : 3 h P2 + 10 h P3).
```

C'est la **séquence avant/après** + le **plan documenté** qui valide le Bloc 1 — pas un score idéal.
