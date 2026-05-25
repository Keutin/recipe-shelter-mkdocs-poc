# ADR — Architecture Decision Records

> Documents formels de traçabilité des choix de conception. Servent de
> support écrit pour la soutenance, et de référence pour l'évolution
> future du projet.

## Pourquoi des ADRs

Un ADR (*Architecture Decision Record*) capture **un** choix de
conception : son contexte, la décision prise, les alternatives
considérées, et les conséquences positives et négatives.

Format inspiré de Michael Nygard (2011), adapté en français.

**Bénéfice direct pour la soutenance** : le jury voit que les choix
sont **réfléchis, datés, et tracés** — pas faits au hasard. Quand un
juré demande *"pourquoi Express ?"*, la réponse n'est pas une
improvisation orale mais un document écrit que tu as relu et que tu
peux défendre.

## Contenu

| Fichier | Sujet | Statut |
|---|---|---|
| `00-narration-from-scratch.md` | **Le document central** : défense de l'interprétation "from scratch" face au jury | Accepté |
| `adr-001-express-comme-couche-http.md` | Choix d'Express comme couche HTTP | Accepté |
| `adr-002-pattern-repository-interface-impl.md` | Découplage repository (interface + impl MySQL) | Accepté |
| `adr-003-jwt-cookie-httponly.md` | JWT en cookie HttpOnly vs header Authorization | Accepté |
| `adr-004-soft-delete-et-log-moderation.md` | Soft-delete + log de modération | Accepté |
| `adr-005-slug-en-deux-phases.md` | Slug technique en brouillon, slug public à la publication | Accepté |

## ADRs à considérer pour plus tard

Si le temps le permet avant la soutenance, ces ADRs renforceraient la
défense du Bloc 3 (frontend Angular) :

- **ADR-006** — Angular Signals + standalone components (vs RxJS + NgModules)
- **ADR-007** — Angular SSR (server-side rendering) — pourquoi et trade-offs
- **ADR-008** — TypeScript strict end-to-end (front + back)
- **ADR-009** — Tests Node natifs (`node:test`) côté backend (vs Jest/Vitest)
- **ADR-010** — Bootstrap 5 vs framework CSS custom

## Intégration dans le repo documentation (walkthrough pour Arthur)

### Étape 1 — Copier dans le repo

```powershell
# Windows PowerShell
mkdir -p soutenance/backend/adr
Copy-Item C:\DEV\ARTHUR\RECETTES\_draft_adr\*.md soutenance/backend/adr/
```

### Étape 2 — Relire et personnaliser

Pour la **narration "from scratch"** en particulier :

- Reformuler avec **tes propres mots** — le jury repère immédiatement
  un argumentaire récité d'un copier-coller.
- Ajuster les exemples de fichiers si le code a évolué.
- Vérifier que tu peux **expliquer chaque ligne** sans préparation : si
  tu ne peux pas, c'est qu'il faut simplifier ou retirer.
- Le document doit être ton aide-mémoire, pas un script à réciter.

Pour les ADRs :

- Si tu n'es pas d'accord avec une décision listée comme "considérée"
  ou avec une conséquence — supprime ou modifie. Un ADR doit refléter
  ta vraie pensée, pas un consensus théorique.
- Si tu as eu une **vraie hésitation** historique sur un choix, ajoute-la :
  ça rend le document plus crédible et ça te donne une histoire à
  raconter en soutenance.

### Étape 3 — Commit

Suivre la convention `docs/<short-name>` du repo
(`GIT_CONVENTION.md`) :

```bash
git checkout -b docs/adr-architecture-decisions
git add soutenance/backend/adr/
git commit -m "docs(adr): add architecture decision records and from-scratch narration"
git push -u origin docs/adr-architecture-decisions
```

Puis ouvrir une PR vers `main`.

## Pour la soutenance

- Imprimer (ou avoir en PDF) la **narration `00`** + les ADRs.
- Avoir une copie ouverte sur l'écran pendant la soutenance pour
  pouvoir y référer si le jury creuse.
- Si un juré demande *"vous avez documenté vos choix ?"* — montrer
  directement ce dossier `adr/` sur GitHub. Effet immédiat sur la
  perception de maturité du projet.
