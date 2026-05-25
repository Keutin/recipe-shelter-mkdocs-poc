# MkDocs — Plan d'intégration

Plan pour adopter **Material for MkDocs** sur le repo `recipe-shelter-documentation` d'Arthur, avec une étape POC sandbox côté Quentin (`Keutin` sur GitHub) pour valider le pipeline avant handoff.

## Pourquoi ce draft

L'état actuel du repo `documentation/` :
- `README.md` racine + `GIT_CONVENTION.md`
- `soutenance/` avec sous-dossiers `backend/`, `demo/`, `frontend/`
- ~10 fichiers Markdown au total (dont les 6 UML livrés en `_draft_uml/`)
- Pas de site généré, pas de navigation, pas de recherche
- Le `README.md` racine référence des dossiers (`docs/Database/`, `docs/API backend/`) **qui n'existent pas** → incohérence visible par le jury

MkDocs apporte :
1. **Navigation latérale + recherche full-text** (UX jury bien meilleure qu'un README GitHub)
2. **Rendu Mermaid natif** (cohérent avec tous les diagrammes déjà livrés en `_draft_uml/`, `_draft_adr/`)
3. **Déploiement GitHub Pages en 1 fichier** (`.github/workflows/gh-pages.yml`)
4. **Effet pro côté soutenance** : URL `https://arthur-lagenebre.github.io/recipe-shelter-documentation/` à montrer au jury

## Structure du draft

- `01-architecture-et-justification.md` — choix de Material, structure `docs/`, `mkdocs.yml` cible, plugins, ce qui va où
- `02-plan-poc-sandbox.md` — pas-à-pas côté Quentin : créer `recipe-shelter-mkdocs-poc` sur `Keutin`, mirror partiel, valider GitHub Pages
- `03-walkthrough-handoff-arthur.md` — ce qu'Arthur exécute sur son repo une fois le POC validé (installation, copie du `mkdocs.yml`, activation Pages, premier déploiement)

## Principe d'invisibilité — rappel

Le POC sandbox vit **sur le compte GitHub de Quentin (`Keutin`)**, pas sur celui d'Arthur. Personne d'extérieur ne voit la trace. Au final, Arthur exécute le walkthrough sur **son** repo et tous les commits MkDocs sont sous **son** nom.

Le repo sandbox sert seulement à :
- Valider que le pipeline GitHub Actions → Pages fonctionne (évite de polluer l'historique d'Arthur avec des essais)
- Voir le rendu réel avec une portion de la vraie doc avant de figer le `mkdocs.yml`
- Pouvoir montrer une preview à Arthur sans qu'il ait à installer Python+MkDocs juste pour décider

Une fois le walkthrough validé, le repo POC peut être archivé ou supprimé. Aucune dépendance entre les deux après handoff.

## Ordre de lecture suggéré

1. `01-architecture-et-justification.md` pour valider les choix techniques
2. `02-plan-poc-sandbox.md` pour exécuter côté Quentin (~1h30)
3. `03-walkthrough-handoff-arthur.md` pour préparer le handoff à Arthur (~30 min côté Arthur)

## Décisions ouvertes

- **Nom du repo POC** : `recipe-shelter-mkdocs-poc` par défaut (à adapter)
- **Étendue du mirror** : copier `soutenance/` en entier ou seulement `backend/uml/` ? → recommandation : tout `soutenance/` + le futur `_draft_adr/` pour avoir un rendu représentatif
- **Faire le POC avant ou après que les autres `_draft_*` soient copiés chez Arthur ?** → indépendant, peut tourner en parallèle
