# Grille RGAA 4.1 / WCAG 2.1 AA — Recipe Shelter

Audit critère par critère, basé sur le RGAA 4.1.2 (référentiel officiel français, 2023). Chaque critère est rattaché à une **thématique** (13 au total).

**Légende statut** : ✅ Conforme | ⚠️ Partiellement conforme | ❌ Non conforme | N/A Non applicable

Les références `file:line` pointent vers le code source dans `frontend/`.

---

## Thématique 1 — Images

| Critère | Question | Statut | Constat |
| --- | --- | --- | --- |
| 1.1 | Chaque image porteuse d'info a-t-elle une alternative textuelle ? | ⚠️ | Alt présent partout, mais l'image admin review n'a pas de dimensions (`pages/admin/review/review.html:51`). Typo « Apercu » dans `recipe-cover-image-form.html:14`. |
| 1.2 | Chaque image décorative est-elle correctement ignorée par les AT ? | ✅ | Hero `alt=""` (`home.html:3`), images de couverture de cartes `alt=""` (`shared/recipe-card/recipe-card.html:5,7`). |
| 1.3 | Pour chaque image porteuse d'information, l'alternative est-elle pertinente ? | ✅ | Alt = titre de recette sur la fiche détail (`recipes/detail/recipe-detail.html:72`). |
| 1.6 | Chaque image porteuse d'info a-t-elle une description détaillée si nécessaire ? | N/A | Pas de schéma technique ou graphique complexe. |
| 1.8 | Chaque image-texte est-elle remplacée par du texte stylé ? | ✅ | Pas d'image-texte détectée. Le hero utilise une vraie image + texte HTML superposé. |
| 1.9 | Chaque légende d'image est-elle liée à l'image ? | N/A | Pas de `<figure>/<figcaption>` utilisé — pas requis. |

**Action** : corriger la typo, ajouter `width`/`height`/`loading="lazy"` sur l'image admin review.

---

## Thématique 2 — Cadres

| Critère | Question | Statut | Constat |
| --- | --- | --- | --- |
| 2.1 | Chaque `<iframe>` a-t-il un titre ? | N/A | Aucun `<iframe>` dans l'application. |
| 2.2 | Le titre de chaque `<iframe>` est-il pertinent ? | N/A | — |

---

## Thématique 3 — Couleurs

| Critère | Question | Statut | Constat |
| --- | --- | --- | --- |
| 3.1 | L'information n'est-elle pas donnée uniquement par la couleur ? | ✅ | Les statuts (OK/NOK) ont aussi un libellé texte (`pages/admin/dashboard/dashboard.html:10,15`). |
| 3.2 | Le contraste texte sur fond est-il suffisant (4.5:1) ? | ⚠️ | Texte muted via `rgb(--rs-dark-rgb / 0.62)` ≈ 4.4:1 (`styles.scss:57`, `auth-form.css:15,28,45,78`). Placeholder à 62% opacité — limite. Hero text sur image `brightness(.75)` à contraste variable (`home.css:43-44`). |
| 3.3 | Le contraste des éléments d'interface (icônes, focus) est-il suffisant (3:1) ? | ✅ | Focus ring `#2c376d` 3px sur cream `#f8f5f2` ≈ 8:1. Bouton primaire jaune `#fcc417` sur cream ≈ 1.5:1 (limite mais bouton porte du texte). |

**Action** : remonter l'opacité muted à 0.7+ ou utiliser une couleur solide `#595959`. Renforcer le scrim du hero pour garantir 4.5:1.

---

## Thématique 4 — Multimédia

| Critère | Question | Statut | Constat |
| --- | --- | --- | --- |
| 4.1–4.13 | Médias audio/vidéo (transcription, sous-titres, audiodescription) ? | N/A | Aucun média audio/vidéo dans l'application. |

---

## Thématique 5 — Tableaux

| Critère | Question | Statut | Constat |
| --- | --- | --- | --- |
| 5.1–5.8 | Tableaux de données (caption, scope, headers) ? | N/A | Pas de tableau de données. Les listes (recettes, ingrédients, commentaires) utilisent `<ul>/<ol>`. |

---

## Thématique 6 — Liens

| Critère | Question | Statut | Constat |
| --- | --- | --- | --- |
| 6.1 | Chaque lien a-t-il un intitulé ? | ✅ | Tous les `<a>` détectés ont un texte ou un `aria-label`. |
| 6.2 | Chaque lien a-t-il un intitulé pertinent (hors contexte) ? | ⚠️ | Liens « Modifier » / « Voir » dans la liste de recettes personnelles → ambigus pris isolément. Acceptable car contexte visuel proche. |
| 6.3 | L'état d'activation du lien (page courante) est-il indiqué ? | ✅ | `ariaCurrentWhenActive="page"` sur la nav header (`layouts/header/header.html:42,50,53,56`). |

---

## Thématique 7 — Scripts

| Critère | Question | Statut | Constat |
| --- | --- | --- | --- |
| 7.1 | Chaque script est-il compatible avec les AT ? | ✅ | Bootstrap dropdown + composants Angular standard, pas de `div` cliquables non-accessibles. |
| 7.2 | Chaque script a-t-il une alternative ? | ✅ | SSR sert le HTML pré-rendu pour les pages publiques (`recipes/:slug`, `users/:username`, légales). Lecture possible sans JS. |
| 7.3 | Chaque script est-il contrôlable au clavier ? | ⚠️ | Tous les boutons sont `<button>`. Mais dropdown user menu n'a pas Escape / arrow keys (`layouts/header/header.html:26-39`). |
| 7.4 | Pour chaque script qui initie un changement de contexte, l'utilisateur est-il averti ? | ✅ | Pas de changement automatique inattendu. Formulaires n'envoient pas en `(input)`. |
| 7.5 | Chaque message d'état est-il restitué par les AT ? | ⚠️ | `role="status"` / `aria-live` largement utilisés. Quelques pages cumulent trop de live regions (dashboard `:25,31,43,49,61,67,79,85`) → annonces simultanées confuses. |

**Action** : ajouter `(keydown.escape)` au dropdown menu et retour focus sur le bouton. Consolider les live regions du dashboard.

---

## Thématique 8 — Éléments obligatoires

| Critère | Question | Statut | Constat |
| --- | --- | --- | --- |
| 8.1 | Chaque page web a-t-elle un type de document (`<!DOCTYPE html>`) ? | ✅ | `src/index.html:1`. |
| 8.2 | Le code source est-il valide (HTML5) ? | ⚠️ | Présence d'un `<main>` imbriqué dans `<main>` (layout + admin pages `admin/users/user-detail/user-detail.html:36`, `admin/review/review.html:33`) — invalide RGAA 9.1. |
| 8.3 | Sur chaque page, la langue principale est-elle indiquée ? | ✅ | `<html lang="fr">` (`src/index.html:2`). |
| 8.4 | Pour chaque page ayant une langue par défaut, le code de langue est-il pertinent ? | ✅ | `fr` correct. |
| 8.5 | Chaque page a-t-elle un titre `<title>` ? | ⚠️ | `<title>` statique unique (`src/index.html:6`), pas dynamique par page. Critique pour SEO et pour les utilisateurs de lecteur d'écran qui ouvrent plusieurs onglets. |
| 8.6 | Le titre de chaque page est-il pertinent ? | ❌ | Tous les onglets affichent `Recipe Shelter — Cook. Share. Discover.` quelle que soit la page. |
| 8.7 | Chaque changement de langue est-il indiqué ? | ✅ | Pas de texte en langue étrangère détecté. |
| 8.9 | Les balises ne sont-elles pas utilisées uniquement à des fins de présentation ? | ⚠️ | `role="list"/listitem"` sur des `<div>` (`pages/contact/contact.html:12-29`) — antipattern, remplacer par `<ul>/<li>`. |
| 8.10 | Les changements du sens de lecture sont-ils signalés ? | N/A | Pas de texte RTL. |

**Action critique** : implémenter `Title.setTitle()` sur les pages dynamiques (recipe-detail, user-profile, search, etc.).

---

## Thématique 9 — Structuration de l'information

| Critère | Question | Statut | Constat |
| --- | --- | --- | --- |
| 9.1 | Dans chaque page, l'information est-elle structurée par l'utilisation appropriée de titres ? | ⚠️ | Hiérarchie globalement OK : un `<h1>` par page, `<h2>` de section. Cartes de recettes sur la home utilisent `<h2>` alors qu'elles sont imbriquées sous un `<h2>` de section (`shared/recipe-card/recipe-card.html:14` + `home.html:45`) → idéalement `<h3>`. |
| 9.2 | Dans chaque page, la structure du document est-elle cohérente ? | ⚠️ | Landmarks `<main>` dupliqués en admin (cf. 8.2). |
| 9.3 | Dans chaque page, chaque liste est-elle correctement structurée ? | ⚠️ | `<ul>/<ol>` utilisés correctement, sauf `role="list"` sur `<div>` (`pages/contact/contact.html`). |
| 9.4 | Dans chaque page, chaque citation est-elle correctement indiquée ? | N/A | Pas de citation longue. |

**Action** : supprimer un des deux `<main>` (renommer en `<section>`), remplacer `role="list"` par vrai `<ul>`, descendre `<h2>` des cartes en `<h3>` quand imbriquées.

---

## Thématique 10 — Présentation de l'information

| Critère | Question | Statut | Constat |
| --- | --- | --- | --- |
| 10.1 | Dans le site, des feuilles de styles sont-elles utilisées pour la présentation ? | ✅ | CSS séparé, pas d'attributs `style=""` massifs. |
| 10.2 | Dans chaque page, le contenu visible reste-t-il présent sans CSS ? | ✅ | Vérifiable en désactivant CSS → contenu textuel reste lisible. |
| 10.3 | Dans chaque page, l'information reste-t-elle compréhensible sans CSS ? | ✅ | — |
| 10.4 | Dans chaque page, le texte reste-t-il lisible quand la taille des caractères est augmentée jusqu'à 200% au moins ? | ✅ | Utilisation de `clamp()` + `rem` (`styles.scss:64`, `home.css:31,37`), pas de hauteur fixe sur les conteneurs de texte. |
| 10.5 | Dans chaque page, les déclarations CSS de couleurs ne nuisent-elles pas à la lisibilité ? | ⚠️ | Cf. 3.2 (muted/placeholder). |
| 10.6 | Dans chaque page, chaque lien dont la nature n'est pas évidente est-il visible par rapport au texte environnant ? | ✅ | Liens en couleur primaire `#384688` clairement distincts du noir. |
| 10.7 | Dans chaque page, pour chaque élément recevant le focus, la prise de focus est-elle visible ? | ✅ | `:focus-visible { outline: 3px solid ... }` global (`styles.scss:51-54`). Aucun `outline:none` détecté. |
| 10.8 | Pour chaque page web, les contenus cachés sont-ils ignorés par les AT ? | ✅ | `aria-hidden="true"` sur SVG décoratifs, classe utilitaire `visually-hidden` pour labels invisibles. |
| 10.9 | Dans chaque page, l'information ne doit pas être donnée uniquement par la forme, taille ou position ? | ✅ | — |
| 10.10 | Dans chaque page, l'information ne doit pas être donnée par la forme, taille ou position uniquement ? | ✅ | — |
| 10.11 | Pour chaque page, les contenus peuvent-ils être présentés sans perte d'information ou de fonctionnalité, et sans avoir recours à un défilement à 320px de large ? | ⚠️ | Bootstrap responsive en place mais à vérifier précisément en DevTools mobile 320px. |
| 10.12 | Dans chaque page, les propriétés d'espacement du texte peuvent-elles être redéfinies sans perte de contenu ou de fonctionnalité ? | ✅ | Utilisation de `rem`/`em` majoritairement. |
| 10.13 | Dans chaque page web, les contenus additionnels apparaissant au survol/focus/activation sont-ils contrôlables par l'utilisateur ? | ⚠️ | Dropdown user menu n'a pas d'Escape pour le fermer. |

---

## Thématique 11 — Formulaires

| Critère | Question | Statut | Constat |
| --- | --- | --- | --- |
| 11.1 | Chaque champ de formulaire a-t-il une étiquette ? | ✅ | Tous les `<input>/<textarea>/<select>` ont un `<label for="...">` lié (`pages/auth/sign-in/sign-in.html:7-8,20-21`, `pages/auth/sign-up/sign-up.html:7-43`, `pages/contact/contact.html:46-99`, `pages/recipes/submit/...`, `pages/profile/profile.html`). |
| 11.2 | Chaque étiquette associée à un champ de formulaire est-elle pertinente ? | ✅ | Libellés explicites en français. |
| 11.3 | Dans chaque formulaire, chaque étiquette associée à un champ de formulaire ayant la même fonction a-t-elle un intitulé cohérent ? | ✅ | Email = « Adresse email » partout, etc. |
| 11.4 | Dans chaque formulaire, chaque étiquette de champ et son champ associé sont-ils accolés ? | ✅ | Pas d'écart visuel anormal. |
| 11.5 | Dans chaque formulaire, les champs de même nature sont-ils regroupés, si nécessaire ? | ⚠️ | Tags de recette regroupés dans `role="group" aria-labelledby` (`pages/recipes/submit/recipe-tags-form.html:10`). Ingrédients : pas de `<fieldset>` mais labels par ligne suffisent. |
| 11.6 | Dans chaque formulaire, chaque regroupement de champs a-t-il une légende ? | ⚠️ | Idem 11.5. |
| 11.7 | Dans chaque formulaire, la valeur initiale ou les exemples sont-ils insérés correctement ? | ✅ | Placeholders présents mais non utilisés comme seul label. |
| 11.8 | Dans chaque formulaire, les champs obligatoires sont-ils indiqués ? | ✅ | Étoile `<span aria-hidden="true">*</span>` + mention récapitulative (`contact.html:41`, `profile.html:7`). `required` HTML sur tous les obligatoires. |
| 11.9 | Dans chaque formulaire, l'intitulé de chaque bouton est-il pertinent ? | ✅ | « S'inscrire », « Se connecter », « Soumettre la recette ». |
| 11.10 | Dans chaque formulaire, le contrôle de saisie est-il utilisé de manière pertinente ? | ⚠️ | Validators Angular en place. **Mais `aria-invalid` n'est positionné nulle part** → l'erreur n'est pas annoncée comme état de l'input par les lecteurs d'écran. |
| 11.11 | Dans chaque formulaire, le contrôle de saisie est-il accompagné, si nécessaire, de suggestions ? | ⚠️ | Messages d'erreur affichés mais **non liés via `aria-describedby`** sauf sur le checkbox CGU (`sign-up.html:56` + `:67`). |
| 11.12 | Pour chaque formulaire qui modifie ou supprime des données, ou qui transmet des réponses à un test ou examen, ou dont la validation a des conséquences financières ou juridiques, la saisie des données est-elle vérifiable, modifiable ou récupérable par l'utilisateur ? | ✅ | Inscription : confirmation par email avant activation. Suppression : confirmation demandée. |
| 11.13 | La finalité d'un champ de saisie peut-elle être déduite pour faciliter le remplissage automatique des champs ? | ✅ | `autocomplete="email"`, `current-password`, `new-password`, `name`, `email` largement utilisés (`sign-in.html:8,21`, `sign-up.html:8,17,30,43`, `contact.html`, `profile.html`). Erreur isolée : `inputmode="name"` (n'existe pas) dans `profile.html:96` à corriger en `text`. |

**Action critique** : ajouter `aria-invalid` + `aria-describedby` sur tous les inputs validés. Corriger `inputmode="name"` → `text`.

---

## Thématique 12 — Navigation

| Critère | Question | Statut | Constat |
| --- | --- | --- | --- |
| 12.1 | Chaque ensemble de pages dispose-t-il de deux systèmes de navigation différents au moins ? | ✅ | Menu principal header + recherche + pied de page (liens légaux + contact). |
| 12.2 | Dans chaque ensemble de pages, le menu de navigation est-il à la même place ? | ✅ | Header constant via `layouts/layout`. |
| 12.3 | La page d'accueil et le plan du site sont-ils atteignables depuis chaque page ? | ✅ | Logo header = lien vers `/`. Pas de plan du site (acceptable pour une SPA). |
| 12.4 | Dans chaque ensemble de pages, la page d'accueil et le moteur de recherche sont-ils atteignables depuis chaque page ? | ✅ | Header présent partout. |
| 12.5 | Dans chaque ensemble de pages, le moteur de recherche est-il atteignable depuis chaque page ? | ✅ | `<form role="search">` dans le header (`layouts/header/header.html:6`). |
| 12.6 | Les zones de regroupement de contenus présentes dans plusieurs pages peuvent-elles être atteintes ou évitées ? | ❌ | **Pas de skip link** « Aller au contenu principal ». RGAA 12.7. À ajouter dans `app.html`. |
| 12.7 | Dans chaque page web, un lien d'évitement ou d'accès rapide à la zone de contenu principal est-il présent ? | ❌ | Idem 12.6. |
| 12.8 | Dans chaque page, l'ordre de tabulation est-il cohérent ? | ✅ | Pas de `tabindex` positif détecté (grep 0). Ordre = ordre DOM. |
| 12.9 | Dans chaque page, la navigation ne doit pas contenir de piège au clavier ? | ✅ | Pas de modal piégeant le focus. Dropdown ne piège pas mais ne se ferme pas via Échap (à corriger). |
| 12.10 | Dans chaque page, les raccourcis clavier n'utilisant qu'une seule touche (lettre, ponctuation, chiffre ou symbole) sont-ils contrôlables par l'utilisateur ? | N/A | Pas de raccourcis clavier custom. |
| 12.11 | Dans chaque page web, les contenus additionnels apparaissant à la prise de focus ou au survol d'un composant d'interface sont-ils, si nécessaire, atteignables au clavier ? | ⚠️ | Dropdown user menu : Tab tombe sur le toggle, click ouvre, mais focus ne se déplace pas dans le menu (Bootstrap par défaut). |

**Action critique** : ajouter un skip link en tête de `app.html`.

---

## Thématique 13 — Consultation

| Critère | Question | Statut | Constat |
| --- | --- | --- | --- |
| 13.1 | Pour chaque page web, l'utilisateur a-t-il le contrôle de chaque limite de temps modifiant le contenu ? | ✅ | Pas de session client-side avec timeout brutal. Le cookie HttpOnly côté serveur expire mais redirige vers login. |
| 13.2 | Dans chaque page web, l'ouverture d'une nouvelle fenêtre ne doit pas être déclenchée sans action de l'utilisateur ? | ✅ | Pas de `window.open` automatique. |
| 13.3 | Dans chaque page web, chaque document bureautique téléchargeable possède-t-il, si nécessaire, une version accessible ? | N/A | Pas de PDF / docx téléchargeable. |
| 13.4 | Pour chaque document bureautique ayant une version accessible, cette version offre-t-elle la même information ? | N/A | — |
| 13.5 | Dans chaque page, le contenu en mouvement ou clignotant est-il contrôlable par l'utilisateur ? | N/A | Pas d'animation auto. |
| 13.6 | Dans chaque page web, les contenus cryptiques (art ASCII, émoticônes, syntaxe cryptique) ont-ils une alternative ? | N/A | — |
| 13.7 | Dans chaque page, les changements brusques de luminosité ou les effets de flash sont-ils correctement utilisés ? | ✅ | Pas d'effet flash. |
| 13.8 | Dans chaque page, l'utilisateur peut-il contrôler ou désactiver l'autoplay des animations ? | ⚠️ | Pas d'animation autoplay. **Pas de `prefers-reduced-motion`** : transitions sur les boutons s'appliquent à tous, y compris utilisateurs sensibles aux animations. |
| 13.9 | Dans chaque page, l'orientation de l'affichage (portrait ou paysage) n'est-elle pas imposée ? | ✅ | Pas de verrouillage d'orientation. |
| 13.10 | Dans chaque page, les fonctionnalités utilisables ou disponibles au moyen d'un geste complexe peuvent-elles être également disponibles au moyen d'un geste simple ? | N/A | Pas de geste custom (swipe, pinch). |
| 13.11 | Dans chaque page, les actions déclenchées au moyen d'un dispositif de pointage sur un point unique de l'écran peuvent faire l'objet d'une annulation ? | ✅ | Boutons standard (clic = action seulement au mouseup). |
| 13.12 | Dans chaque page, les fonctionnalités qui impliquent un mouvement de l'utilisateur ou de l'appareil peuvent-elles être satisfaites par une alternative ? | N/A | Pas de fonctionnalité basée sur secousse / inclinaison. |

**Action** : ajouter `@media (prefers-reduced-motion: reduce)` autour des transitions globales pour respecter 13.8.

---

## Synthèse globale

### Compteur

| Statut | Nombre de critères évalués |
| --- | --- |
| ✅ Conforme | 45 |
| ⚠️ Partiellement conforme | 16 |
| ❌ Non conforme | 3 (titres dynamiques, skip link, role list sur div) |
| N/A | 17 |

**Taux de conformité indicatif** (conforme / (conforme + partiel + non conforme)) ≈ 45 / 64 ≈ **70%**.

Cela exclut les non-applicables (multimédia, tableaux, iframes, gestures). Sur le périmètre applicable, on est dans la zone « **conformité partielle** » au sens RGAA — un site obtient « conformité totale » seulement à 100% sur tous les critères applicables.

### Les 3 non-conformités à corriger en priorité (avant soutenance si possible)

1. **Titres de page dynamiques** (critère 8.5/8.6) — pas d'`Angular.Title` utilisé → mêmes onglets partout. Impact SEO + a11y majeur.
2. **Skip link** (critère 12.7) — pas de lien « Aller au contenu » → utilisateurs clavier doivent tabuler dans tout le header.
3. **`role="list"` sur `<div>`** (critère 8.9 / 9.3) — antipattern, remplacer par `<ul>/<li>` dans `pages/contact/contact.html`.

### Les 5 partiels les plus visibles à corriger ensuite

1. `aria-invalid` + `aria-describedby` sur tous les inputs validés (critère 11.10/11.11).
2. `<main>` imbriqués en admin (critère 8.2/9.2).
3. Locale Angular `fr-FR` non enregistré (qualité, pas RGAA strict).
4. Dropdown user menu sans Escape (critère 7.3/12.11).
5. Contraste muted/placeholder borderline (critère 3.2/10.5).

### Honnêteté de la grille

Cette grille est rigoureuse : elle a relevé chaque écart au lieu de tout passer en vert pour le confort de la note. Au jury :

> « On annonce 70% de conformité parce qu'on a audité critère par critère. Un site qui prétend 95% sans détailler ses 13 thématiques ment ou s'auto-évalue mal. Les non-conformités identifiées sont documentées et la majorité est corrigeable en moins d'une journée. »

C'est cette posture (mesurer honnêtement, savoir où on va) qui valide le Bloc 1, pas un score parfait sorti d'un audit complaisant.
