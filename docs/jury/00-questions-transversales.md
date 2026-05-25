# 00 — Questions transversales

Questions générales sur le projet, l'organisation, le cycle de vie. Ce sont presque toujours les **premières questions** du jury — c'est l'occasion de poser un cadre clair, de paraître posé, et d'aiguiller le jury vers les sujets que tu maîtrises le mieux.

---

## Q1. Présentez-nous votre projet en 2 minutes.

**Intention jury** — Premier filtre : est-ce qu'Arthur sait synthétiser ? Est-ce qu'il connaît son projet de bout en bout ?

**Ossature de réponse**

1. **Le quoi** — « Recipe Shelter est un site collaboratif de recettes. Les utilisateurs peuvent publier, commenter, et sauvegarder des recettes. Un rôle administrateur permet la modération. »
2. **Le pourquoi (côté cert)** — « Le projet couvre les trois blocs : un front HTML/CSS/JS pur, un back-end Node.js développé entièrement from-scratch sans framework HTTP de haut niveau, et une application Angular qui consomme l'API. »
3. **Les chiffres** — « Côté back : environ X endpoints, Y entités MySQL, Z tests unitaires. Côté front Angular : N composants standalone, SSR activé. »
4. **L'aboutissement** — « C'est déployé sur [URL], et tout le code est sur GitHub sous l'organisation `arthur-lagenebre`. »

**Ancres** — connaître par cœur le nombre de tables, le nombre d'endpoints, le nombre de tests.

**Pièges**
- ❌ Commencer par la techno (« j'ai utilisé Node, Angular, MySQL ») → ça sonne comme un CV.
- ❌ Dépasser 2 min sur la présentation → le jury décroche.

---

## Q2. Pourquoi avoir choisi ce projet ?

**Intention jury** — Vérifier la motivation, voir si Arthur s'est approprié le sujet ou s'il l'a subi.

**Ossature de réponse**

1. Sujet imposé dans la liste de la formation, **mais** :
2. Il combine plusieurs axes intéressants techniquement : authentification + rôles + recherche multi-critères + modération + interactions sociales (favoris, commentaires).
3. C'est un domaine qu'il comprend (la cuisine, c'est concret, ça l'a aidé à imaginer les cas d'usage utilisateurs).

**Pièges**
- ❌ « C'est le sujet qu'on m'a donné. » → trop passif.
- ❌ Survendre une passion personnelle → si le jury creuse et qu'il n'y a rien derrière, ça se voit.

---

## Q3. Comment vous êtes-vous organisé pour mener ce projet ?

**Intention jury** — Méthodologie de travail, capacité à planifier seul.

**Ossature de réponse**

1. **Découpage par bloc** — d'abord le back-end from-scratch (Bloc 2) parce qu'il définit le contrat d'API ; ensuite l'Angular (Bloc 3) qui consomme l'API ; le front pur HTML/CSS/JS (Bloc 1) en parallèle pour les pages statiques / showcase.
2. **Sprints courts** — découpage en petites itérations, une fonctionnalité = une branche = une PR.
3. **Conventional Commits** — formalisé dans `documentation/GIT_CONVENTION.md`. Ça aide à se relire et à générer un changelog si besoin.
4. **Documentation au fil de l'eau** — pas en bloc à la fin. Les ADRs sont écrits quand la décision est prise, pas reconstruits a posteriori.

**Ancres** — `documentation/GIT_CONVENTION.md`, structure `feat:`, `fix:`, `docs:`.

**Pièges**
- ❌ Dire « j'ai utilisé Scrum » sans pouvoir nommer les rituels concrets → le jury creuse et c'est embarrassant.

---

## Q4. Avez-vous utilisé un gestionnaire de versions ? Comment ?

**Intention jury** — Maîtrise de Git, bonne hygiène.

**Ossature de réponse**

1. **Trois repos séparés** sur GitHub (`recipe-shelter-backend`, `-frontend`, `-documentation`) — décision consciente pour bien isoler les responsabilités.
2. **Workflow par branches** : `main` protégée, branches `feat/`, `fix/`, `docs/` selon le type de changement.
3. **Conventional Commits** documentés.
4. **Hooks Husky** côté back pour passer le lint + les tests avant chaque commit.

**Ancres** — `documentation/GIT_CONVENTION.md`, le `husky/` du back.

**Pièges**
- ❌ Si jamais une question sur le rebase vs merge : avoir une réponse honnête (« j'ai fait des merges principalement, je n'ai pas systématiquement rebase »).

---

## Q5. Quelle est votre stack technique complète ?

**Intention jury** — Inventaire propre, mais surtout : capacité à expliquer **pourquoi** chaque brique.

**Ossature de réponse** — Annoncer par couche :

- **Front Bloc 1** : HTML5, CSS3 (avec media queries, variables CSS), JavaScript ES modules
- **Back Bloc 2** : Node.js (LTS), TypeScript, Express (5.x) — *uniquement comme couche HTTP* (voir ADR-001), mysql2 (driver natif, pas d'ORM), bcrypt, jsonwebtoken, cookie-parser, cors, dotenv, nodemailer
- **Tests back** : `node:test` (runner natif Node, voir narration from-scratch)
- **DB** : MySQL 8
- **Front Bloc 3** : Angular 21, Signals, standalone components, SSR, Bootstrap 5, TypeScript strict
- **Outillage** : ESLint, Husky, Stylelint, Postman pour les essais d'API

**Pièges**
- ❌ Énumérer sans justifier → préparer une mini-justif pour chaque brique au cas où le jury demande « pourquoi celle-là ? ».

---

## Q6. Où est déployée votre application ? Comment ?

**Intention jury** — Le cahier des charges exige une **démo en ligne** pour chaque bloc.

**Ossature de réponse** — *(à compléter avec Arthur — pending #9 dans la roadmap)*

- Front Angular : `[URL]` — hébergé sur `[plateforme]`
- API back : `[URL]` — hébergée sur `[plateforme]`
- DB : `[où]`
- Variables d'environnement gérées via `.env` (non versionné), template dans `.env.example`
- Process de déploiement : `[manuel / CI]`

**Pièges**
- ❌ « C'est en local. » → rouge vif. Le cahier des charges l'exige en ligne.

> **⚠ TODO Arthur** : ce point est **bloquant**. Si rien n'est en ligne au jour J, c'est éliminatoire. À traiter avant la soutenance.

---

## Q7. Combien de temps avez-vous passé sur ce projet ? Quelle a été la phase la plus difficile ?

**Intention jury** — Auto-évaluation, lucidité, identifier les points de douleur (donc les zones où ils vont creuser).

**Ossature de réponse**

1. Temps total réaliste (ne pas survendre — un jury sent l'exagération).
2. **Phase la plus difficile** : choisir un sujet qu'Arthur maîtrise vraiment, parce que le jury va creuser dessus. Ex possible :
   - « La gestion de l'authentification — quel format de token, où le stocker, comment gérer l'expiration. J'ai itéré 2-3 fois avant de fixer JWT + cookie HttpOnly (voir ADR-003). »
   - ou « Le pattern Repository pour rester from-scratch tout en gardant le code testable — j'ai dû désapprendre les réflexes ORM. »

**Ancres** — choisir une difficulté avec un ADR derrière.

**Pièges**
- ❌ « Rien n'était difficile » → ça paraît arrogant ou superficiel.
- ❌ Choisir une difficulté qu'on ne maîtrise plus → ils vont creuser.

---

## Q8. Si vous deviez recommencer le projet aujourd'hui, que changeriez-vous ?

**Intention jury** — Recul, lucidité, force de proposition.

**Ossature de réponse** — Citer 2-3 points concrets, **pas trop critiques** sur le travail réalisé :

1. **Tests E2E** : « J'ai des tests unitaires nombreux côté back, mais j'aurais aimé ajouter une couche d'E2E avec Playwright pour valider le parcours bout-en-bout. »
2. **CI/CD** : « Mettre en place GitHub Actions pour le lint + tests automatiques + déploiement à chaque push sur `main`. »
3. **Observabilité** : « Ajouter un logger structuré (Pino) et exposer un endpoint `/health` pour le monitoring. »

**Pièges**
- ❌ Énumérer 10 choses → ça revient à dire « mon projet est mauvais ».
- ❌ Pointer un défaut qu'on aurait dû corriger en 2 h → ça donne l'air paresseux.

---

## Q9. Quel a été votre processus de test ? Combien de tests, quelle couverture ?

**Intention jury** — Maturité sur la qualité.

**Ossature de réponse**

1. **Test runner natif** Node (`node:test`) — choix cohérent avec la contrainte from-scratch (voir ADR / narration).
2. **~40 fichiers de tests** côté back, couvrant DTOs (validation), services (logique métier), middlewares (auth/erreurs), mappers.
3. **Stratégie** : tests unitaires sur les services en mockant les repositories ; pas de tests d'intégration en base réelle (limite assumée).
4. **Front Angular** : tests sur les services et composants critiques avec Jasmine/Karma (le runner par défaut d'Angular).

**Ancres** — `backend/tests/`, montrer un fichier `*.test.ts` représentatif (ex : un service).

**Pièges**
- ❌ Dire « j'ai 90 % de couverture » sans pouvoir le prouver → si le jury demande un rapport, c'est gênant.

---

## Q10. Comment avez-vous géré les erreurs et les cas limites ?

**Intention jury** — Robustesse, vision défensive.

**Ossature de réponse**

1. **Middleware d'erreur Express** centralisé qui transforme les exceptions en réponses HTTP propres (404, 400, 500).
2. **Validation des entrées via DTOs** : chaque endpoint a un DTO qui valide la forme du payload avant que la requête atteigne le service.
3. **Erreurs typées** dans le domaine (ex : `RecipeNotFoundError`, `UnauthorizedError`) → le middleware sait les mapper vers le bon code HTTP.
4. **Côté front Angular** : intercepteur HTTP pour capter les 401 et rediriger vers `/login`, toasts pour les autres erreurs.

**Ancres** — middleware d'erreur du back (chemin exact), un DTO représentatif.

**Pièges**
- ❌ Avoir un `try/catch` géant qui renvoie 500 sur tout → si c'est ça le code, mieux vaut le dire et l'assumer comme limite.

---

## Q11. Avez-vous documenté votre projet ? Pour qui ?

**Intention jury** — Vérifier que le livrable « documentation » du cahier des charges est respecté.

**Ossature de réponse**

1. **README** dans chacun des 3 repos avec install + démarrage.
2. **`documentation/` repo dédié** avec :
   - Diagrammes UML (5 diagrammes : domaine, archi, cas d'usage, séquence connexion, séquence recette)
   - ADRs (5 décisions architecturales avec contexte / option / conséquences)
   - Convention Git
   - Captures écran + maquettes responsive
3. **Cibles différentes** : le README s'adresse à un développeur qui rejoint le projet ; les ADRs s'adressent à un futur lui-même ou un mainteneur qui veut comprendre **pourquoi** une décision a été prise.

**Ancres** — `documentation/soutenance/backend/uml/`, `documentation/soutenance/backend/adr/`, README de chaque repo.

**Pièges**
- ❌ Ne pas savoir où sont les fichiers → fatal. Avoir l'arborescence en tête.

---

## Q12. Quels outils ou bibliothèques vous ont sauvé la vie ?

**Intention jury** — Question légère, mais permet de voir la curiosité et l'écosystème connu.

**Ossature de réponse** — Choisir 2-3 vrais :

- **bcrypt** côté back — pour ne pas réinventer le hashage de mots de passe (et ne pas se planter).
- **`node:test` + assert** — pour valider que TypeScript a tenu ses promesses (typage) et que la logique métier est correcte.
- **Postman** — pour valider les endpoints à la main avant chaque commit.
- **Angular CLI** — pour le scaffolding et le serveur de dev avec hot reload.

**Pièges**
- ❌ Citer une lib qu'on n'a en fait pas utilisée.

---

## Q13. Y a-t-il une partie de votre code dont vous êtes particulièrement fier ?

**Intention jury** — Voir où Arthur place son curseur, et l'amener à parler d'un sujet qu'il maîtrise.

**Ossature de réponse** — Choisir **un seul point** et le développer :

Exemples :
- « Le pattern Repository côté back. C'est ce qui m'a permis de rester from-scratch sur la persistance tout en gardant des services testables — chaque repo a une interface et une impl MySQL, et au démarrage `app.ts` câble tout à la main. C'est l'illustration la plus claire de l'inversion de dépendance dans le projet. » (voir ADR-002)
- « La logique de slug en deux phases — d'abord je crée la recette pour récupérer l'ID, ensuite je calcule le slug à partir du titre + ID. Ça évite les collisions sans avoir à faire de retry. » (voir ADR-005)

**Pièges**
- ❌ Choisir un point que le jury va démolir.

---

## Q14. Quelles sont les fonctionnalités principales de votre application ?

**Intention jury** — Vérifier que toutes les exigences du cahier des charges sont couvertes.

**Ossature de réponse** — Aller dans l'ordre du cahier :

1. **Auth + rôles** — inscription, connexion, récupération de mot de passe, profil, rôle admin.
2. **CRUD recettes** — création, édition, suppression (soft-delete), affichage.
3. **Recherche avancée** — par ingrédients, type de cuisine, temps de préparation.
4. **Interactions sociales** — commentaires, favoris.
5. **Modération** — l'admin peut supprimer ou désactiver une recette / un commentaire ; log de modération.
6. **Responsivité** — testée sur mobile / tablette / desktop.
7. **Accessibilité** — ARIA, contrastes, navigation clavier.

**Ancres** — pouvoir associer chaque fonctionnalité à un endpoint + une page Angular.

---

## Q15. Qu'avez-vous appris sur vous-même en faisant ce projet ?

**Intention jury** — Question parfois posée à la fin pour clore. Permet de montrer la maturité.

**Ossature de réponse** — Un point technique + un point soft :

- Technique : « J'ai compris que la contrainte from-scratch n'était pas un handicap mais un cadre qui m'a forcé à comprendre ce que les frameworks cachent normalement — l'injection de dépendances à la main dans `app.ts`, c'est devenu mon référentiel mental. »
- Soft : « J'ai sous-estimé le temps de la documentation. La prochaine fois je commencerai le README et les ADRs avant d'avoir terminé le code. »

**Pièges**
- ❌ « J'ai appris à coder. » → trop vague.
- ❌ Réponse purement personnelle (« je suis plus patient ») → on s'éloigne du contenu.
