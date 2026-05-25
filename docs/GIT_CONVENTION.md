# Recipe Shelter - Git Convention

Ce document definit les conventions Git communes aux repositories du projet Recipe Shelter.

---

## Organisation des repositories

### Repositories principaux

- `recipe-shelter-frontend` : application Angular.
- `recipe-shelter-backend` : API Node.js/Express.

---

## Branches

### Branches principales

- `main` : branche stable.
- `develop` : branche d'integration, optionnelle selon le rythme du projet.

### Branches de travail

Format recommande :

- `feature/<scope>-<short-name>`
- `fix/<scope>-<short-name>`
- `chore/<short-name>`
- `docs/<short-name>`
- `test/<short-name>`

Exemples :

- `feature/auth-register`
- `feature/recipes-search`
- `fix/login-redirect`
- `chore/eslint-config`
- `docs/backend-readme`

---

## Commits

Le projet suit les Conventional Commits.

Format :

```text
type(scope): message
```

Types conseilles :

- `feat` : nouvelle fonctionnalite.
- `fix` : correction de bug.
- `docs` : documentation.
- `refactor` : refactorisation sans changement fonctionnel.
- `test` : ajout ou modification de tests.
- `chore` : configuration, dependances, CI.

Exemples :

- `feat(auth): add register endpoint`
- `fix(recipes): handle empty tags query`
- `docs(readme): add setup instructions`
- `test(comments): cover dto validation`
- `chore(ci): add github workflow`
