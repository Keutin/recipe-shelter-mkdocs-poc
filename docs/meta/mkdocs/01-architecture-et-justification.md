# 01 — Architecture cible et justification

## Choix du thème : Material for MkDocs

| Critère | Material | ReadTheDocs | Pourquoi Material gagne |
|---|---|---|---|
| Recherche full-text | ✅ intégrée, instantanée | ✅ mais plus basique | Bonus jury : démo en live d'une recherche |
| Mermaid natif | ✅ via `pymdownx.superfences` | ⚠️ nécessite plugin tiers | Tous les diagrammes UML/ADR déjà en Mermaid |
| Code highlighting | ✅ Pygments + copy button | ✅ Pygments | Égalité |
| Dark mode toggle | ✅ natif | ❌ | Détail mais effet "moderne" |
| Navigation tabs + sections | ✅ multi-niveaux | ✅ basique | Mieux pour soutenance/{backend,frontend,demo} |
| Admonitions (note/warning/tip) | ✅ riche | ✅ basique | Utile pour ADR et notes RGPD |
| Mainteneur | Squidfunk (actif, sponsor GitHub) | RTD team | Égalité |
| Effet visuel jury | "site pro" | "doc open-source 2010" | Material |

**Décision** : Material for MkDocs.

## Structure cible du repo `documentation/` après intégration

```
documentation/
├── README.md                    # garde-fou : pointe vers le site déployé
├── GIT_CONVENTION.md            # conserve à la racine (méta-doc)
├── mkdocs.yml                   # config MkDocs
├── requirements.txt             # mkdocs-material + plugins (pour CI)
├── .github/
│   └── workflows/
│       └── deploy-docs.yml      # build + push gh-pages
└── docs/                        # racine du site MkDocs (convention)
    ├── index.md                 # page d'accueil (peut reprendre README projet)
    ├── soutenance/
    │   ├── index.md             # vue d'ensemble blocs 1/2/3
    │   ├── backend/
    │   │   ├── uml/             # les 5 diagrammes + README → index.md
    │   │   └── adr/             # une fois _draft_adr/ copié
    │   ├── frontend/
    │   │   ├── audit/           # une fois _draft_bloc1_audit/ copié
    │   │   ├── Responsive/
    │   │   └── Screenshots/
    │   └── demo/
    │       ├── scenario-demo.md
    │       └── comptes-test.md
    └── manuel-utilisateur/      # une fois _draft_user_guide/ copié
        ├── index.md
        ├── 01-decouverte.md
        └── ...
```

**Note** : MkDocs exige par défaut que le contenu soit sous `docs/`. Deux options :
- A) Déplacer le contenu actuel sous `docs/` (commit "chore: prepare for mkdocs")
- B) Configurer `docs_dir: .` dans `mkdocs.yml` pour garder la structure plate

→ **Recommandation : A**, c'est la convention et ça évite que `mkdocs.yml`/`requirements.txt`/`.github/` polluent la racine du contenu.

## `mkdocs.yml` cible (squelette)

```yaml
site_name: Recipe Shelter — Documentation
site_description: Documentation technique et soutenance du projet Recipe Shelter
site_author: Arthur Lagenèbre
site_url: https://arthur-lagenebre.github.io/recipe-shelter-documentation/
repo_url: https://github.com/arthur-lagenebre/recipe-shelter-documentation
repo_name: arthur-lagenebre/recipe-shelter-documentation
edit_uri: edit/main/docs/

theme:
  name: material
  language: fr
  features:
    - navigation.tabs
    - navigation.sections
    - navigation.expand
    - navigation.top
    - search.highlight
    - search.suggest
    - content.code.copy
    - content.action.edit
  palette:
    - scheme: default
      primary: deep orange
      accent: amber
      toggle:
        icon: material/weather-night
        name: Mode sombre
    - scheme: slate
      primary: deep orange
      accent: amber
      toggle:
        icon: material/weather-sunny
        name: Mode clair
  icon:
    repo: fontawesome/brands/github

markdown_extensions:
  - admonition
  - attr_list
  - md_in_html
  - tables
  - toc:
      permalink: true
  - pymdownx.details
  - pymdownx.superfences:
      custom_fences:
        - name: mermaid
          class: mermaid
          format: !!python/name:pymdownx.superfences.fence_code_format
  - pymdownx.tabbed:
      alternate_style: true
  - pymdownx.highlight:
      anchor_linenums: true
  - pymdownx.inlinehilite
  - pymdownx.snippets

plugins:
  - search:
      lang: fr
  - awesome-pages   # optionnel : permet d'ordonner via .pages au lieu de tout déclarer

nav:
  - Accueil: index.md
  - Soutenance:
    - Vue d'ensemble: soutenance/index.md
    - Backend:
      - UML: soutenance/backend/uml/README.md
      - ADR: soutenance/backend/adr/README.md
    - Frontend:
      - Audit qualité: soutenance/frontend/audit/README.md
    - Démo: soutenance/demo/scenario-demo.md
  - Manuel utilisateur:
    - manuel-utilisateur/index.md
  - Convention Git: GIT_CONVENTION.md
```

## `requirements.txt`

```
mkdocs-material==9.5.*
mkdocs-awesome-pages-plugin==2.9.*
pymdown-extensions==10.*
```

Versions épinglées en minor pour CI reproductible, mineures auto-update.

## Workflow GitHub Actions

`.github/workflows/deploy-docs.yml` :

```yaml
name: Deploy MkDocs to GitHub Pages

on:
  push:
    branches: [main]
    paths:
      - 'docs/**'
      - 'mkdocs.yml'
      - 'requirements.txt'
      - '.github/workflows/deploy-docs.yml'
  workflow_dispatch:

permissions:
  contents: write

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.12'
          cache: pip
      - run: pip install -r requirements.txt
      - run: mkdocs gh-deploy --force --clean
```

`mkdocs gh-deploy` build + push automatiquement sur la branche `gh-pages`. GitHub Pages doit être configuré pour servir depuis cette branche (réglage UI une fois).

## Ce que ça apporte au jury (Bloc 1 et Bloc 3)

- **Bloc 1 (qualité doc)** : démontre une démarche pro de documentation versionnée + CI/CD
- **Bloc 3 (méthodes)** : ajoute un outil moderne au CV technique (Python ecosystem, GitHub Actions, Pages)
- **Argument "from scratch" préservé** : MkDocs est purement côté doc, ne change rien à l'app

## Risques et mitigations

| Risque | Mitigation |
|---|---|
| Casser les liens internes du repo en déplaçant tout sous `docs/` | Lister les liens actuels avant déplacement, ajuster en masse |
| Pages cassées si Mermaid mal échappé | Tester en local avec `mkdocs serve` avant chaque push |
| Délai serré avant soutenance | Le POC sandbox sert justement à dérisquer — handoff Arthur = simple copie |
| Arthur n'a pas Python | `mkdocs serve` optionnel, GitHub Actions fait tout sinon |
