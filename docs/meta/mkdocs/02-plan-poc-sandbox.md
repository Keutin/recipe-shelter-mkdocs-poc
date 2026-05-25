# 02 — POC sandbox côté Quentin (`Keutin`)

Objectif : valider l'intégralité du pipeline MkDocs → GitHub Pages sur un repo sandbox **avant** de proposer le walkthrough à Arthur. Estimation : **~1h30**.

## Pré-requis

- ✅ `gh auth status` confirme compte actif `Keutin` (déjà vérifié)
- ✅ Identité git globale = Quentin (`Keutin` / `quentin.lagenebre@gmail.com`)
- Python 3.10+ installé localement (sinon : `winget install Python.Python.3.12`)
- Une partie représentative de la doc d'Arthur à mirrorer (cf. phase 2)

## Phase 1 — Créer le repo sandbox (~10 min)

```powershell
# Depuis C:\DEV\ARTHUR\RECETTES\_draft_mkdocs\
cd C:\DEV\CLAUDE  # ou n'importe quel dossier hors de C:\DEV\ARTHUR\
mkdir recipe-shelter-mkdocs-poc
cd recipe-shelter-mkdocs-poc
git init
```

Créer le repo distant sous le compte Keutin :

```powershell
gh repo create Keutin/recipe-shelter-mkdocs-poc `
  --public `
  --description "POC MkDocs pour le projet de cert de mon frère — sandbox, sera archivé" `
  --source . `
  --remote origin
```

**Pourquoi public** : nécessaire pour GitHub Pages sur compte free. Mettre une mention claire dans le README (« sandbox personnel, voir le vrai repo chez arthur-lagenebre »).

## Phase 2 — Mirror partiel de la doc d'Arthur (~15 min)

Copier `soutenance/` en entier depuis `C:\DEV\ARTHUR\RECETTES\documentation\` + intégrer les drafts déjà prêts pour avoir un rendu représentatif :

```powershell
$src = "C:\DEV\ARTHUR\RECETTES"
$dst = "C:\DEV\CLAUDE\recipe-shelter-mkdocs-poc\docs"

mkdir $dst
Copy-Item "$src\documentation\README.md" "$dst\index.md"
Copy-Item "$src\documentation\GIT_CONVENTION.md" "$dst\GIT_CONVENTION.md"
Copy-Item "$src\documentation\soutenance" "$dst\soutenance" -Recurse

# Intégrer les drafts déjà livrés (pour voir le rendu Mermaid + ADR)
Copy-Item "$src\_draft_uml\*.md" "$dst\soutenance\backend\uml\" -Force
mkdir "$dst\soutenance\backend\adr"
Copy-Item "$src\_draft_adr\*.md" "$dst\soutenance\backend\adr\"
mkdir "$dst\soutenance\frontend\audit"
Copy-Item "$src\_draft_bloc1_audit\*.md" "$dst\soutenance\frontend\audit\"
mkdir "$dst\manuel-utilisateur"
Copy-Item "$src\_draft_user_guide\*.md" "$dst\manuel-utilisateur\"
```

Vérifier visuellement la structure obtenue avant la phase 3.

## Phase 3 — Setup MkDocs local (~20 min)

```powershell
cd C:\DEV\CLAUDE\recipe-shelter-mkdocs-poc
python -m venv .venv
.\.venv\Scripts\Activate.ps1
```

Créer `requirements.txt` à la racine (cf. `01-architecture-et-justification.md`), puis :

```powershell
pip install -r requirements.txt
```

Créer `mkdocs.yml` à la racine (squelette dans `01-architecture-et-justification.md`). **Adapter** :
- `site_url` → `https://keutin.github.io/recipe-shelter-mkdocs-poc/`
- `repo_url` → `https://github.com/Keutin/recipe-shelter-mkdocs-poc`
- `site_author` → `POC sandbox (Quentin Lagenèbre)`

Lancer le serveur local :

```powershell
mkdocs serve
```

→ ouvrir http://127.0.0.1:8000

**Points à vérifier (checklist preview locale)** :
- [ ] Navigation latérale s'affiche avec toutes les sections
- [ ] Tous les diagrammes Mermaid rendent (UML + ADR)
- [ ] Recherche full-text fonctionne (taper "JWT", "RGAA", etc.)
- [ ] Toggle dark mode marche
- [ ] Tables (RGAA, comparatifs) bien rendues
- [ ] Liens internes (entre ADR, depuis README) ne sont pas cassés
- [ ] Code blocks ont le bouton "copier"

Si une page est cassée → ajuster le Markdown source (souvent un escape Mermaid manquant). Itérer jusqu'à preview clean.

## Phase 4 — Premier déploiement GitHub Pages (~20 min)

Créer `.gitignore` :

```
.venv/
site/
__pycache__/
*.pyc
```

Créer `.github/workflows/deploy-docs.yml` (cf. `01-architecture-et-justification.md`).

Premier commit + push :

```powershell
git add .
git commit -m "feat: mkdocs material POC with arthur's docs mirror"
git push -u origin main
```

Côté GitHub UI :
1. Aller sur `https://github.com/Keutin/recipe-shelter-mkdocs-poc/settings/pages`
2. Source : "Deploy from a branch"
3. Branch : `gh-pages` / `(root)` — **n'apparaîtra qu'après le premier run du workflow**
4. Save

Vérifier que le workflow tourne : `gh run watch` ou onglet Actions de l'UI.

À la fin du run, refresh la page Pages settings, sélectionner `gh-pages`, save.

→ URL live : `https://keutin.github.io/recipe-shelter-mkdocs-poc/` (peut prendre 1-2 min après la save Pages).

## Phase 5 — Validation finale (~20 min)

Sur le site déployé :
- [ ] Tous les points de la checklist preview locale, mais sur l'URL publique
- [ ] Navigation OK sur mobile (resize Chrome DevTools)
- [ ] Recherche fonctionne (l'index est généré au build)
- [ ] Liens "edit this page" pointent vers GitHub
- [ ] Lighthouse audit du site (curiosité : `Performance > 90`, `A11y > 95` attendu sur Material)

Faire un changement bidon dans `docs/index.md`, commit, push → vérifier que le workflow re-déploie automatiquement.

## Phase 6 — Documenter et archiver (~10 min)

Mettre à jour le `README.md` du repo POC avec :
- Mention "sandbox personnel"
- Lien vers le vrai repo Arthur
- Lien vers le walkthrough handoff (`_draft_mkdocs/03-walkthrough-handoff-arthur.md`)

Une fois Arthur a appliqué le walkthrough chez lui :
```powershell
gh repo archive Keutin/recipe-shelter-mkdocs-poc
```
ou suppression complète (`gh repo delete`).

## Sortie attendue

À la fin de cette phase, on a :
- Une URL live qui montre à quoi ressemblera la doc d'Arthur
- Un `mkdocs.yml` éprouvé, prêt à être copié
- Un workflow Actions validé, prêt à être copié
- Une liste de gotchas rencontrés (à intégrer dans `03-walkthrough-handoff-arthur.md`)
