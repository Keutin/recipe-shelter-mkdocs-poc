---
title: Banque de questions jury
description: ~104 questions de préparation à la soutenance RNCP
tags:
  - jury
  - soutenance
---

# Banque de questions jury — Recipe Shelter

> Préparation du **oral de soutenance** (RNCP, jury de 2 professionnels ≥ 3 ans d'expérience).
> Objectif : Arthur doit pouvoir **défendre chaque choix** de conception, **adapter son code en direct** face à une question imprévue, et **être force de proposition** sur les améliorations.

Cette banque n'est PAS un script à réciter — c'est un **terrain d'entraînement**. Le jury sentira immédiatement la récitation. Le but est qu'Arthur ait, pour chaque sujet, **la structure mentale et 2-3 ancres concrètes** (un fichier, une ligne de code, un schéma) pour répondre naturellement.

---

## Mode d'emploi

Chaque question est présentée selon le même gabarit :

- **Q** — la question telle que le jury peut la poser
- **Intention jury** — ce que le jury cherche réellement à vérifier (la question derrière la question)
- **Ossature de réponse** — les 3-5 points à couvrir, dans l'ordre
- **Ancres** — code / ADR / UML / fichier précis à pouvoir citer
- **Pièges** — ce qu'il NE faut PAS dire (et pourquoi)

---

## Sommaire

| # | Fichier | Thème | Volume |
|---|---|---|---|
| 00 | [`00-questions-transversales.md`](00-questions-transversales.md) | Projet, méthodo, organisation, Git, déploiement | ~15 Q |
| 01 | [`01-bloc1-frontend.md`](01-bloc1-frontend.md) | HTML/CSS/JS, responsive, accessibilité, Lighthouse | ~12 Q |
| 02 | [`02-bloc2-backend-from-scratch.md`](02-bloc2-backend-from-scratch.md) | Node, Express, POO, Repository, JWT, MySQL, tests | ~25 Q |
| 03 | [`03-bloc3-angular.md`](03-bloc3-angular.md) | Angular 21, Signals, SSR, standalone, routing, tests | ~15 Q |
| 04 | [`04-securite-rgpd.md`](04-securite-rgpd.md) | OWASP top 10, cookies HttpOnly, hashing, RGPD | ~12 Q |
| 05 | [`05-pieges-et-defense.md`](05-pieges-et-defense.md) | « Pourquoi pas X ? » — justifier Express, JWT, repo pattern, etc. | ~15 Q |
| 06 | [`06-ameliorations-innovantes.md`](06-ameliorations-innovantes.md) | Réponses prêtes pour « Comment iriez-vous plus loin ? » | ~10 Q |
| 07 | [`07-mode-entrainement.md`](07-mode-entrainement.md) | Plan de drill, simulation de soutenance, auto-évaluation | — |

**~100 questions au total.** Lire et travailler ~10/jour sur deux semaines avant la soutenance.

---

## Walkthrough pour Arthur

### Étape 1 — Lecture survol (1 h)

Lire les 7 fichiers en diagonale. Pas besoin de mémoriser ; identifier les questions qui font **flotter le ventre** — celles-là sont les chantiers prioritaires.

### Étape 2 — Drill par thème (2 semaines, ~1 h/jour)

Pour chaque question :

1. **Sans regarder l'ossature**, te répondre à voix haute (vraiment à voix haute, dans une pièce vide). Chronométrer : 60-90 s par réponse.
2. Comparer avec l'ossature. Identifier les **3 mots-clés manquants**.
3. Ouvrir le fichier/ADR/UML cité dans **Ancres** et relire 30 s. C'est ça qui ancre la réponse — pas le texte de la banque.
4. Refaire la réponse à voix haute, **sans regarder ni la question ni l'ossature**. Si encore raté, marquer la question d'un `[!]` dans la marge pour repasser le lendemain.

### Étape 3 — Simulation de soutenance (J-3 et J-1)

Voir [`07-mode-entrainement.md`](07-mode-entrainement.md) pour le protocole complet :
- Demander à Quentin (ou un pote dev) de poser 15 questions au hasard tirées des 7 fichiers
- Aucune note autorisée, juste l'écran de code ouvert
- Chronométrer la session totale (cible : ~30 min)
- Débrief : noter les 3 questions les plus faibles, les retravailler

### Étape 4 — Ne PAS copier ce dossier dans le repo `documentation`

Cette banque reste **dans `_draft_jury/`** au workspace, **hors des 3 repos git**. Ce sont des notes personnelles d'Arthur pour préparer sa soutenance, pas un livrable.

> Si Arthur veut quand même garder une trace dans son repo, il peut ajouter un **document court** (1-2 pages) dans `documentation/soutenance/Q&A-resumé.md` reprenant **uniquement** les 10-15 questions les plus probables avec des réponses en 3 lignes chacune. Mais ce n'est pas requis et ça peut donner au jury l'impression d'avoir trop préparé. À discuter.

---

## Principes de réponse à garder en tête

### Le triptyque « contrainte → choix → conséquence »

Pour chaque choix technique, structurer la réponse en 3 temps :

1. **Contrainte** — Qu'est-ce qui m'a amené à ce choix ? (cahier des charges, performance, sécurité, sobriété, contrainte pédagogique « from scratch »)
2. **Choix** — Qu'ai-je fait, concrètement ?
3. **Conséquence** — Quels compromis ai-je accepté ? Qu'est-ce que je ferais autrement en prod ?

Cette structure rassure le jury parce qu'elle montre qu'Arthur **a vu les alternatives** et **a fait un choix conscient**, pas un choix par défaut.

### L'humilité « du candidat lucide »

Le jury ne s'attend pas à du code parfait. Ils s'attendent à un candidat qui :
- **assume** ses choix (pas « j'ai pris ça parce qu'on m'a dit »)
- **connaît les limites** de son projet (« en prod je rajouterais X et Y »)
- **sait ce qu'il ne sait pas** (« je n'ai pas exploré Z, parce que… »)

Mieux vaut une réponse honnête « je n'ai pas implémenté ça mais voilà comment j'aurais fait » qu'une réponse vague.

### Le réflexe « on regarde le code »

À chaque fois que c'est possible : **proposer d'ouvrir le fichier**.
> « Si vous voulez, je peux ouvrir le `recipeService.ts`, c'est la classe centrale, j'y montre comment l'injection des dépendances est faite à la main. »

Ça transforme la question en démo, donne du temps de réflexion, et montre la maîtrise du code.

---

## Liens vers les autres artefacts déjà livrés

| Artefact | Emplacement | Utilité pour la soutenance |
|---|---|---|
| Diagrammes UML | [Diagrammes UML](../soutenance/backend/uml/README.md) | Support visuel pour expliquer l'archi |
| ADRs (5 décisions) | [ADRs](../soutenance/backend/adr/README.md) | Justifications écrites des choix techniques |
| Narration « from scratch » | [Narration from-scratch](../soutenance/backend/adr/00-narration-from-scratch.md) | **Pierre angulaire** pour défendre le respect du Bloc 2 |
| Cahier des charges officiel | `CAHIER_DES_CHARGES.md` (hors site, racine du workspace) | Le référentiel auquel se raccrocher en cas de doute |
