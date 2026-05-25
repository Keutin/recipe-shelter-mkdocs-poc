# `_draft_backend_docs/` — Documentation backend pour soutenance

Drafts produits par Paige (tech writer) le 2026-05-25 pour la **Vague 1** du plan documentation backend de Recipe Shelter.

**Destination finale** : repo `recipe-shelter-documentation`, dossier `documentation/soutenance/backend/`. Voir [Walkthrough handoff](#walkthrough-handoff-arthur).

---

## Inventaire des livrables

| # | Fichier | Type | Cible repo doc |
| --- | --- | --- | --- |
| D1 | [`openapi/openapi.yaml`](openapi/openapi.yaml) | OpenAPI 3.1 (53 endpoints, 20+ schemas) | `backend/openapi/openapi.yaml` |
| D1 | [`openapi/index.html`](openapi/index.html) | Redoc viewer statique (CDN) | `backend/openapi/index.html` |
| D2 | [`errors.md`](errors.md) | Catalogue codes d'erreur (~140 codes, 11 domaines) | `backend/errors.md` |
| D3 | [`architecture.md`](architecture.md) | Vue d'ensemble narrative (820 lignes, 12 sections) | `backend/architecture.md` |
| D4 | [`securite.md`](securite.md) | Sécurité consolidée (440 lignes, 15 sections, OWASP, RGPD) | `backend/securite.md` |
| G1 | [`diagrams/g1-architecture-c4.md`](diagrams/g1-architecture-c4.md) | Mermaid C4 Container | `backend/diagrams/g1-architecture-c4.md` |
| G2 | [`diagrams/g2-pipeline-middleware.md`](diagrams/g2-pipeline-middleware.md) | Mermaid flowchart pipeline Express | `backend/diagrams/g2-pipeline-middleware.md` |
| G3 | [`diagrams/g3-sequence-moderation.md`](diagrams/g3-sequence-moderation.md) | Mermaid séquence modération recettes | `backend/diagrams/g3-sequence-moderation.md` |
| G4 | [`diagrams/g4-sequence-auth-email.md`](diagrams/g4-sequence-auth-email.md) | Mermaid séquences reset pwd + validation email | `backend/diagrams/g4-sequence-auth-email.md` |

Tous les documents couvrent le **Bloc 2** (backend from-scratch) de la certification.

---

## Points d'attention relevés pendant la production

Ces écarts entre le code et l'attendu / la convention ont été identifiés en passant. À toi de décider si tu les corriges avant soutenance ou si tu les transformes en "axes d'amélioration" dans ta défense (les deux postures se défendent).

### Nommage de l'erreur applicative

Le brief Vague 1 parlait de `AppError`. Le code réel utilise `HttpError` ([backend/src/utils/errors.ts:1](https://github.com/arthur-lagenebre/recipe-shelter-backend/blob/main/src/utils/errors.ts#L1)). Toute la doc emploie le nom **réel** (`HttpError`). Si tu renommes un jour, mets à jour `architecture.md`, `errors.md`, `securite.md`.

### Catalogue d'erreurs — 9 inconsistances signalées dans `errors.md`

Les plus notables :
- `USER_BANNED` est levé avec **deux statuts HTTP différents** (401 dans `require-auth.ts`, 403 dans `auth.service.ts`). Choisir l'un.
- `DbError` relaie les codes MySQL bruts en **500** : `ER_DUP_ENTRY` devrait devenir un 409, `ER_NO_REFERENCED_ROW` un 400. À mapper dans `db/errors.ts`.
- `NOT_FOUND` non préfixé par domaine dans `recipes.controller.ts:63` (convention violée).
- Pluralisation incohérente : `RECIPES_NOT_FOUND` vs `COMMENT_NOT_FOUND`.
- Tous les messages sont en **anglais** — colonne "Message FR proposé" prête dans `errors.md` si tu veux i18n-er côté frontend.

### Sécurité — vraies failles à connaître

Documentées dans `securite.md` section "Améliorations identifiées". Le jury *peut* tomber dessus :
- Aucun `helmet` / CSP / HSTS — facile à ajouter avant soutenance (1 ligne, gros effet sur la défense).
- Pas de `DELETE /users/me` exposé — **trou RGPD** explicite (cahier des charges mentionne droit à l'effacement).
- Pas de rate limit sur `/auth/reset-password` (alors qu'il y en a sur `/forgot-password`).
- Valeur littérale `12` au lieu de `env.auth.bcryptCost` dans `users.service.ts:112`.
- Rate limiter en mémoire (non scalable multi-instance) — acceptable en cert, à signaler.
- Pas de purge des tokens expirés (table grossit indéfiniment).

### OpenAPI — 3 zones TODO

Dans `openapi/openapi.yaml`, schémas marqués `oneOf` avec note TODO :
1. `GET /recipes/{recipeId}/comments` : pagination réelle ? (liste plate vs paginée)
2. `GET /favorites/me` : même question.
3. `POST /admin/recipes/:id/reject` : `409` ajouté par précaution sur transition de statut, à confirmer.

### Modération — pas d'email auteur sur rejet

`AdminRecipeService.reject` fait juste un UPDATE SQL. Pas d'appel mailer pour notifier l'auteur du refus. Signalé en `Note` dans `diagrams/g3-sequence-moderation.md`. Évolution naturelle (utiliser `SmtpMailService` déjà en place).

### Tests — 29 fichiers (le brief en annonçait 40)

Compté via glob `backend/tests/**/*.test.ts`. Si tu en as ajouté depuis, l'écart se résorbera.

---

## Walkthrough handoff Arthur

**Pré-requis** : être à jour sur `main` du repo `recipe-shelter-documentation` (cloné dans `C:\DEV\ARTHUR\RECETTES\documentation\`).

### Étape 1 — Créer la branche dédiée

```powershell
cd C:\DEV\ARTHUR\RECETTES\documentation
git checkout main
git pull
git checkout -b docs/backend-soutenance-vague1
```

### Étape 2 — Créer la structure cible

```powershell
mkdir documentation\soutenance\backend\diagrams
mkdir documentation\soutenance\backend\openapi
```

(Le dossier `documentation/soutenance/backend/uml/` existe déjà — on cohabite à côté.)

### Étape 3 — Copier les fichiers depuis `_draft_backend_docs/`

Depuis `C:\DEV\ARTHUR\RECETTES\` :

```powershell
copy _draft_backend_docs\architecture.md documentation\soutenance\backend\
copy _draft_backend_docs\errors.md documentation\soutenance\backend\
copy _draft_backend_docs\securite.md documentation\soutenance\backend\
copy _draft_backend_docs\diagrams\*.md documentation\soutenance\backend\diagrams\
copy _draft_backend_docs\openapi\*.* documentation\soutenance\backend\openapi\
```

### Étape 4 — Vérifier le rendu Mermaid sur GitHub

Les 4 diagrammes (`g1`–`g4`) sont en **Mermaid inline**. GitHub les rend nativement à l'ouverture du `.md`. Pas de tooling requis.

Pour prévisualiser **avant** push : VS Code avec l'extension *Markdown Preview Mermaid Support* (`bierner.markdown-mermaid`).

### Étape 5 — Vérifier le rendu Redoc en local

Ouvrir `documentation/soutenance/backend/openapi/index.html` directement dans le navigateur. Redoc se charge depuis le CDN (`cdn.redoc.ly`). Si tu veux le rendre **offline-friendly**, télécharger `redoc.standalone.js` localement et changer le `<script src>`.

### Étape 6 — Vérifier les liens internes

Plusieurs fichiers se référencent entre eux (`architecture.md` pointe vers `securite.md`, `errors.md`, `diagrams/g1`, `diagrams/g2`). Une fois les fichiers en place dans le repo doc, vérifier que les liens relatifs fonctionnent en cliquant sur l'aperçu GitHub.

### Étape 7 — Commit + push (Conventional Commits)

Respecter [`GIT_CONVENTION.md`](../../GIT_CONVENTION.md).

```powershell
cd C:\DEV\ARTHUR\RECETTES\documentation
git add documentation/soutenance/backend/
git commit -m "docs(soutenance): add backend vague 1 — openapi, architecture, errors, securite, diagrams"
git push -u origin docs/backend-soutenance-vague1
```

### Étape 8 — Ouvrir la PR

Sur GitHub `arthur-lagenebre/recipe-shelter-documentation`, ouvrir PR vers `main`. Titre suggéré :

> `docs(soutenance): backend vague 1 — openapi, architecture, errors, sécurité, diagrammes`

Description suggérée : copier l'inventaire de la table ci-dessus + section "Points d'attention".

---

## Et après ? — Vagues 2 et 3 différées

Cf. la mémoire `project_session_state.md` (hors site) — pas dans cette livraison :

- **Vague 2** : D5 data dictionary, D6 doc tests, D7 doc env vars, D8 README backend repo.
- **Vague 3** : G5 ERD Mermaid, G6 pipeline d'erreur, G7 slug two-phase, G8 user lifecycle, G9 archi déploiement (bloqué tant que Railway pas fait — cf. `_draft_deployment/`).

---

## Discours jury suggéré pour la doc backend

> « Pour le Bloc 2, j'ai documenté l'API en OpenAPI 3.1 avec un viewer Redoc — c'est le contrat unique entre back et front. À côté, trois documents textuels couvrent l'architecture en couches, le catalogue exhaustif des codes d'erreur, et la consolidation des choix sécurité avec mapping OWASP. Quatre diagrammes Mermaid complètent : un C4 Container pour la vue d'ensemble, le pipeline middleware Express qu'une requête traverse, et deux séquences sur les flux les plus sensibles — modération et auth par email. »

Pour les questions sur les **manques identifiés** (helmet, RGPD DELETE, etc.), assume-les : « j'ai produit la doc en parallèle d'un audit, voici ce que j'ai trouvé, voici ce que j'ai corrigé / voici ce que je mettrais en sprint suivant ».
