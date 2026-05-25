# 02 — Bloc 2 : Back-End from scratch

**Le bloc le plus critique du dossier.** Le cahier des charges exige explicitement « **développer le back-end sans utiliser de frameworks ou de librairies prédéfinies** » et « **utiliser les concepts liés à la POO** ». Le jury va creuser ici.

> 📌 **Pierre angulaire de défense** : [`_draft_adr/00-narration-from-scratch.md`](../soutenance/backend/adr/00-narration-from-scratch.md). À relire 3 fois avant la soutenance.

---

## Q1. Vous dites avoir codé le back-end « from scratch ». Mais vous utilisez Express. Comment justifiez-vous cela ?

**Intention jury** — La question piège n°1 du dossier. Si Arthur bafouille ici, le reste du Bloc 2 sera examiné à la loupe. Voir aussi [`05-pieges-et-defense.md`](05-pieges-et-defense.md) pour la version étendue.

**Ossature de réponse**

1. **Distinguer deux niveaux** :
   - **Framework applicatif** (Symfony, NestJS, Rails) : impose une structure MVC, un container DI, des conventions de scaffolding, un ORM intégré → **non utilisé**.
   - **Couche utilitaire HTTP** (Express, http natif de Node) : un simple parseur de requêtes et un routeur ; ne dicte ni l'architecture, ni le modèle métier, ni la persistance.
2. **Le cahier des charges** demande de coder « contrôleurs, modèles et vues » from-scratch. Tout cela est écrit à la main dans le projet : contrôleurs, services, repositories, mappers, DTOs, middlewares. Express n'écrit aucune de ces couches.
3. **L'alternative honnête** : `http.createServer()` natif. Je l'ai considéré, mais ça revenait à réécrire le parsing JSON, le routing, la gestion des cookies, le multipart — autant de roues réinventées qui n'apportent rien à l'apprentissage et augmentent le risque de bugs de sécurité.
4. **Preuve par le code** : ouvrir `app.ts` et montrer que **chaque dépendance est instanciée à la main**. Pas de container DI, pas de `@Injectable`, pas de scaffolding. C'est l'illustration la plus claire que l'architecture est faite-main.

**Ancres**
- [`_draft_adr/00-narration-from-scratch.md`](../soutenance/backend/adr/00-narration-from-scratch.md)
- [`_draft_adr/adr-001-express-comme-couche-http.md`](../soutenance/backend/adr/adr-001-express-comme-couche-http.md)
- `backend/src/app.ts` (à ouvrir pendant la soutenance si la question vient)

**Pièges**
- ❌ Dire « Express c'est pas un framework » → faux, c'est un micro-framework. Mieux : « C'est un framework utilitaire de routing HTTP, pas un framework applicatif. »
- ❌ Dire « le formateur m'a dit que c'était ok » → infantilisant, et le jury n'est pas le formateur.

---

## Q2. Quels concepts de POO avez-vous utilisés ?

**Intention jury** — Cahier des charges : « utiliser les concepts liés à la POO ». Faut pouvoir nommer et illustrer.

**Ossature de réponse**

1. **Encapsulation** : les services exposent des méthodes publiques (`createRecipe`, `findById`) ; leur état (le repository injecté) est privé.
2. **Abstraction par interface** : chaque repository est défini par une interface (`RecipeRepository`) et implémenté par une classe concrète (`MysqlRecipeRepository`). Les services dépendent de l'interface, pas de l'impl → **inversion de dépendance**.
3. **Composition plutôt qu'héritage** : pas de hiérarchies de classes profondes ; les services reçoivent leurs collaborateurs par constructeur.
4. **Polymorphisme** : implicite via les interfaces — un service ne sait pas si son repo est MySQL ou un mock de test.
5. **Classes vs fonctions** : les services et repositories sont des **classes** (état + méthodes), les mappers et utils sont des fonctions pures (pas d'état).

**Ancres**
- `backend/src/.../services/recipeService.ts` (ou équivalent)
- Une interface de repository + son impl MySQL

**Pièges**
- ❌ Citer « héritage / polymorphisme / encapsulation » par cœur sans pouvoir donner un exemple **dans le code Recipe Shelter**.
- ❌ Dire « tout est en classe » alors qu'on utilise des fonctions pures aussi — la composition fonctionnelle est compatible avec POO et c'est mature de le reconnaître.

---

## Q3. Pouvez-vous nous expliquer votre architecture en couches ?

**Intention jury** — Vérifier la séparation des responsabilités.

**Ossature de réponse**

```
┌─────────────────────────────────────────────┐
│ HTTP layer (Express routes)                 │  ← parse request, valide via DTO, appelle controller
├─────────────────────────────────────────────┤
│ Controllers                                 │  ← orchestrent, traduisent input → service, output → HTTP
├─────────────────────────────────────────────┤
│ Services (logique métier)                   │  ← règles fonctionnelles, transactions, validation domaine
├─────────────────────────────────────────────┤
│ Repositories (persistance)                  │  ← seul endroit qui parle SQL
├─────────────────────────────────────────────┤
│ MySQL (via mysql2)                          │  ← driver natif, requêtes paramétrées
└─────────────────────────────────────────────┘
                ↕
       Mappers : DB row ↔ Domain object
       DTOs : payload HTTP → object validé
```

**Règles invariantes** :
- Aucun service ne touche directement à la DB.
- Aucun contrôleur ne contient de logique métier.
- Aucun repo ne fait de validation fonctionnelle.

**Ancres** — `backend/src/app.ts` (le câblage), [`_draft_uml/02-architecture-classes.md`](../soutenance/backend/uml/02-diagramme-classes-architecture.md) ou équivalent.

**Pièges**
- ❌ Dessiner les couches mais avoir, en pratique, du SQL dans le contrôleur → ouvrir le code et montrer la séparation.

---

## Q4. Pourquoi un pattern Repository plutôt que d'écrire le SQL directement dans les services ?

**Intention jury** — Justifier le surcoût d'abstraction.

**Ossature de réponse**

1. **Testabilité** : en tests unitaires, on remplace l'impl MySQL par un repo en mémoire (ou un mock). Les services se testent sans DB.
2. **Substituabilité** : si demain je remplace MySQL par PostgreSQL ou une base in-memory, seul le repo change.
3. **Centralisation des requêtes** : toutes les requêtes SQL d'une entité sont dans un seul fichier. Plus facile à auditer pour les perfs ou la sécu (vérifier que tout est paramétré).
4. **Séparation des soucis** : le service raisonne en objets domaine (`Recipe`, `User`), pas en lignes SQL.

**Ancres** — [`_draft_adr/adr-002-pattern-repository-interface-impl.md`](../soutenance/backend/adr/adr-002-pattern-repository-interface-impl.md).

**Pièges**
- ❌ « C'est plus propre » sans justification concrète → mou.

---

## Q5. Pourquoi avoir choisi mysql2 plutôt qu'un ORM (TypeORM, Prisma, Sequelize) ?

**Intention jury** — Comprendre que ce n'est pas par méconnaissance.

**Ossature de réponse**

1. **Contrainte cahier des charges** : pas de framework / pas de librairie prédéfinie pour la couche métier. Un ORM est exactement ça — il scaffolde les modèles, génère les requêtes, gère les relations.
2. **Connaître ce que l'ORM cache** : écrire les requêtes à la main force à comprendre les joins, les index, les transactions. Pédagogiquement bien plus formateur.
3. **mysql2 est un driver, pas un ORM** : il convertit le JS en protocole MySQL et retourne les lignes. Il ne génère aucune requête.
4. **Conséquence assumée** : plus de code à écrire (chaque repo a son `SELECT`, `INSERT`, `UPDATE`, `DELETE`). En prod sur un projet de taille industrielle, je prendrais probablement Prisma ou Drizzle.

**Pièges**
- ❌ Dire que les ORMs sont mauvais → contre-productif et faux.

---

## Q6. Comment évitez-vous les injections SQL ?

**Intention jury** — Sécurité de base, exigée.

**Ossature de réponse**

1. **Requêtes paramétrées systématiques** : `mysql2` accepte les `?` placeholders, et le driver échappe les valeurs avant de les passer à MySQL.
   ```ts
   await db.query('SELECT * FROM recipes WHERE id = ?', [id]);
   ```
2. **Jamais de concaténation** de strings utilisateur dans une requête.
3. **Validation des types** en amont via DTO → un `id` est garanti `number` avant d'atteindre le repo.
4. **Si requête dynamique** (recherche multi-critères) : construction de la clause `WHERE` par fragments paramétrés, jamais en injectant des valeurs en string.

**Ancres** — ouvrir un repo MySQL et montrer un `db.query(..., [params])`.

**Pièges**
- ❌ « J'utilise un ORM donc je suis safe » → faux ici car pas d'ORM, et même un ORM peut être détourné.

---

## Q7. Expliquez votre système d'authentification.

**Intention jury** — JWT, cookies, sessions — c'est le cœur de la sécurité applicative.

**Ossature de réponse**

1. **Inscription** : le mot de passe est hashé avec **bcrypt** (cost factor 10 ou 12). Seul le hash est stocké.
2. **Connexion** : on récupère l'utilisateur par email, on vérifie le mot de passe avec `bcrypt.compare`. Si OK, on génère un **JWT** signé avec une clé secrète (variable d'env).
3. **Transport du JWT** : envoyé dans un **cookie HttpOnly, Secure, SameSite=Strict (ou Lax)**, **pas** dans `localStorage`. C'est résistant au XSS car le JS du navigateur ne peut pas y accéder.
4. **Vérification** : middleware Express qui, à chaque requête protégée, lit le cookie, vérifie la signature du JWT, attache l'utilisateur à `req.user`.
5. **Expiration** : durée de vie limitée (ex : 1 h). Au-delà, l'utilisateur se reconnecte. *(Si refresh token implémenté, le dire ; sinon, l'assumer comme limite.)*
6. **Rôles** : le JWT contient le rôle (`user` ou `admin`). Un autre middleware vérifie `req.user.role === 'admin'` sur les routes admin.

**Ancres**
- [`_draft_adr/adr-003-jwt-cookie-httponly.md`](../soutenance/backend/adr/adr-003-jwt-cookie-httponly.md)
- [`_draft_uml/04-diagramme-sequence-connexion.md`](../soutenance/backend/uml/04-diagramme-sequence-connexion.md)
- Middleware d'auth dans le back

**Pièges**
- ❌ Dire « j'ai mis le JWT dans le localStorage » → si c'est ça le code, mieux vaut dire pourquoi et reconnaître que HttpOnly est l'option recommandée.
- ❌ Ne pas savoir ce qu'est un JWT (header.payload.signature).

---

## Q8. Pourquoi bcrypt et pas SHA-256 ou MD5 ?

**Intention jury** — Comprend-il la différence entre hash crypto et hash de mots de passe ?

**Ossature de réponse**

1. **MD5 et SHA-256 sont rapides** → un attaquant qui dump la base peut tester des milliards de combinaisons par seconde sur GPU.
2. **bcrypt est lent par design** : un *cost factor* configurable rend le hash coûteux (~100 ms par tentative à cost 10).
3. **bcrypt intègre un sel aléatoire par hash** → deux utilisateurs avec le même mot de passe ont des hash différents, ce qui empêche les rainbow tables.
4. **Alternatives modernes** : Argon2 (recommandé par l'OWASP depuis 2015 environ), scrypt. bcrypt reste largement acceptable.

**Pièges**
- ❌ Confondre « chiffrement » et « hashage » (le hash est à sens unique, le chiffrement est réversible).

---

## Q9. Que fait votre middleware d'erreur ?

**Intention jury** — Vérifier qu'il y a une gestion centralisée.

**Ossature de réponse**

1. **Signature Express** : `(err, req, res, next) => void`. Le 4e argument est ce qui en fait un middleware d'erreur.
2. **Pattern** : on `throw` des erreurs typées dans les services (`NotFoundError`, `UnauthorizedError`, `ValidationError`). Le middleware les `instanceof`-teste et mappe vers le bon code HTTP.
3. **Fallback** : si l'erreur n'est pas reconnue, on log côté serveur et on renvoie un 500 générique au client (pas de stack trace exposée).
4. **Format de réponse** : `{ error: { code, message } }` pour que le front puisse traiter de façon homogène.

**Ancres** — ouvrir le fichier `middlewares/errorHandler.ts` (ou son équivalent).

**Pièges**
- ❌ Renvoyer la stack au client → fuite d'info.

---

## Q10. Comment validez-vous les données entrantes ?

**Intention jury** — Robustesse + sécurité.

**Ossature de réponse**

1. **DTOs en TypeScript** : chaque endpoint a un DTO qui décrit la forme attendue.
2. **Validation programmatique** : fonction de validation par DTO qui vérifie présence, type, longueur, format. Tests unitaires sur ces validations.
3. **Côté front aussi** : Reactive Forms Angular pour le confort utilisateur, mais **la vérité reste côté back**.
4. **Pas d'utilisation de Joi/Zod** car libs externes — choix conscient pour rester from-scratch. Conséquence : un peu de code de validation à maintenir.

**Ancres** — un DTO + son test unitaire.

**Pièges**
- ❌ Dire qu'on fait confiance au front.

---

## Q11. Décrivez votre schéma de base de données.

**Intention jury** — Vérifier la maîtrise du modèle de données. Le cahier des charges exige un **script SQL** et des **schémas fonctionnels**.

**Ossature de réponse**

Lister les tables principales :
- `users` (id, email unique, password_hash, role, created_at, ...)
- `recipes` (id, title, slug unique, description, prep_time, cook_time, author_id FK, status, created_at, deleted_at)
- `ingredients` (id, name) — ou table de liaison `recipe_ingredients`
- `comments` (id, recipe_id FK, author_id FK, content, created_at, deleted_at)
- `favorites` (user_id FK, recipe_id FK) — clé primaire composite
- `categories` / `tags` selon ce qui est implémenté
- `moderation_logs` (id, action, target_type, target_id, moderator_id, reason, created_at)

**Points à savoir expliquer** :
- **Index** : sur les colonnes de recherche (slug, email), sur les FK.
- **Contraintes** : `UNIQUE` sur email et slug, `ON DELETE` strategy (cascade ou SET NULL selon la table).
- **Soft-delete** : `deleted_at NULL` par défaut, set à un timestamp pour soft-delete (voir ADR-004).

**Ancres** — `backend/database/` (scripts SQL), [`_draft_uml/01-diagramme-classes-domaine.md`](../soutenance/backend/uml/01-diagramme-classes-domaine.md).

**Pièges**
- ❌ Ne pas savoir le nom exact d'une de ses tables → fatal.
- ❌ Ne pas savoir ce qu'est un index ou pourquoi on en met.

---

## Q12. Avez-vous des index sur votre base ? Lesquels et pourquoi ?

**Intention jury** — Conscience perf.

**Ossature de réponse**

1. **Index primaire** sur `id` (auto, AUTO_INCREMENT).
2. **Index unique** sur `users.email` (et `recipes.slug`) → unicité + accélération des `WHERE email = ?`.
3. **Index sur FK** : `recipes.author_id`, `comments.recipe_id`, etc. → indispensables pour les jointures.
4. **Index composite** éventuel sur les colonnes de recherche multi-critères (ex : `recipes(category_id, prep_time)`).
5. **Compromis** : un index accélère les lectures mais ralentit les écritures → ne pas en mettre partout.

**Pièges**
- ❌ « MySQL met les index automatiquement » → seulement le PK, pas les FK ni les index secondaires.

---

## Q13. Comment fonctionne votre système de slug ?

**Intention jury** — Détail technique, mais c'est un ADR du dossier (preuve de réflexion).

**Ossature de réponse**

1. **Pourquoi un slug** : URL lisible (`/recipes/tarte-aux-pommes-42` plutôt que `/recipes/42`), bon pour le SEO.
2. **Algorithme deux phases** (ADR-005) :
   - Phase 1 : on insère la recette en base, on récupère l'ID auto-incrémenté.
   - Phase 2 : on calcule `slug = slugify(title) + '-' + id`, on update la ligne.
3. **Avantage** : pas de collision (l'ID est unique par nature), pas de retry.
4. **Compromis** : deux requêtes au lieu d'une — acceptable car création de recette est rare comparée à la lecture.

**Ancres** — [`_draft_adr/adr-005-slug-en-deux-phases.md`](../soutenance/backend/adr/adr-005-slug-en-deux-phases.md).

---

## Q14. Comment gérez-vous la suppression des recettes ?

**Intention jury** — Lien avec la modération + bonnes pratiques.

**Ossature de réponse**

1. **Soft-delete** : on set `deleted_at = NOW()` plutôt que de faire un `DELETE`.
2. **Pourquoi** :
   - Préserver l'historique pour la modération et l'audit.
   - Permettre la restauration.
   - Préserver les commentaires liés sans casser les FK.
3. **Lecture** : toutes les requêtes filtrent `WHERE deleted_at IS NULL`.
4. **Log de modération** : chaque action admin (suppression, désactivation) est journalisée dans `moderation_logs` avec auteur + motif.

**Ancres** — [`_draft_adr/adr-004-soft-delete-et-log-moderation.md`](../soutenance/backend/adr/adr-004-soft-delete-et-log-moderation.md).

**Pièges**
- ❌ Conflit RGPD potentiel — voir question sécurité Q11 ([`04-securite-rgpd.md`](04-securite-rgpd.md)) : le droit à l'effacement implique parfois un `DELETE` réel.

---

## Q15. Comment fonctionne la recherche avancée ?

**Intention jury** — Cahier des charges : « recherche par ingrédients, type de cuisine, temps de préparation ».

**Ossature de réponse**

1. **Endpoint** unique `/api/recipes?ingredients=...&category=...&maxPrepTime=...`.
2. **Construction dynamique de la requête** côté back : on part d'un `SELECT ... WHERE 1=1`, et on **ajoute des clauses paramétrées** selon les filtres présents.
3. **Recherche par ingrédients** : jointure sur la table de liaison, `WHERE ingredient_id IN (...)`. Logique « tous les ingrédients » ou « au moins un » selon le besoin.
4. **Pagination** : `LIMIT ? OFFSET ?` (ou cursor-based si volumes grands).
5. **Tri** : white-list des colonnes triables côté back pour éviter l'injection via paramètre.

**Pièges**
- ❌ Injecter le nom de colonne de tri directement depuis le query string sans white-list → injection SQL.

---

## Q16. Quelle est la stratégie de tests côté back ?

**Intention jury** — Maturité qualité.

**Ossature de réponse**

1. **Runner** : `node:test` natif (depuis Node 18 stable, 20 sans flag) + `node:assert`. Choix from-scratch — pas de Jest, pas de Vitest.
2. **Couverture** : ~40 fichiers de tests sur :
   - **DTOs** — validation des payloads.
   - **Services** — logique métier en mockant les repositories.
   - **Middlewares** — auth, error handler, validation.
   - **Mappers** — DB row → domain object.
3. **Pas de tests d'intégration en DB réelle** (limite assumée — exécutés sur impl in-memory du repo).
4. **Exécution** : `npm test` lance tout. Hook Husky pre-commit empêche de commit du code qui casse les tests.

**Ancres** — `backend/tests/`, un fichier de test représentatif à ouvrir.

**Pièges**
- ❌ Pas connaître la commande pour lancer les tests.

---

## Q17. Pourquoi TypeScript et pas du JavaScript pur ?

**Intention jury** — Choix de typage.

**Ossature de réponse**

1. **Sécurité au compile-time** : attrape ~80 % des bugs naïfs avant l'exécution.
2. **Documentation vivante** : les types remplacent une grande partie des commentaires.
3. **Refactoring** : renommer une propriété propage automatiquement.
4. **Choix neutre vs from-scratch** : TypeScript est un *compilateur*, pas un framework. Il ne dicte ni l'architecture ni les abstractions métier. Le cahier des charges autorise Node.js, et TS est compilé en JS Node compatible.

**Pièges**
- ❌ Dire « TypeScript m'évite les bugs » sans nuancer → ça attrape les bugs de typage, pas les bugs logiques.

---

## Q18. Comment gérez-vous les variables d'environnement et les secrets ?

**Intention jury** — Pratiques DevOps de base.

**Ossature de réponse**

1. **`.env` non versionné** (`.gitignore`), un **`.env.example`** versionné qui liste les clés attendues sans les valeurs.
2. **Chargement via `dotenv`** au démarrage de l'app.
3. **Secrets sensibles** : `JWT_SECRET`, `DB_PASSWORD`, `SMTP_PASSWORD` → injectés au déploiement, jamais en clair dans le code ni dans Git.
4. **Validation au démarrage** : si une variable critique manque, l'app crashe avec un message clair plutôt que de démarrer dans un état dégradé.

**Pièges**
- ❌ Avoir un secret en clair dans un commit Git → si c'est le cas, mieux vaut l'admettre et expliquer le plan (rotation du secret, nettoyage de l'historique avec BFG).

---

## Q19. Décrivez le cycle de vie d'une requête HTTP type.

**Intention jury** — Comprendre la traversée de toutes les couches.

**Ossature de réponse** — Exemple : `POST /api/recipes` :

1. Requête arrive sur Express → middleware CORS, parser JSON, parser cookies.
2. Middleware d'auth : lit cookie JWT, vérifie signature, attache `req.user`. Si absent → 401.
3. Route `/api/recipes` → contrôleur `recipeController.create`.
4. Contrôleur : extrait `req.body`, le passe au DTO `CreateRecipeDTO` qui valide → si invalide, `throw ValidationError` → middleware d'erreur → 400.
5. Contrôleur appelle `recipeService.create(dto, userId)`.
6. Service : applique les règles métier (vérif auteur, slug en deux phases…), appelle `recipeRepository.create(...)`.
7. Repo : exécute `INSERT` MySQL via mysql2 paramétré.
8. Retour : repo → service → contrôleur → `res.status(201).json(recipe)`.
9. Si erreur à n'importe quel étage → middleware d'erreur → réponse normalisée.

**Ancres** — [`_draft_uml/05-diagramme-sequence-recette.md`](../soutenance/backend/uml/05-diagramme-sequence-recette.md).

---

## Q20. Avez-vous géré le CORS ? Pourquoi ?

**Intention jury** — Comprendre pourquoi il y a un middleware CORS et comment il est configuré.

**Ossature de réponse**

1. **Pourquoi** : le front Angular et l'API back sont sur deux origines différentes (deux ports en dev, deux sous-domaines en prod). Sans CORS, le navigateur bloque les requêtes XHR/fetch.
2. **Configuration** :
   - `origin` : whitelist explicite (l'URL du front), pas `*`.
   - `credentials: true` → indispensable car on envoie un cookie HttpOnly.
   - `methods` : limités à ceux nécessaires (`GET, POST, PUT, DELETE`).
3. **CORS != sécurité** : CORS est une politique navigateur, pas un pare-feu. La vraie sécurité reste dans l'auth + la validation.

**Pièges**
- ❌ Mettre `origin: '*'` avec `credentials: true` → contradictoire et rejeté par les navigateurs modernes.

---

## Q21. Comment fonctionne le système de mot de passe oublié ?

**Intention jury** — Récupération de mot de passe = vecteur d'attaque classique.

**Ossature de réponse**

1. **Endpoint `POST /auth/forgot-password`** : reçoit un email.
2. **Réponse identique que l'email existe ou non** → évite l'énumération d'emails.
3. Si l'email existe, on génère un **token unique aléatoire** (UUID ou random bytes), on le stocke en base avec une date d'expiration (ex : 1 h), et on envoie un email contenant le lien `/reset-password?token=...`.
4. **`POST /auth/reset-password`** : reçoit `{ token, newPassword }`. Vérifie que le token existe, n'est pas expiré, et n'a pas déjà été consommé. Hashe le nouveau mot de passe, met à jour l'utilisateur, **invalide le token**.
5. **Envoi d'email via nodemailer** — SMTP configuré en variable d'env.

**Pièges**
- ❌ Réutilisation du même token plusieurs fois → faille.
- ❌ Token court / prédictible.

---

## Q22. Quelle est la différence entre une authentification et une autorisation chez vous ?

**Intention jury** — Vocabulaire de sécurité.

**Ossature de réponse**

1. **Authentification (authn)** = *qui es-tu ?* → middleware qui vérifie le JWT et attache `req.user`.
2. **Autorisation (authz)** = *as-tu le droit de faire ça ?* → middleware suivant (ex : `requireRole('admin')`) qui vérifie le rôle, ou logique métier dans le service qui vérifie que `recipe.author_id === user.id` avant un update.
3. **Séparation claire** dans le code : deux middlewares distincts, pas un seul gros.

**Pièges**
- ❌ Confondre les deux.

---

## Q23. Pouvez-vous nous montrer une partie de votre code et l'expliquer ?

**Intention jury** — Demande très probable. Le jury choisira le fichier — ou laissera Arthur choisir.

**Ossature de réponse** — Préparer **3 candidats** à proposer si on laisse le choix :

1. **`app.ts`** : montre le câblage manuel des dépendances → preuve from-scratch.
2. **Un service** (ex : `recipeService.ts`) : montre la POO, l'injection par constructeur, une méthode métier.
3. **Le middleware d'auth** : montre la sécurité, l'usage du JWT, l'attachement de `req.user`.

Pour chaque fichier, savoir expliquer **ligne par ligne** sur les sections critiques.

**Pièges**
- ❌ Ouvrir un fichier qu'on n'a pas relu récemment.
- ❌ Choisir un fichier mal écrit pour faire le malin.

---

## Q24. Avez-vous mis en place du logging ? Comment ?

**Intention jury** — Observabilité.

**Ossature de réponse**

- *(Si simple console.log)* : « J'utilise `console.log` / `console.error` pour le moment, sans logger structuré. En prod je rajouterais Pino pour avoir des logs JSON, des niveaux configurables, et l'envoi vers un agrégateur (Loki, Datadog). »
- *(Si logger custom)* : décrire l'implémentation.

**Pièges**
- ❌ Dire qu'on a un logger structuré si c'est faux.

---

## Q25. Que se passe-t-il si votre base de données tombe ?

**Intention jury** — Vision opérationnelle.

**Ossature de réponse**

1. **Comportement actuel** : les requêtes échouent → le middleware d'erreur renvoie un 500. L'utilisateur voit un message d'erreur.
2. **Pas de retry automatique** ni de circuit breaker (limite assumée).
3. **Ce que je ferais en prod** :
   - Healthcheck endpoint `/health/db`.
   - Connection pool avec retry (mysql2 supporte les pools).
   - Backup régulier de la base.
   - Monitoring (Prometheus + Grafana) pour alerter.

**Pièges**
- ❌ Inventer des mécanismes qui n'existent pas dans le code.
