# Défendre le Bloc 1 en soutenance

Notes pour Arthur : comment parler de l'audit, comment encaisser les questions, comment retourner les défauts en preuves de méthode.

## Posture générale

Le jury Bloc 1 cherche **trois signaux** :

1. **Tu connais les standards** (RGAA / WCAG / Lighthouse), pas par cœur, mais tu sais où chercher.
2. **Tu as mesuré honnêtement** ton site (pas auto-évaluation complaisante).
3. **Tu sais ce qui reste à faire** (roadmap concrète, pas vœux pieux).

Un score Lighthouse à 99 sans capacité à discuter les écarts vaut **moins** qu'un score à 82 défendu avec lucidité. Le jury préfère la rigueur à la perfection.

## Le pitch d'ouverture (45 secondes)

> « Pour le Bloc 1, j'ai mené deux audits parallèles. Le premier, Lighthouse : performance, accessibilité, bonnes pratiques, SEO sur cinq pages clés (home, liste, fiche, sign-in, admin), en mobile et desktop, en production. Le deuxième, plus exigeant : une grille RGAA 4.1 critère par critère, 64 critères applicables sur les 13 thématiques officielles.
>
> Sur la grille RGAA, je suis aujourd'hui à environ 70 % de conformité : 45 critères conformes, 16 partiels, 3 non conformes. Je vais vous présenter d'où viennent les écarts et ce que j'ai prévu de corriger. »

Ce pitch :
- **Annonce le périmètre** (5 pages, 2 modes, 64 critères).
- **Annonce le résultat** sans le maquiller.
- **Promet la suite** (les écarts + le plan).

## Les 3 non-conformités à expliquer

Si on a corrigé avant la soutenance (recommandé), expliquer la séquence :

> « Pendant l'audit, j'ai identifié trois non-conformités : pas de skip link, pas de titre de page dynamique, et l'utilisation détournée de `role="list"` sur des div. Les trois sont corrigées sur la branche `docs/bloc1-audit`. Le skip link prend dix minutes mais débloque le critère 12.7. Les titres dynamiques s'appuient sur le service `Title` d'Angular — c'est trois lignes par page. Le `role="list"` était un mauvais réflexe : un vrai `<ul>` fait la même chose en HTML natif. »

Si on **n'a pas corrigé** : honnêteté → « identifié, documenté, planifié dans P1 du plan de remédiation ».

## Les questions probables et leurs réponses

### Q1. « Pourquoi pas 100% de conformité ? »

> « Parce que 100% sur 64 critères demande un investissement disproportionné par rapport à un projet de certification. J'ai priorisé : les non-conformités critiques sont corrigées, les partiels les plus visibles aussi. Les chantiers restants — comme le dropdown user menu avec gestion clavier complète — demandent une refonte du composant qui sort du périmètre. C'est documenté dans le plan P3. »

### Q2. « Vous avez utilisé quel référentiel ? »

> « RGAA 4.1.2 — la version 2023, hébergée sur accessibilite.numerique.gouv.fr. Je m'appuie aussi sur WCAG 2.1 niveau AA pour les zones grises, parce que WCAG est plus précis sur certains critères techniques. Lighthouse utilise WCAG en interne, donc les deux audits sont compatibles. »

### Q3. « Vous avez testé avec un lecteur d'écran ? »

Honnête :

> « Une passe rapide avec NVDA sur la home et une fiche recette. Pas un audit complet. Ce qui en est sorti : les annonces des `aria-live` du dashboard se chevauchent, ce qui valide le finding de l'audit statique. Une passe NVDA complète est dans la P3 du plan. »

Ou si rien fait :

> « Pas de passe lecteur d'écran à ce stade. Je le note dans le plan de remédiation. L'audit s'est fait sur le code et avec l'extension axe DevTools, ce qui couvre les critères automatisables. »

### Q4. « Comment vous avez mesuré le contraste ? »

> « Deux méthodes. Pour les couleurs définies en tokens CSS — `#384688` sur cream `#f8f5f2` — calcul manuel via la formule WCAG. Pour les textes avec opacité — par exemple `rgb(--rs-dark-rgb / 0.62)` — j'ai converti en RGB final puis calculé. Pour le hero où le texte blanc est superposé à une image filtrée `brightness(.75)`, je note que le contraste est runtime-variable et ne peut pas être garanti par CSS seul — c'est dans les findings. La correction proposée est un scrim solide sombre. »

### Q5. « Lighthouse vous donne 82 en performance, pourquoi pas plus ? »

> « Trois raisons identifiées dans l'audit. Premièrement, je n'utilise pas `NgOptimizedImage` — toutes mes balises `<img>` sont des `<img>` natives avec `width`/`height`/`loading`. C'est suffisant pour éviter le CLS et la priorité LCP, mais ça ne génère pas de `srcset` automatique. Deuxièmement, mes composants Angular ne sont pas en `OnPush` — avec les signals que j'utilise, c'est gratuit à activer, c'est dans P2. Troisièmement, je n'ai pas de `@defer` sur les zones lourdes comme les commentaires, qui pourraient se charger après le viewport. Les trois sont planifiées. »

### Q6. « Vous parlez d'OpenGraph manquant — c'est important ? »

> « Pour un site de partage de recettes, oui. Quand on partage un lien de recette sur WhatsApp ou Slack, sans `og:image` ni `og:title`, on voit "Recipe Shelter" tout court. Avec les balises OG dynamiques par recette, on voit l'image de couverture et le titre — beaucoup plus engageant. Lighthouse le détecte aussi dans la catégorie SEO. C'est en P3 parce que ça demande de toucher chaque page de détail. »

### Q7. « Vous avez parlé de RGPD — où est la bannière cookies ? »

> « Volontairement absente. J'ai un seul cookie côté navigateur, le cookie de session auth en HttpOnly, qui rentre dans la catégorie "strictement nécessaire" de la CNIL — donc pas de consentement obligatoire. La politique de confidentialité actuelle mentionne malheureusement une "mesure d'audience" qui n'existe pas vraiment en production : c'est une incohérence que j'ai identifiée pendant l'audit. La correction est soit de retirer cette mention, soit d'implémenter vraiment la mesure d'audience avec une bannière conforme. Je penche pour la première option. »

### Q8. « 70% de conformité, c'est suffisant pour parler de "site accessible" ? »

> « Non. La déclaration officielle de conformité distingue trois niveaux : non conforme (< 50%), partiellement conforme (50–95%), totalement conforme (100%). À 70% je suis dans "partiellement conforme". Pour pouvoir afficher "totalement conforme", il faudrait corriger toutes les non-conformités et la majorité des partiels, et idéalement faire valider l'audit par un auditeur certifié — ce qui sort du périmètre cert. »

### Q9. « C'est quoi un skip link ? »

> « Un lien invisible par défaut, qui apparaît au premier `Tab` en haut de page, et qui permet de sauter directement au contenu principal sans tabuler dans la navigation. Critère RGAA 12.7. Sans ça, un utilisateur clavier doit traverser tous les liens du header à chaque page — pénible. »

### Q10. « Pourquoi pas tout en Angular Material ou autre framework a11y-ready ? »

> « Trois raisons. D'abord, Angular Material n'est pas exempt de défauts a11y — il faut quand même les configurer correctement. Ensuite, Bootstrap 5 est plus léger en bundle pour ce que j'utilise — j'ai cherry-picked nav, dropdown, buttons, helpers — c'est documenté dans `styles.scss`. Enfin, écrire mes propres composants m'oblige à comprendre les patterns ARIA plutôt que les déléguer — c'est l'esprit du from-scratch que je défends. La contrepartie, c'est que je dois assumer certains défauts comme le dropdown sans Escape, et les corriger. »

## Pièges à éviter

### Ne pas confondre Lighthouse et RGAA

Lighthouse mesure une **partie** de WCAG (les critères automatisables). RGAA va plus loin et inclut des critères qui demandent une revue manuelle (cohérence des intitulés, pertinence des alternatives, etc.). Ne pas dire « j'ai 95 en accessibilité Lighthouse donc le site est conforme RGAA » — c'est faux et le jury le sait.

### Ne pas inventer

Si le jury demande « quel est le critère exact pour les titres de page ? » — répondre « 8.5 et 8.6 » si on le sait, ou « la thématique 8, éléments obligatoires, je vérifie le numéro exact » si on ne le sait pas. Mieux que d'inventer un numéro.

### Ne pas minimiser

> « Oh, le skip link, c'est pas grave, personne ne s'en sert. »

→ Faux. Les utilisateurs clavier — handicap moteur, RSI, utilisateurs avancés — l'utilisent quotidiennement. **Ne jamais minimiser un manquement RGAA devant un jury de certification.**

### Ne pas survendre la SSR

La SSR Angular **n'est pas** une garantie d'accessibilité. Elle améliore le SEO et la performance perçue (LCP). Mais le HTML rendu côté serveur peut très bien être inaccessible — c'est ce qu'on aurait sans audit.

### Ne pas oublier les images

L'attribut `alt` n'est **pas** une option. Une image décorative = `alt=""` (présent et vide). Une image porteuse = `alt="description courte"`. Une image absente = balise `<img>` sans `alt` = non conforme.

## Mantras Bloc 1

- « Mesurer avant d'optimiser. »
- « 70% honnête vaut mieux que 95% maquillé. »
- « Le skip link, ça coûte 10 minutes et ça gagne un critère. »
- « Lighthouse mesure ce qui est mesurable. Le reste, c'est de la lecture critique. »
- « L'accessibilité, c'est une roadmap, pas un livrable one-shot. »

## Lien avec les autres livrables

- Les questions ci-dessus recoupent **`_draft_jury/01-bloc1-frontend.md`** : ne pas dupliquer, harmoniser les réponses entre les deux fichiers.
- Le slide `Bloc 1 — Lighthouse + accessibilité` du **`_draft_slides/soutenance-slides.md`** doit afficher les scores Lighthouse remplis ET la ligne « 45 / 16 / 3 / 17 » de la grille RGAA.
- Si on cite les chiffres au jury, garder une **copie imprimée du rapport Lighthouse HTML** sous la main — preuve matérielle.
