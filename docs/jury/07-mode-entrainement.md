# 07 — Mode d'entraînement

Comment passer **de la lecture à la maîtrise**. Cette banque ne sert à rien si Arthur la lit une fois et ferme l'onglet. Voici un protocole concret.

---

## Principes

### 1. À voix haute, dans une pièce vide

Lire mentalement n'entraîne **pas** la diction. Le jour J, Arthur va parler — il faut donc s'entraîner à parler. Une pièce vide (pas la salle de bain, l'écho fausse les repères) suffit.

### 2. Sans regarder les notes

Le seul intérêt d'une question, c'est de la **rejouer sans support**. Une réponse bien lue ne vaut rien. Lire l'ossature **avant** la question, puis la cacher et répondre.

### 3. Chronométrer

Cible : **60-90 secondes par réponse**. Plus court → on n'a pas vraiment répondu. Plus long → on perd le jury. Un chrono sur le téléphone, lancé en silencieux, suffit pour calibrer.

### 4. Le code en arrière-plan

Ouvrir VSCode sur les fichiers cités dans les **Ancres** de chaque question, et **pointer vers le bon endroit en répondant** comme si le jury était là. Ça ancre la réponse au code réel.

### 5. Le code lui-même est le meilleur révisable

Si une question demande « explique-moi `recipeService.ts` », la seule préparation valide c'est de **relire ce fichier en entier**, pas de relire la banque. La banque est un *index*, pas un *substitut* au code.

---

## Plan sur 2 semaines (J-14 → J-0)

### Semaine 1 — Lecture et drill par thème

| Jour | Activité | Durée |
|---|---|---|
| J-14 | Lecture survol des 7 fichiers du dossier `_draft_jury/` | 1 h |
| J-13 | Drill `00-transversales.md` (les 15 questions) + relire les README des 3 repos | 1 h |
| J-12 | Drill `01-bloc1-frontend.md` + lancer Lighthouse sur les 3 pages clés, noter les scores | 1 h 30 |
| J-11 | Drill `02-bloc2-backend-from-scratch.md` **partie 1** (Q1–Q12) + relire `app.ts` ligne par ligne | 1 h 30 |
| J-10 | Drill `02-bloc2-backend-from-scratch.md` **partie 2** (Q13–Q25) + relire un service complet + un repo complet | 1 h 30 |
| J-9 | Drill `03-bloc3-angular.md` + relire les routes, un service Angular, un composant clé | 1 h 30 |
| J-8 | **Repos.** Pas de drill. Recopier mentalement les 5 ADRs en se les racontant. | 30 min |

### Semaine 2 — Sécurité, pièges, simulation

| Jour | Activité | Durée |
|---|---|---|
| J-7 | Drill `04-securite-rgpd.md` + ouvrir l'OWASP Top 10 sur owasp.org pour rafraîchir | 1 h 30 |
| J-6 | Drill `05-pieges-et-defense.md` — c'est le fichier le plus important après le Bloc 2 | 2 h |
| J-5 | Drill `06-ameliorations-innovantes.md` — choisir 3 idées qu'Arthur veut **vraiment** défendre | 1 h |
| J-4 | **Simulation #1** : Quentin (ou un dev pote) pose 15 questions au hasard, chrono total ~30 min, débrief 15 min | 1 h |
| J-3 | Retour sur les 3 questions les plus faibles de la simulation. Re-drill. | 1 h |
| J-2 | **Simulation #2** : 20 questions cette fois, dont 5 sur du code à expliquer en ouvrant le fichier en direct | 1 h 30 |
| J-1 | **Repos.** Lecture rapide des ADRs. Relire les slides. Dormir. | 30 min |
| J-0 | **Lecture ZERO.** Petit-déjeuner. Marche. Ne PAS relire la banque le matin — ça fatigue. | — |

---

## Protocole de simulation (J-4 et J-2)

### Setup

- Une pièce calme, à une table.
- Arthur a son **laptop ouvert sur le code** (les 3 repos) — c'est sa béquille légitime, le jury le permet aussi.
- L'examinateur (Quentin / pote) tire des questions au hasard dans les 7 fichiers (utiliser une roulette de 1 à ~100 si possible).
- **Aucune note papier** autorisée sur la table à part le cahier des charges.

### Déroulé

1. **Présentation 2 min** (Q1 de `00-transversales.md`) — chronométré.
2. **15-20 questions** réparties :
   - 3-4 transversales
   - 2 Bloc 1
   - 5-6 Bloc 2 (le plus gros poids)
   - 2-3 Bloc 3
   - 2 sécurité/RGPD
   - 2 pièges
   - 1 amélioration innovante
3. **Question d'ouverture de code** : « Montre-moi comment fonctionne X » — Arthur ouvre le fichier et explique.
4. **Question d'adaptation** : « Ajoute le rôle modérateur » — Arthur fait la modif en direct (5-10 min).

### Débrief

Après la simu, Quentin note pour chaque question :
- ✅ Réponse claire, structurée, bien sourcée
- ⚠ Réponse approximative ou hésitante
- ❌ Réponse fausse, vague ou réfutée par le jury fictif

Les ⚠ et ❌ sont les chantiers à retravailler dans les jours suivants.

---

## Checklist du jour J

### Matériel
- [ ] Laptop chargé (chargeur dans la sacoche)
- [ ] Connexion internet vérifiée la veille (si démo en ligne)
- [ ] Les 3 repos clonés et fonctionnels en local (lance le back + le front une dernière fois la veille)
- [ ] Base MySQL démarrée et seedée avec des données démo
- [ ] Postman ouvert avec les requêtes pré-configurées
- [ ] VSCode ouvert avec les onglets clés : `app.ts`, `recipeService.ts`, le middleware d'auth, `app.routes.ts` Angular
- [ ] Cahier des charges imprimé ou en PDF accessible
- [ ] Slides ouvertes en local (chantier #7 — quand fait)

### Présentation
- [ ] Slides exportées en PDF (au cas où LibreOffice/PowerPoint plante)
- [ ] Diagrammes UML ouverts dans un onglet (chantier #2 fait dans `documentation/soutenance/backend/uml/`)
- [ ] Démo en ligne ouverte dans un onglet (chantier #9)
- [ ] Compte de démo `admin` + un compte `user` prêts à se loguer

### Mental
- [ ] Arrivée 30 min en avance
- [ ] Hydraté, mais sans excès (pas envie de devoir partir aux toilettes en plein milieu)
- [ ] Vêtements **sobres et confortables** (rien de trop neuf ou trop habillé)
- [ ] Téléphone en mode avion
- [ ] Si question = blanc total : « Bonne question. Laissez-moi 10 secondes pour structurer ma réponse. » — ça achète du temps légitime.

---

## Auto-diagnostic : suis-je prêt ?

Arthur est prêt si :

- [ ] Il peut faire la **présentation 2 min** par cœur sans hésitation.
- [ ] Il peut nommer **les 5 ADRs** et expliquer chacune en 30 secondes.
- [ ] Il peut **ouvrir `app.ts` et expliquer chaque ligne** de l'instanciation des services.
- [ ] Il peut **expliquer le flow d'auth complet** : connexion → JWT → cookie → middleware → service.
- [ ] Il peut **citer 3 améliorations sécurité, 3 améliorations perf, 3 fonctionnalités UX** sans réfléchir.
- [ ] Il peut **défendre Express comme couche HTTP** sans broncher face à 3 contre-arguments successifs.
- [ ] Il peut **modifier le code en direct** : ajouter un rôle, ajouter un endpoint, modifier une validation.
- [ ] Il a fait au moins **une simulation complète** avec un tiers, et a retravaillé les ⚠/❌.
- [ ] La **démo en ligne fonctionne** et il l'a testée la veille (chantier #9).

Si toutes les cases sont cochées : **c'est bon**. Arthur peut y aller détendu.

---

## Si ça se passe mal pendant la soutenance

### Question hors-sujet ou floue

> « J'aimerais m'assurer de bien comprendre — quand vous parlez de [terme], vous voulez dire [reformulation A] ou plutôt [reformulation B] ? »

### Question dont on ignore la réponse

> « Honnêtement, je ne saurais pas répondre avec certitude. Ce que je sais en revanche, c'est [parler de ce qu'on sait sur le sujet adjacent]. Et je traiterais la question en [hypothèse de démarche]. »

C'est largement mieux que d'inventer.

### Le jury insiste avec un contre-argument fort

Deux cas :

1. **Le contre-argument est valide** → reconnaître : « Vous avez raison, je n'avais pas envisagé ça sous cet angle. Effectivement [reprendre l'argument du jury]. Ce que je referais c'est [adaptation]. »
2. **Le contre-argument est discutable** → tenir, mais avec humilité : « Je comprends votre point. Mon raisonnement reste que [argument]. Mais je note la nuance et c'est typiquement le genre de décision que je rediscuterais avec une équipe en prod. »

### Erreur dans le code montrée en live

> « Vous avez raison, il y a un souci ici. Laissez-moi le corriger. »

Et le faire. C'est exactement ce que le cahier des charges décrit comme « modifier son code en temps réel ».

---

## Mantras

À se répéter dans les jours qui précèdent :

1. **Je connais mon code.** Personne dans cette salle ne connaît Recipe Shelter mieux que moi.
2. **Le jury veut me voir réussir.** Ils ne sont pas là pour me piéger, ils sont là pour vérifier que j'ai compris ce que j'ai fait.
3. **Une réponse honnête « je ne sais pas » + une démarche** vaut mieux qu'un mensonge cohérent.
4. **Le code est mon ami.** Quand je doute, j'ouvre le fichier et je raisonne dessus, pas dans ma tête.
5. **C'est un échange, pas un interrogatoire.** Je peux poser des questions, demander à reformuler, prendre 5 secondes pour penser.
