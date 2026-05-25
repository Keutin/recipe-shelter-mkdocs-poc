# ADR-001 — Express comme couche HTTP

## Statut

Accepté · 2026-05-25

## Contexte

Le projet doit exposer une API HTTP REST permettant au frontend Angular
de consommer les ressources (recettes, utilisateurs, commentaires,
favoris, modération…). Le cahier des charges du Bloc 2 stipule de
développer le back-end "sans frameworks ni librairies prédéfinies",
mais autorise Node.js comme runtime.

Trois options techniques se présentent pour la couche HTTP :

1. `node:http` natif (zéro dépendance)
2. **Express** (micro-routeur, ~3000 lignes, dépendance unique)
3. NestJS / Fastify avec plugins (framework full-stack ou semi-full-stack)

## Décision

Utiliser **Express 5** comme couche HTTP, avec un découpage manuel en
controllers + services + repositories. Aucun framework full-stack, aucun
ORM, aucune génération de code.

## Alternatives considérées

### `node:http` natif

- **Pour** : interprétation littérale du cahier "sans librairie".
- **Contre** : ~150 lignes de boilerplate à réécrire (parsing URL,
  dispatch par méthode, parsing du body JSON, chaînage de middlewares,
  gestion des erreurs async). Aucune valeur pédagogique ajoutée — ce
  code ne fait que rejouer ce qu'Express fait déjà depuis 15 ans.
- **Risque** : bugs subtils sur la gestion des erreurs async ou le
  parsing des bodies.

### NestJS

- **Pour** : DI container, décorateurs, génération CLI, intégration
  testing.
- **Contre** : opinionated, génère du code, masque les flux. Trahit
  l'esprit du cahier "from scratch sur les contrôleurs, modèles, vues".

### Fastify

- **Pour** : plus rapide qu'Express, validation de schémas intégrée.
- **Contre** : ses plugins (`@fastify/jwt`, `@fastify/cors`) sont plus
  opinionated, et la validation de schéma intégrée (TypeBox / Ajv) aurait
  retiré l'exercice pédagogique des DTOs écrits à la main.

## Conséquences

### Positives

- L'architecture (couches, DI, pattern Repository) est entièrement
  conçue par l'auteur, pas imposée par le framework.
- Express est minimal : il ne fait que router et chaîner des middlewares.
  Tout le reste — DTOs, validation, auth, autorisations — est manuel.
- Compatibilité maximale avec l'écosystème Node (le projet déclare
  `"type": "module"` et utilise ESM natif).

### Négatives

- Quelqu'un de strict sur la lecture du cahier peut considérer Express
  comme "une librairie" et reprocher son utilisation. Mitigation : la
  narration de défense (`00-narration-from-scratch.md`) explique
  pourquoi Express est une brique d'infrastructure standard, pas un
  framework full-stack.

### À surveiller

- Express 5 est récent (sortie stable 2024). Quelques middlewares de
  l'écosystème ne sont pas encore migrés. Ici aucun n'a posé problème.
