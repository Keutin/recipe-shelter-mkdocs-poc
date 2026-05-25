# 01 — Bloc 1 : Front-End HTML/CSS/JS

Questions ciblées sur le **Bloc 1** : HTML5/CSS3/JavaScript pur, responsivité, accessibilité, performance.

> ⚠ **Point de vigilance** — Vérifier avec Arthur si le Bloc 1 attend un livrable HTML/CSS/JS **séparé** du Bloc 3 Angular (landing page, pages institutionnelles…), ou si l'Angular couvre les deux. Les questions ci-dessous supposent qu'il y a bien un livrable HTML/CSS/JS distinct. Si ce n'est pas le cas, les questions de ce fichier valent aussi pour les **templates HTML générés par Angular**.

---

## Q1. Quelles techniques avez-vous utilisées pour rendre le site responsive ?

**Intention jury** — Vérifier les bases CSS et la maîtrise du *mobile-first* (ou *desktop-first* assumé).

**Ossature de réponse**

1. **Approche mobile-first** : les media queries ajoutent des règles à partir de certaines largeurs (`min-width: 768px`, `min-width: 1024px`).
2. **Layouts modernes** : Flexbox pour les composants linéaires (header, listes de cartes), Grid pour les pages avec colonnes.
3. **Unités relatives** : `rem` pour la typo, `%` ou `fr` pour les largeurs, `vh/vw` parcimonieux.
4. **Images responsive** : `<picture>` ou `srcset` si applicable, sinon `max-width: 100%; height: auto`.
5. **Test** : BrowserStack pour valider sur plusieurs navigateurs et tailles, plus le devtools responsive de Chrome au quotidien.

**Ancres** — citer un fichier CSS précis avec une media query.

**Pièges**
- ❌ Confondre responsive et adaptive.
- ❌ « J'ai utilisé Bootstrap, donc c'est responsive » → si Bootstrap est utilisé en Bloc 3 Angular, dire que ça **complète** mais que les règles custom restent appliquées.

---

## Q2. Avez-vous utilisé un framework CSS ? Si oui lequel et pourquoi ?

**Intention jury** — Voir si le choix est conscient.

**Ossature de réponse**

- Bloc 1 (HTML/CSS pur) : *(à confirmer avec Arthur — vraisemblablement pas de framework, ou alors un reset/normalize)*
- Bloc 3 Angular : **Bootstrap 5** pour la grille, les composants de base (boutons, formulaires, modales) et la cohérence visuelle. Bootstrap 5 ne dépend plus de jQuery — c'est un atout.
- Conséquence assumée : un peu de classes utilitaires dans les templates ; en contrepartie, vélocité accrue et moins de bugs cross-browser.

**Pièges**
- ❌ Dire qu'on a tout pris dans Bootstrap → laisse penser qu'il n'y a pas eu de design propre.

---

## Q3. Comment avez-vous traité l'accessibilité ?

**Intention jury** — C'est dans le cahier des charges (« respect WCAG/RGAA, rôles ARIA »). C'est **graded**.

**Ossature de réponse**

1. **Sémantique HTML5** d'abord : `<header>`, `<nav>`, `<main>`, `<article>`, `<footer>`, `<button>` vs `<a>` selon l'intention, `<label>` lié à chaque `<input>`.
2. **Rôles ARIA quand le HTML natif ne suffit pas** : `role="alert"` pour les toasts, `aria-live` pour les zones dynamiques, `aria-label` sur les boutons-icônes.
3. **Navigation clavier** : tous les éléments interactifs sont atteignables au `Tab`, focus visible (outline non supprimé), `Esc` ferme les modales.
4. **Contrastes** : vérifiés avec l'inspecteur Lighthouse / l'outil de contraste de Chrome (cible AA : 4.5:1 sur le texte normal).
5. **Tests** : Lighthouse en mode Accessibility, et **navigation manuelle au clavier uniquement** pour valider.

**Ancres** — pouvoir ouvrir un template et montrer un `aria-label`, un `<label for>`.

**Pièges**
- ❌ Réciter ARIA sans pouvoir donner un exemple concret du code.
- ❌ Dire « j'ai fait du AAA » sans pouvoir le justifier — viser **AA**.

---

## Q4. Quel score Lighthouse obtenez-vous ?

**Intention jury** — Vérifier qu'Arthur a vraiment passé Lighthouse.

**Ossature de réponse** *(à compléter après run réel — c'est le chantier #5 de la roadmap)*

- **Performance** : *[score]* — citer 1-2 optimisations (compression d'images, lazy-loading, defer sur les scripts).
- **Accessibilité** : *[score]* — viser ≥ 90.
- **Best Practices** : *[score]*.
- **SEO** : *[score]*.

Si certains scores sont bas, **avoir une explication** : « Le score performance descend à 75 sur la page recette à cause des images uploadées par les utilisateurs que je ne contrôle pas — en prod je rajouterais un service d'optimisation type Cloudinary ou imgproxy. »

**Pièges**
- ❌ « Je n'ai pas testé avec Lighthouse » → fatal vu que c'est dans le cahier des charges.

> **⚠ TODO Arthur** : faire tourner Lighthouse sur 3 pages (accueil, liste recettes, détail recette) et noter les scores ici. Voir chantier #5 (Bloc 1 punch list).

---

## Q5. Comment avez-vous géré les images et leur poids ?

**Intention jury** — Performance et pragmatisme.

**Ossature de réponse**

1. **Images de l'app** (logos, illustrations) : optimisées en amont (SVG quand vectoriel, WebP ou PNG compressé sinon).
2. **Images des recettes uploadées par les utilisateurs** : stockées en *[à compléter — base ? FS ? CDN ?]*, servies via *[à compléter]*. Limite de taille à l'upload côté back.
3. **Lazy loading** : `loading="lazy"` sur les `<img>` en dehors de la fold.
4. **Dimensions explicites** (`width`/`height`) pour éviter le layout shift (CLS).

**Pièges**
- ❌ Si les images sont stockées en base64 en base ou non optimisées du tout, mieux vaut le dire et l'assumer comme une limite.

---

## Q6. Comment avez-vous structuré vos fichiers CSS ?

**Intention jury** — Maintenabilité.

**Ossature de réponse** — Citer la méthode adoptée :

- **Découpage par composant** : `styles/components/recipe-card.css`, `styles/components/nav.css`…
- **Variables CSS** dans `:root` pour la palette de couleurs, les espacements, les radius.
- **Pas de !important** sauvage.
- *(Si applicable)* SCSS avec partials et un `@import` principal.

**Pièges**
- ❌ Un seul fichier CSS de 2000 lignes → l'assumer comme une limite plutôt que prétendre le contraire.

---

## Q7. Quel JavaScript avez-vous écrit en vanilla ? (Bloc 1, hors Angular)

**Intention jury** — Vérifier que le Bloc 1 a bien du JS, pas juste du HTML/CSS statique.

**Ossature de réponse**

*(À compléter selon ce qui existe vraiment dans le Bloc 1.)*

Exemples probables :
- Validation côté client de formulaires (`<form>` + `event.preventDefault()` + vérif champs).
- Toggle de menu mobile (clic sur le burger → ajoute classe `open` sur le `<nav>`).
- Carrousel d'images en JS pur.
- Recherche avec filtrage à la volée (`input` event + filtrage d'une liste en DOM).

**Pièges**
- ❌ Ne pas avoir de JS vanilla si le Bloc 1 est censé être séparé d'Angular.

> **⚠ TODO Arthur** : clarifier ce qui constitue le livrable Bloc 1 vs Bloc 3 ; ajuster cette question en conséquence.

---

## Q8. Comment avez-vous testé la compatibilité entre navigateurs ?

**Intention jury** — Cahier des charges mentionne BrowserStack.

**Ossature de réponse**

1. **BrowserStack** (ou les outils intégrés de Chrome/Firefox) pour tester sur les principaux : Chrome, Firefox, Edge, Safari (desktop + mobile).
2. **Polyfills / fallbacks** : pas nécessaires sur les browsers modernes ciblés (pas de support IE11), mais conscience qu'il faut tester chaque fonctionnalité CSS récente (Grid, `gap`, `:has()` si utilisé).
3. **Mobile** : tests sur iOS Safari et Chrome Android, attention particulière aux interactions tactiles.

**Pièges**
- ❌ « J'ai testé seulement sur Chrome » → c'est dans le cahier des charges de tester ailleurs.

---

## Q9. Le site fonctionne-t-il sans JavaScript ?

**Intention jury** — Question fine sur l'accessibilité et la robustesse.

**Ossature de réponse** — Honnête :

- Les pages statiques (accueil HTML/CSS) fonctionnent sans JS.
- L'application Angular **ne fonctionne pas sans JS** par nature (SPA). Cependant, le **SSR (Server-Side Rendering)** activé permet de servir une première version HTML rendue côté serveur, ce qui :
  - améliore le SEO,
  - améliore le temps de premier rendu,
  - rend la page lisible pour un bot ou un user-agent sans JS, même si l'interactivité reste désactivée.

**Ancres** — `frontend/angular.json` (clé `ssr`), ou la config de SSR du projet.

---

## Q10. Comment gérez-vous les formulaires (validation, erreurs) ?

**Intention jury** — Robustesse côté front + UX.

**Ossature de réponse**

1. **Côté HTML** : attributs natifs (`required`, `type="email"`, `minlength`, `pattern`).
2. **Côté JS / Angular** : validation programmatique avant envoi (Angular Reactive Forms côté Bloc 3 — voir question Bloc 3).
3. **Feedback utilisateur** : messages d'erreur inline sous chaque champ, focus automatique sur le premier champ en erreur, `aria-invalid` + `aria-describedby` pour la liaison avec le message.
4. **Côté back** : DTO de validation qui re-vérifie tout — **ne jamais faire confiance au client**.

**Ancres** — un formulaire d'inscription ou de création de recette.

**Pièges**
- ❌ Dire qu'on valide seulement côté client → faille de sécu énorme.

---

## Q11. Avez-vous mis en place une stratégie de cache ou des optimisations de chargement ?

**Intention jury** — Performance perçue.

**Ossature de réponse**

- **Headers HTTP de cache** sur les assets statiques (configurés au niveau du serveur web / hébergeur).
- **Lazy-loading** des images.
- **`defer` / `async`** sur les `<script>`.
- **Minification** : Angular fait ça out-of-the-box en build de prod.
- *(Si applicable)* Service Worker pour le mode offline / PWA — sinon, mentionner que ce serait une évolution.

**Pièges**
- ❌ « J'ai mis du cache » sans pouvoir dire où.

---

## Q12. Comment respectez-vous le RGAA et le WCAG ?

**Intention jury** — Niveau de référence légal en France.

**Ossature de réponse**

1. **WCAG 2.1 niveau AA visé** : c'est le seuil pratique recommandé (AAA est rarement atteignable sans contraintes très lourdes).
2. **RGAA** : référentiel français qui s'aligne sur WCAG. Les critères principaux respectés :
   - **Perceptible** : alternatives textuelles sur les images (`alt`), contrastes ≥ 4.5:1.
   - **Utilisable** : navigation clavier complète, focus visible.
   - **Compréhensible** : labels explicites, messages d'erreur clairs.
   - **Robuste** : HTML valide, ARIA correctement utilisé.
3. **Outils de vérification** : Lighthouse + extension axe DevTools (si utilisée).

**Pièges**
- ❌ Confondre RGAA et RGPD (l'un est l'accessibilité, l'autre la donnée perso).
- ❌ Citer WCAG 1.0 → obsolète.
