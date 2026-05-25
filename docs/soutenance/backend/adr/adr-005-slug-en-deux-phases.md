# ADR-005 — Slug en deux phases (brouillon puis public)

## Statut

Accepté · 2026-05-25

## Contexte

Les recettes sont identifiées dans les URLs publiques par un **slug**
unique (ex : `/recipes/tarte-aux-pommes-de-mamie-jeannette`). La colonne
`Slug VARCHAR(255) UNIQUE` est obligatoire dans la table `Recipes`.

Or, le cycle de vie d'une recette est :

```
draft → pending → published | rejected → archived
```

Une recette peut rester en brouillon (`draft`) pendant des semaines,
être éditée plusieurs fois, et **ne jamais être publiée**.

Le problème : faut-il calculer le slug définitif dès la création du
brouillon, ou attendre la publication ?

## Décision

**Deux phases** :

1. À la création (`POST /api/v1/recipes`), génération d'un **slug
   technique temporaire** sous la forme `draft-{userId}-{randomHash}`.
   Garantit l'unicité sans interférer avec l'espace de noms public.

2. À la soumission (`POST /api/v1/recipes/:id/submit`), génération du
   **slug public définitif** dérivé du titre :
   `tarte-aux-pommes-de-mamie-jeannette`. Vérification d'unicité en
   base ; en cas de collision, ajout d'un suffixe incrémental
   (`-2`, `-3`).

Implémentation : `src/services/recipes/recipe-slug.service.ts`
(`createDraftSlug` et `createPublicSlug`).

## Alternatives considérées

### Slug définitif dès la création

- **Pour** : un seul slug pendant toute la vie de la recette, URL
  stable dès le brouillon.
- **Contre** : "réserve" un slug public alors que la recette pourrait
  ne jamais être publiée. Si l'auteur tape un titre puis abandonne,
  le slug est gaspillé.
- **Contre** : si l'auteur renomme la recette pendant l'édition, soit
  on garde l'ancien slug (URL incohérente avec le titre final), soit
  on régénère (et alors c'est exactement le mécanisme actuel).

### Pas de slug, ID numérique en URL

- **Pour** : trivial, pas de gestion d'unicité.
- **Contre** : mauvais pour le SEO, mauvaise UX (URL non parlante),
  exposition de l'ID auto-incrément (révélation indirecte du volume
  de la BDD).

### UUID comme slug

- **Pour** : pas de risque de collision.
- **Contre** : URLs cryptiques, SEO inexistant.

## Conséquences

### Positives

- **L'espace de noms public reste propre** : un slug public n'est
  posé que pour des recettes effectivement soumises à modération.
- **SEO-friendly** : les URLs de recettes publiées sont lisibles et
  contiennent des mots-clés.
- **Pas de migration de slug en cours de vie** : une fois publié, le
  slug ne change plus (URL stable, partageable, indexable).
- **Robustesse aux abandons** : un auteur qui crée 50 brouillons
  jamais soumis ne pollue pas l'espace de noms public.

### Négatives

- **Deux mécanismes de génération à maintenir**. Mitigation : le
  service `RecipeSlugService` les encapsule dans une seule classe.
- **L'URL change au moment de la soumission** : le brouillon est
  accessible par son slug `draft-X-Y`, puis devient
  `tarte-aux-pommes-...` une fois publié. Pas de redirection 301 mise
  en place (le slug `draft-` n'est de toute façon visible que par
  l'auteur).

### À surveiller

- Risque de collision lors de la génération du slug public si deux
  recettes ont exactement le même titre normalisé. Le `existsBySlug`
  + suffixe incrémental gère ce cas. Vérifier que le `LIKE` ne crée
  pas de hot spot SQL si la table grossit (index unique sur `Slug`
  déjà présent : `UNIQUE KEY recipes_slug_UK`).
