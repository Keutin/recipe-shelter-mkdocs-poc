# UML — Brouillons à intégrer

> 5 diagrammes UML générés à partir de l'analyse du code backend
> (TypeScript/Express + MySQL) et du schéma de base de données.
>
> Format **Mermaid** : rendu nativement par GitHub dans le markdown,
> pas de build, pas d'outillage à installer.

## Contenu

| Fichier | Diagramme | Bloc concerné |
|---|---|---|
| `01-diagramme-classes-domaine.md` | Classes du domaine métier (3 vues : auth, recettes, social) | Bloc 2 |
| `02-diagramme-classes-architecture.md` | Architecture en couches du backend (pattern + instance Recipe) | Bloc 2 |
| `03-diagramme-cas-utilisation.md` | Cas d'utilisation par acteur (Visiteur / Utilisateur / Admin) | Bloc 2 |
| `04-diagramme-sequence-connexion.md` | Séquence de connexion + vérification de session | Bloc 2 |
| `05-diagramme-sequence-recette.md` | Cycle de vie d'une recette (création, soumission, modération) | Bloc 2 |

## Intégration dans le repo documentation (walkthrough pour Arthur)

### Étape 1 — Copier les fichiers

Depuis la racine de `recipe-shelter-documentation`, créer un dossier dédié
puis copier les 5 fichiers :

```powershell
# Windows PowerShell
mkdir -p soutenance/backend/uml
Copy-Item C:\DEV\ARTHUR\RECETTES\_draft_uml\*.md soutenance/backend/uml/
```

```bash
# macOS / Linux
mkdir -p soutenance/backend/uml
cp /chemin/vers/_draft_uml/*.md soutenance/backend/uml/
```

### Étape 2 — Renommer le README

Le `README.md` du dossier `_draft_uml` est ce fichier-ci, à ne pas copier
tel quel. Soit le supprimer après copie, soit l'adapter en un vrai README
décrivant les 5 diagrammes pour les lecteurs du repo doc.

### Étape 3 — Relire et personnaliser

Pour chaque fichier :

- Vérifier que les noms de classes/méthodes correspondent encore au code
  (le backend peut avoir évolué depuis la génération).
- Ajuster les notes de défense soutenance avec **tes propres mots** —
  un jury repère immédiatement un texte trop générique ou trop "AI".
- Ajouter ou retirer des cas d'usage selon ce qui est réellement
  implémenté à la date de la soutenance.

### Étape 4 — Vérifier le rendu

GitHub rend automatiquement les blocs Mermaid. Pour prévisualiser
localement :

- **VSCode** : extension `bierner.markdown-mermaid` (rendu dans l'aperçu
  markdown).
- **CLI** : `npx @mermaid-js/mermaid-cli -i fichier.md -o fichier.svg`
  pour exporter en SVG (utile pour la soutenance papier ou PDF).

### Étape 5 — Commit

Suivre la convention du projet (cf. `GIT_CONVENTION.md` à la racine du
repo doc). Exemple :

```bash
git checkout -b docs/uml-diagrams
git add soutenance/backend/uml/
git commit -m "docs(uml): add class, use case and sequence diagrams"
git push -u origin docs/uml-diagrams
```

Puis ouvrir une PR vers `main`.

## Pourquoi Mermaid plutôt que PlantUML ou draw.io ?

| Critère | Mermaid | PlantUML | draw.io |
|---|---|---|---|
| Rendu GitHub natif | ✅ | ❌ (image à exporter) | ❌ |
| Versionnable en texte | ✅ | ✅ | ⚠️ (XML non lisible) |
| Pas d'outillage requis | ✅ | ❌ (Java + serveur) | ❌ (app) |
| Conformité UML stricte | ⚠️ (proche) | ✅ | ✅ |
| Édition à 2h du matin | ✅ | ⚠️ | ❌ |

**Le compromis** : Mermaid est légèrement moins UML-pur que PlantUML
(notamment pour les diagrammes de cas d'utilisation), mais l'avantage
"rendu direct dans la PR GitHub" pèse beaucoup pour un projet
documentaire vivant. Si le jury exige du UML strict, l'export PlantUML
est facile à faire — la note dans `03-diagramme-cas-utilisation.md`
donne déjà la source PlantUML équivalente.

## Pour la soutenance

- Imprimer ou exporter en PDF les 5 diagrammes (1 par page) pour les
  avoir sous la main pendant la défense.
- Préparer les **3 questions probables du jury** pour chaque diagramme
  (notes de défense déjà incluses dans chaque fichier).
- Si projection : ouvrir les fichiers sur GitHub directement, le rendu
  Mermaid est propre et navigable.
