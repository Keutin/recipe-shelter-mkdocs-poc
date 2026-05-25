# 03 — Walkthrough handoff Arthur

À envoyer à Arthur **une fois le POC sandbox validé** (phases 1-5 de `02-plan-poc-sandbox.md` terminées). Tout le travail ci-dessous se fait sous son compte GitHub `arthur-lagenebre`, ses commits, son repo.

Estimation Arthur : **~30 minutes** (pas besoin d'installer Python s'il fait confiance au CI).

---

## Pourquoi MkDocs

> Ta doc actuelle est sur GitHub mais c'est juste des fichiers Markdown. Un site MkDocs avec recherche, navigation, et une vraie URL `https://arthur-lagenebre.github.io/recipe-shelter-documentation/` à montrer au jury, ça change tout l'effet. Et c'est juste un fichier de config + un workflow GitHub Actions à copier — la CI build et déploie toute seule à chaque push.

## Étape 1 — Réorganiser le repo (10 min)

Sur ton clone local de `recipe-shelter-documentation`, branche dédiée :

```bash
cd C:\DEV\ARTHUR\RECETTES\documentation
git checkout -b docs/mkdocs-setup
```

Déplacer tout le contenu sous `docs/` (convention MkDocs) :

```powershell
mkdir docs
git mv soutenance docs/soutenance
git mv README.md docs/index.md
git mv GIT_CONVENTION.md docs/GIT_CONVENTION.md
```

Créer un nouveau `README.md` racine **minimal** qui pointe vers le site :

```markdown
# Recipe Shelter — Documentation

📖 **Site complet** : https://arthur-lagenebre.github.io/recipe-shelter-documentation/

Cette doc est générée avec [MkDocs Material](https://squidfunk.github.io/mkdocs-material/).

## Développement local

\`\`\`bash
pip install -r requirements.txt
mkdocs serve
\`\`\`

Puis ouvrir http://127.0.0.1:8000.
```

## Étape 2 — Copier les fichiers de config (5 min)

Récupérer **3 fichiers** depuis le POC sandbox (Quentin te les enverra) :
- `mkdocs.yml` (à la racine)
- `requirements.txt` (à la racine)
- `.github/workflows/deploy-docs.yml`

Adapter `mkdocs.yml` :
- `site_url: https://arthur-lagenebre.github.io/recipe-shelter-documentation/`
- `repo_url: https://github.com/arthur-lagenebre/recipe-shelter-documentation`
- `repo_name: arthur-lagenebre/recipe-shelter-documentation`
- `site_author: Arthur Lagenèbre`

Ajouter à `.gitignore` (le créer si absent) :
```
.venv/
site/
__pycache__/
*.pyc
```

## Étape 3 — (Optionnel) Preview locale (10 min)

**Si Python installé** :

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
pip install -r requirements.txt
mkdocs serve
```

→ http://127.0.0.1:8000

Vérifier que la navigation + Mermaid + recherche fonctionnent.

**Sinon** : skip cette étape, le CI te dira si quelque chose casse.

## Étape 4 — Commit + push (2 min)

```powershell
git add .
git commit -m "feat(docs): integrate mkdocs material with github pages deployment"
git push -u origin docs/mkdocs-setup
```

Ouvrir une PR sur ton repo (ou merger direct sur `main` selon ton workflow).

## Étape 5 — Activer GitHub Pages (3 min)

1. https://github.com/arthur-lagenebre/recipe-shelter-documentation/actions
   → vérifier que le workflow `Deploy MkDocs to GitHub Pages` a tourné (peut prendre 1-2 min après le merge sur `main`)
2. https://github.com/arthur-lagenebre/recipe-shelter-documentation/settings/pages
   → Source : "Deploy from a branch"
   → Branch : `gh-pages` / `(root)`
   → Save

3. Attendre 1-2 min → site live sur `https://arthur-lagenebre.github.io/recipe-shelter-documentation/`

## Étape 6 — Mettre à jour les références (5 min)

Liens à mettre à jour partout (3 endroits typiques) :
- README des **autres repos** (`recipe-shelter-backend`, `recipe-shelter-frontend`) : remplacer toute référence vers `documentation/` par l'URL Pages
- Slides soutenance (`_draft_slides/soutenance-slides.md`) : zone `[À COMPLÉTER]` URL doc
- Scénario démo (`documentation/docs/soutenance/demo/scenario-demo.md`) : ajouter le lien

## Gotchas rencontrés sur le POC (validés 2026-05-25)

POC live : https://keutin.github.io/recipe-shelter-mkdocs-poc/ (build 1.4s, deploy ~30s).

- **Ne pas utiliser `mkdocs build --strict` au premier build** — les drafts (`_draft_uml/`, `_draft_adr/`, etc.) se référencent entre eux par chemins relatifs sortants (`../_draft_jury/...`). Une fois tous les drafts copiés dans `docs/` chez Arthur, certains liens restent à ajuster manuellement. À traiter en passe dédiée après intégration.
- **Encoding Windows + ancres FR** — les ancres avec accents (`#24-mot-de-passe-oublié`) génèrent un warning à cause de l'encoding CP1252. N'empêche pas le build mais à corriger : remplacer par des ancres ASCII (`#mot-de-passe-oublie`) dans les liens internes du `_draft_user_guide/`.
- **GitHub Pages branche `gh-pages` PAS auto-activée** — `mkdocs gh-deploy` crée la branche mais il faut activer Pages via Settings → Pages (ou via API gh) une fois. Alternative scriptée :
  ```bash
  gh api repos/arthur-lagenebre/recipe-shelter-documentation/pages \
    -X POST -f "source[branch]=gh-pages" -f "source[path]=/"
  ```
- **Plugin `awesome-pages`** — fonctionne tel quel sans fichiers `.pages`, donne juste la possibilité d'en ajouter plus tard pour ordonner manuellement les sections.
- **Premier deploy via Actions** — workflow tourne en ~30s, site live ~20s après. Pas de 404 observé.
- **CRLF warnings au commit** — bénins sur Windows (`.gitattributes` non configuré), n'affectent ni le build ni le rendu.

## Que dire au jury à propos de MkDocs

> « Pour la doc, j'ai choisi MkDocs avec le thème Material parce que je voulais un site versionné, déployé automatiquement à chaque commit via GitHub Actions, avec recherche full-text. Ça me permet de garder tout en Markdown dans le repo — donc reviewable en PR — tout en produisant un rendu pro pour le jury et les futurs contributeurs. Le pipeline build dure 30 secondes, le site est sur GitHub Pages, pas de coût d'infra. »

**Si on te demande pourquoi pas Docusaurus / GitBook / Notion** :
> « MkDocs c'est Python, statique, zéro JavaScript côté visiteur, hyper léger. Docusaurus c'est React, overkill ici. GitBook/Notion c'est SaaS, pas versionné dans Git, pas dans l'esprit "from scratch" du projet. »
