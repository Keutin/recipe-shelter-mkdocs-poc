# `_draft_review/` — Revue globale de conformité

> **Outil de pilotage interne**, *pas* un livrable à copier dans le repo `documentation`.
> Permet à Quentin (et Arthur) d'avoir une vue d'ensemble du projet Recipe Shelter vs le cahier des charges RNCP.

## Contenu

| Fichier | Pour qui | Quand le lire |
|---------|---------|--------------|
| [`synthesis.md`](./synthesis.md) | Lecture rapide (≤ 5 min) | À chaque session de coordination Quentin↔Arthur, et la veille de la soutenance |
| [`compliance-matrix.md`](./compliance-matrix.md) | Référence exhaustive ligne-par-ligne | Quand on veut vérifier qu'une exigence précise est couverte ou pas |
| `README.md` (ce fichier) | Index + posture | Une fois, pour cadrage |

## Posture

La revue suit trois principes :

1. **Sources factuelles** — chaque évidence pointe vers un chemin réel (repo Arthur ou draft `_draft_*/`). Pas d'évidence inventée.
2. **Drafts = ready, pas done** — un livrable en draft avec README walkthrough est compté 🟡 (ready), pas 🟢. Le risque réel à signaler est l'exécution des handoffs par Arthur.
3. **Défense préparée** — pour chaque gap non comblable à temps (BrowserStack, tests intégration, etc.), un pitch jury est rédigé dans la synthèse. Le but est de ne jamais laisser Arthur en silence.

## Mise à jour

Ce dossier est un **snapshot** daté. Pour le rafraîchir après une vague de handoffs :

1. Re-lire `compliance-matrix.md` ligne par ligne.
2. Repasser 🟡 → 🟢 les exigences dont la copie vers `documentation` a été poussée et fusionnée.
3. Recompter les totaux de la synthèse.
4. Mettre à jour la date du snapshot.

## Lien avec la mémoire de session

Cette revue complète (sans remplacer) la mémoire de session Claude Code (`project_session_state.md` côté Quentin), qui reste la source de vérité pour les *décisions de session* et la chronologie. La matrice ici est la projection orthogonale : non plus « ce qu'on a fait quand » mais « ce que le cahier exige et où ça en est ».

## Pour Arthur

Si tu lis ça : le fichier à ouvrir en premier est [`synthesis.md`](./synthesis.md). Top 5 risques en haut + plan d'action J-14 → J-0 + discours de défense pour les questions piège. Le reste est en référence.
