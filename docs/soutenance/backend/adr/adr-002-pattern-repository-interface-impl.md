# ADR-002 — Pattern Repository : interface + implémentation MySQL

## Statut

Accepté · 2026-05-25

## Contexte

Le Bloc 2 demande d'utiliser la POO et de manipuler MySQL "from scratch"
(sans ORM). Il faut choisir comment exposer l'accès aux données aux
services métier.

Trois patterns possibles :

1. Appel direct au pool MySQL depuis les services (`pool.query(...)`).
2. Une classe `Repository` par entité qui encapsule les requêtes SQL,
   appelée directement par les services.
3. **Une interface `Repository` + une implémentation `RepositoryMysql`**,
   les services dépendant de l'interface.

## Décision

Adopter le pattern **(3)** : pour chaque domaine, un fichier
`*.repository.interface.ts` définit le contrat (méthodes pures), et un
fichier `*.repository.mysql.ts` l'implémente avec `mysql2`. Les services
reçoivent l'interface par injection de constructeur.

Exemple structurel sur le domaine `recipes` :

```
src/repositories/recipes/
  recipe.repository.interface.ts   ← contrat
  recipe.repository.mysql.ts       ← impl
  recipe.mapper.ts                 ← row SQL → objet métier
  recipe.types.ts                  ← types domaine
```

## Alternatives considérées

### Appel direct au pool depuis les services

- **Pour** : moins de fichiers, moins d'indirection.
- **Contre** : couplage fort service ↔ MySQL. Impossible de tester les
  services sans démarrer MySQL. Difficile à migrer si on change de SGBD.
- **Contre** : la logique métier (validation, autorisations, machine
  à états) est mélangée avec la construction de requêtes SQL.

### Classe Repository sans interface

- **Pour** : encapsule le SQL.
- **Contre** : les services dépendent de l'implémentation concrète. Les
  tests unitaires doivent quand même mocker la classe entière, ce qui
  reste fragile.

## Conséquences

### Positives

- **Testabilité** : les ~30 fichiers de tests dans `tests/services/`
  branchent un repository en mémoire et n'ont pas besoin de MySQL.
  Tests rapides, déterministes, exécutables en CI sans infra.
- **Séparation des préoccupations** : le service ne sait rien du SQL.
  Le repository ne sait rien de la logique métier.
- **Évolutivité** : un futur `RecipeRepositoryPostgres` pourrait
  remplacer l'impl MySQL sans toucher aux services.
- **Pédagogique** : illustre concrètement le principe SOLID
  d'inversion de dépendances (le "D").

### Négatives

- Plus de fichiers à créer (~3 par domaine au lieu d'1).
- Double déclaration : la signature apparaît dans l'interface ET dans
  l'impl. Mitigation : TypeScript vérifie la conformité au compile.

### À surveiller

- Risque de "anemic interface" : si l'interface ne fait que copier
  la classe, sans réelle abstraction métier, le pattern devient
  bureaucratique. Vérifier régulièrement que les noms de méthodes
  expriment l'**intention métier** (`findPublishedBySlug`) et non
  le **détail technique** (`selectFromRecipesJoinWhere`).
