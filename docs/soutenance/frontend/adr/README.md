# ADR — Architectural Decision Records (Frontend)

> Pendant frontend des ADRs backend (`_draft_adr/`). Cinq décisions
> structurantes de la stack Angular 21 de Recipe Shelter, défendues à
> la soutenance RNCP Bloc 3. Numérotation continue avec le backend :
> ADRs 006 à 010.

## Pourquoi des ADRs

Un ADR (*Architecture Decision Record*) capture **un** choix de
conception : son contexte, la décision prise, les alternatives
considérées, et les conséquences. Côté frontend, ces cinq documents
tracent pourquoi le projet est en Signals + Standalone plutôt qu'en
NgModules + RxJS, pourquoi en SSR sélectif plutôt qu'en CSR pur,
pourquoi en TypeScript strict de bout en bout, pourquoi sur Vitest
plutôt que sur Karma, et pourquoi sur Bootstrap partiel plutôt que
sur Tailwind ou Material. Quand un juré demande *« pourquoi Signals ? »*
ou *« pourquoi pas Tailwind ? »*, la réponse est un document écrit
daté, pas une improvisation orale.

Format inspiré de Michael Nygard (2011), adapté en français, identique
à celui des ADRs backend pour cohérence visuelle dans la doc et
facilité de lecture pour le jury.

## Liste des ADRs

| Fichier | Sujet | Statut |
|---|---|---|
| `adr-006-angular-signals-standalone.md` | Signals + composants standalone, zéro NgModule, service-as-store pour la session, RxJS conservé pour HTTP uniquement | Accepté |
| `adr-007-ssr-per-route.md` | SSR Angular avec mode de rendu choisi par route (`Server` pour les fiches recette et profils, `Client` pour auth et admin, `Prerender` en catch-all) | Accepté |
| `adr-008-typescript-strict.md` | `strict: true` + flags additionnels + `strictTemplates` Angular, ESLint en preset strict, `// @ts-ignore` interdit sans description | Accepté |
| `adr-009-vitest.md` | Vitest 4 + jsdom via le builder officiel `@angular/build:unit-test`, specs colocalisés, zéro Karma | Accepté |
| `adr-010-bootstrap-partial-scss.md` | Bootstrap 5 importé partiellement en SCSS (15 modules sur ~40), branding posé en CSS custom properties `--rs-*` | Accepté |

## Lien avec les ADRs backend

Le dossier `_draft_adr/` contient les ADRs **001 à 005**, qui couvrent
les choix structurants du backend (Bloc 2) :

- ADR-001 — Express comme couche HTTP
- ADR-002 — Pattern Repository (interface + impl MySQL)
- ADR-003 — JWT en cookie HttpOnly
- ADR-004 — Soft-delete + log de modération
- ADR-005 — Slug technique en brouillon, slug public à la publication

Ensemble, les dix ADRs (001-005 backend + 006-010 frontend) forment la
trace écrite des décisions structurantes des deux blocs techniques de
la certification. C'est l'angle « architecture défendable » de la
soutenance.

## Lien avec le reste de la doc

Ces ADRs ne vivent pas isolés. Ils sont référencés depuis :

- **Architecture backend** : `_draft_backend_docs/architecture.md`
  (le diagramme d'ensemble cite les ADRs backend par numéro).
- **Audit frontend Bloc 1** : `_draft_bloc1_audit/` (l'audit HTML/CSS/
  accessibilité s'appuie sur les choix d'ADR-010 pour expliquer la
  stratégie CSS).
- **Slides soutenance** : `_draft_slides/soutenance-slides.md` (les
  slides Bloc 3 renvoient à ADR-006, 007, 008, 009, 010).
- **Banque de questions jury** : `_draft_jury/` (chaque question
  technique du jury sur la stack front a une réponse pointant vers
  l'ADR correspondant).

## Walkthrough handoff (Quentin → Arthur)

L'intégration de ces ADRs dans le repo `documentation` se fait en six
étapes. Aucune ne demande de réflexion, juste de l'exécution propre.

### Pré-requis

- Repo `documentation` cloné localement et à jour.
- Branche courante = `main`, working tree propre (`git status` clean).
- Avoir lu les cinq ADRs une fois en entier (pas survol — lecture
  posée). Si une phrase ne te paraît pas vraie ou pas défendable,
  modifie-la avant de commiter. Un ADR doit refléter ta vraie pensée,
  pas ce que j'ai écrit en projection.

### Étape 1 — Créer la branche

```powershell
git checkout main
git pull
git checkout -b docs/frontend-adrs
```

### Étape 2 — Créer le dossier cible

Le dossier `soutenance/frontend/adr/` n'existe pas encore dans le repo
`documentation`. On le crée :

```powershell
New-Item -ItemType Directory -Force -Path soutenance\frontend\adr
```

### Étape 3 — Copier les fichiers

Copier les cinq ADRs + ce README depuis le draft local vers le repo
documentation :

```powershell
Copy-Item C:\DEV\ARTHUR\RECETTES\_draft_adr_frontend\adr-006-angular-signals-standalone.md soutenance\frontend\adr\
Copy-Item C:\DEV\ARTHUR\RECETTES\_draft_adr_frontend\adr-007-ssr-per-route.md            soutenance\frontend\adr\
Copy-Item C:\DEV\ARTHUR\RECETTES\_draft_adr_frontend\adr-008-typescript-strict.md        soutenance\frontend\adr\
Copy-Item C:\DEV\ARTHUR\RECETTES\_draft_adr_frontend\adr-009-vitest.md                   soutenance\frontend\adr\
Copy-Item C:\DEV\ARTHUR\RECETTES\_draft_adr_frontend\adr-010-bootstrap-partial-scss.md   soutenance\frontend\adr\
Copy-Item C:\DEV\ARTHUR\RECETTES\_draft_adr_frontend\README.md                           soutenance\frontend\adr\
```

Vérifie ensuite que les six fichiers sont bien là :

```powershell
Get-ChildItem soutenance\frontend\adr\
```

### Étape 4 — Commit

Convention `docs(<scope>)` du repo (cf. `GIT_CONVENTION.md`) :

```powershell
git add soutenance/frontend/adr/
git commit -m "docs(frontend): ajoute ADRs Bloc 3 (Signals, SSR, TS strict, Vitest, Bootstrap)"
```

### Étape 5 — Push et PR

```powershell
git push -u origin docs/frontend-adrs
```

Puis ouvrir une PR vers `main` depuis l'interface GitHub. Pas besoin
de demander une review externe : tu es seul mainteneur. Tu mergues
après relecture solo.

### Étape 6 — Exposer dans MkDocs (optionnel)

Si le site MkDocs doit refléter cette nouvelle section, mettre à jour
`mkdocs.yml` pour ajouter une entrée de navigation :

```yaml
nav:
  - Soutenance:
      - Frontend:
          - ADR: soutenance/frontend/adr/README.md
          - ADR-006 Signals: soutenance/frontend/adr/adr-006-angular-signals-standalone.md
          - ADR-007 SSR: soutenance/frontend/adr/adr-007-ssr-per-route.md
          - ADR-008 TS strict: soutenance/frontend/adr/adr-008-typescript-strict.md
          - ADR-009 Vitest: soutenance/frontend/adr/adr-009-vitest.md
          - ADR-010 Bootstrap: soutenance/frontend/adr/adr-010-bootstrap-partial-scss.md
```

Tester en local avec `mkdocs serve` avant de pousser.

## Note méthodologique

Tous les ADRs (frontend et backend) suivent la même structure :

1. **Statut** — Accepté / Considéré / Déprécié + date.
2. **Contexte** — la situation qui rend la décision nécessaire.
3. **Décision** — ce qui est choisi, en termes précis et activables.
4. **Alternatives considérées** — chaque option crédible avec ses
   pour / contre / risque. Pas d'hommes de paille.
5. **Conséquences** — Positives / Négatives / À surveiller.
6. **Références code** — chemins fichier + ligne dans le repo
   `frontend/` ou `backend/`, pour qu'on puisse vérifier que le doc
   colle au code.

Cette structure constante est délibérée : elle permet au jury de
naviguer rapidement entre les ADRs sans avoir à se réorienter à
chaque ouverture.

## Principes de défense

Trois réflexes utiles pour défendre ces ADRs à l'oral :

- **Le test des 30 secondes.** Tu dois pouvoir résumer chaque ADR en
  trente secondes : Contexte → Décision → une conséquence majeure.
  Si tu n'arrives pas à le faire sur un ADR, c'est qu'il faut
  reformuler le document — pas que tu maîtrises mal le sujet.
- **Justifier par une contrainte, pas par une préférence.** Réponse
  faible : *« j'ai préféré Vitest, je trouve ça plus moderne »*.
  Réponse forte : *« Karma est déprécié par l'équipe Angular depuis
  fin 2024, le builder officiel `@angular/build:unit-test` cible
  Vitest, donc partir sur Karma en 2026 c'est démarrer avec une dette
  technique »*. La contrainte (cert RNCP, cahier des charges, taille
  équipe, dépréciation upstream, budget bundle) est toujours
  l'ancrage de la décision.
- **Alternatives crédibles, pas hommes de paille.** Quand tu présentes
  les alternatives écartées, présente-les sous leur meilleur jour.
  Le jury doit sentir que tu as comparé loyalement, pas que tu as
  monté un strawman pour pousser ton choix. C'est précisément pour
  ça que les sections « Alternatives considérées » des cinq ADRs
  donnent à chaque option ses vrais points forts avant de lister
  ses limites.

Les ADRs ne sont pas un exercice de style. Ce sont les notes que tu
gardes ouvertes sur l'écran pendant la soutenance, et que tu peux
montrer au jury si une question creuse un choix. Effet immédiat sur
la perception de maturité du projet.
