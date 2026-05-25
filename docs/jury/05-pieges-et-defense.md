# 05 — Questions pièges et défense d'architecture

Les questions de cette section sont **adversariales** : le jury teste si Arthur peut **défendre ses choix** sous pression, ou s'il cède au premier contre-argument. La règle d'or : **ne jamais reculer sans raison**, **ne jamais s'entêter sans raison non plus**. Reconnaître les limites quand elles sont réelles, défendre quand c'est défendable.

---

## Q1. Express est un framework. Vous n'avez donc pas respecté la contrainte « from-scratch ».

**Intention jury** — Tester la fermeté et la nuance.

**Comment répondre**

> « Le cahier des charges dit "sans utiliser de frameworks **ou de librairies prédéfinies**". Le mot "framework" couvre une famille très large — il faut préciser. Express est un *micro-framework de routing HTTP*, comparable à Flask en Python ou Slim en PHP, qui sont autorisés par le cahier. Il **n'impose ni architecture, ni modèle de données, ni scaffolding, ni ORM**. Mes contrôleurs, services, repositories, DTOs et mappers sont tous écrits à la main, et l'injection des dépendances est faite manuellement dans `app.ts`. La seule chose qu'Express me fait gagner, c'est le parsing de requête HTTP et le routing — l'alternative étant `http.createServer()` natif, ce qui m'aurait poussé à réécrire un mini-Express avec plus de bugs. Mon ADR-001 documente cette décision avec les alternatives considérées. »

**Filet de sécurité** — Si le jury insiste : « Je comprends votre point. Pour aller plus loin, je peux vous montrer `app.ts` où vous verrez qu'il n'y a aucune des automatisations qui caractérisent les vrais frameworks applicatifs comme NestJS ou Symfony — pas de container DI, pas de décorateurs métier, pas de scaffolding CLI. »

**Ancres** — ouvrir `app.ts` si on doute, citer ADR-001 et la narration from-scratch.

**Ce qu'il ne faut PAS faire**
- ❌ Reculer en disant « ah oui en fait Express c'est un framework, j'aurais dû faire autrement ». Le jury va flairer le manque de conviction.
- ❌ S'agacer. Rester calme et factuel.

---

## Q2. Pourquoi ne pas avoir utilisé un ORM ? Ça aurait été plus propre et plus sûr.

**Intention jury** — Tester si Arthur connaît la valeur d'un ORM.

**Comment répondre**

> « Un ORM est précisément le type de bibliothèque que le cahier des charges interdit pour le Bloc 2 — il scaffolde les entités, génère les requêtes, gère les migrations. Pédagogiquement, écrire les requêtes à la main m'a forcé à comprendre les jointures, les index, les transactions, ce qu'un ORM cache normalement. En production sur un projet à plus large échelle, je prendrais probablement Prisma ou Drizzle pour la productivité et la sécurité de typage. Mais pour ce projet pédagogique, l'ORM aurait court-circuité l'apprentissage. »

**Ce qu'il ne faut PAS faire**
- ❌ Critiquer les ORMs en disant qu'ils sont mauvais.
- ❌ Prétendre que mysql2 est un ORM (c'est un driver).

---

## Q3. Pourquoi JWT et pas une session classique côté serveur ?

**Intention jury** — Vrai débat technique. Le jury veut voir qu'Arthur connaît les deux.

**Comment répondre**

> « Les deux approches sont valables. Les sessions côté serveur sont plus simples à révoquer (on supprime la session), mais elles imposent un état partagé serveur (cookie stocké en DB ou Redis), ce qui complique le scaling horizontal. Les JWT sont **stateless** — pas de stockage côté serveur, le token contient l'info nécessaire à la validation — mais ils sont **difficiles à révoquer avant expiration** (sauf à maintenir une blacklist, ce qui ramène l'état serveur). J'ai choisi JWT par souci de simplicité côté implémentation et parce que la révocation différée (~1 h via l'expiration courte) est acceptable pour ce contexte. Si je devais reprendre, je rajouterais des refresh tokens pour un meilleur compromis sécurité/UX. »

**Ce qu'il ne faut PAS faire**
- ❌ Affirmer que JWT est universellement meilleur que sessions.

---

## Q4. Avec JWT en cookie HttpOnly, comment faites-vous la déconnexion ?

**Intention jury** — Question subtile. Comme le JWT est stateless, on ne peut pas « le supprimer côté serveur ».

**Comment répondre**

> « La déconnexion côté front consiste à **effacer le cookie** en envoyant un `Set-Cookie` avec une date d'expiration passée et la même config (HttpOnly, Secure, SameSite). Le navigateur supprime alors le cookie. Le JWT reste théoriquement valide jusqu'à son expiration s'il a été intercepté, mais comme il était uniquement dans le cookie HttpOnly (pas accessible par JS), le risque est faible. Pour une révocation immédiate plus stricte, il faudrait maintenir une liste noire de JWT révoqués — ce que je n'ai pas implémenté car ça réintroduit l'état serveur. »

**Ancres** — endpoint `POST /auth/logout` dans le back.

---

## Q5. Vos requêtes SQL dans les repositories — pourquoi pas de query builder ?

**Intention jury** — Voir si Arthur a réfléchi à la lisibilité.

**Comment répondre**

> « Un query builder type Knex aurait été un compromis intéressant entre SQL brut et ORM, mais il reste une **librairie d'abstraction** qui rentre dans la zone grise du cahier des charges. J'ai préféré écrire le SQL en clair pour rester sans ambiguïté, et parce que pour les requêtes que j'ai, le SQL natif est plus lisible et plus performant. Pour les recherches multi-critères (la seule partie où la requête est dynamique), je construis la clause `WHERE` par concaténation de fragments **paramétrés** dans un tableau — pas joli mais sûr. »

---

## Q6. Pourquoi ne pas avoir mis en place de CI/CD ?

**Intention jury** — Maturité DevOps.

**Comment répondre**

> « Je n'ai pas mis en place de CI/CD complète. Côté local, j'ai des hooks Husky qui lancent le lint et les tests avant chaque commit, ce qui couvre la première barrière. Pour un projet en équipe ou en prod, je rajouterais sans hésiter un workflow GitHub Actions qui (1) lint + teste à chaque push, (2) bloque le merge sur PR si les tests cassent, (3) déploie automatiquement la branche `main`. C'est une amélioration que je placerais en première priorité après la soutenance. »

**Pièges**
- ❌ Ne pas savoir ce qu'est un GitHub Action.

---

## Q7. Pourquoi 3 repos GitHub et pas un monorepo ?

**Intention jury** — Choix d'organisation.

**Comment répondre**

> « C'est un compromis. Un monorepo (avec npm workspaces, Nx ou Turborepo) aurait permis le partage de types entre front et back, et un build unifié. Trois repos séparés gardent les responsabilités étanches, simplifient le CI par repo, et reflètent mieux les trois blocs distincts de la certification — chaque repo est un livrable autonome. Pour un projet pédagogique, je trouvais que la séparation rendait mes intentions plus lisibles ; pour un projet d'équipe avec partage de types fort, je choisirais probablement le monorepo. »

---

## Q8. Vos services dépendent d'interfaces de repository, mais vous n'avez qu'une seule implémentation. C'est de l'over-engineering, non ?

**Intention jury** — Tester si Arthur peut défendre une abstraction.

**Comment répondre**

> « L'argument est légitime. Mais l'interface a deux bénéfices concrets, même avec une seule implémentation : **un**, elle me permet de tester les services avec un repo en mémoire ou un mock, sans toucher à MySQL — c'est mon test runner natif `node:test` qui le fait. **Deux**, elle force la séparation conceptuelle : le service ne *sait* pas qu'il y a MySQL derrière, ce qui m'oblige à raisonner en domaine, pas en SQL. Le coût est marginal — une interface, c'est dix lignes de TS. C'est documenté dans mon ADR-002. »

**Ancres** — [`_draft_adr/adr-002-pattern-repository-interface-impl.md`](../soutenance/backend/adr/adr-002-pattern-repository-interface-impl.md).

---

## Q9. Vous n'avez pas de tests d'intégration. C'est embêtant, non ?

**Intention jury** — Tester la lucidité.

**Comment répondre**

> « C'est une limite réelle. J'ai privilégié la couverture unitaire (~40 fichiers) parce que c'est ce qui m'a aidé à itérer rapidement pendant le dev. Pour un projet qui irait en production, j'ajouterais une couche d'intégration avec une base MySQL de test (instance Docker spawnée par le runner), couvrant au minimum les flows critiques : inscription, connexion, création de recette, modération. Playwright pour de l'E2E navigateur ferait sens aussi pour valider le parcours utilisateur de bout en bout. »

**Ce qu'il ne faut PAS faire**
- ❌ Nier la limite. Le jury sait que c'est une limite, mieux vaut l'assumer et montrer qu'on sait comment y remédier.

---

## Q10. Votre projet n'est pas accessible en ligne. Comment le jury peut-il le tester ?

**Intention jury** — **Cassant**. Le cahier des charges exige une démo en ligne.

**Comment répondre**

> *(Si le déploiement n'est PAS prêt au jour J)* « Je n'ai pas réussi à déployer en ligne dans les délais — c'est une vraie lacune que j'assume. J'ai préparé un environnement local démonstrable : avec votre permission je peux le lancer en direct et vous faire la démo. Je peux aussi vous montrer les scripts de déploiement préparés pour `[plateforme]` et expliquer pourquoi je n'ai pas pu finaliser. »
>
> *(Si le déploiement EST prêt)* « Le front est sur `[URL]`, l'API est sur `[URL]`, voici un compte de démo : `[credentials]`. »

> ⚠ **TODO Arthur (CRITIQUE)** : ce point est **éliminatoire**. À traiter en absolue priorité (chantier #9 de la roadmap).

---

## Q11. Vous prétendez que votre code est testable, mais votre couverture est-elle vraiment bonne ?

**Intention jury** — Tester si Arthur a vraiment mesuré.

**Comment répondre**

> *(Si la couverture est mesurée)* « Oui, j'ai un rapport de couverture généré par `c8` / `node:test --coverage`. Sur le back, c'est autour de **X %**, avec les services à **Y %** et les repositories à **Z %**. Les zones les moins couvertes sont les middlewares techniques et `app.ts` qui sont plus testés via les tests d'intégration que je n'ai pas. »
>
> *(Si non mesurée)* « Honnêtement, je n'ai pas généré de rapport de couverture systématique. Je sais que j'ai ~40 fichiers de test couvrant toutes les couches métier, mais je ne peux pas vous donner un chiffre précis. Lancer `node --test --experimental-test-coverage` me donnerait ce rapport — je peux le faire en direct si vous voulez. »

**Pièges**
- ❌ Inventer un chiffre.

---

## Q12. Vous n'avez pas implémenté la pagination. Comment ça scale ?

**Intention jury** — Performance et passage à l'échelle.

**Comment répondre**

> *(Si pagination implémentée)* « Si, j'ai mis de la pagination par `LIMIT/OFFSET` sur l'endpoint de liste — par défaut 20 par page, configurable par paramètre. Pour des volumes très grands (> 100k), je passerais à une pagination par cursor (sur un `id` croissant) qui ne souffre pas du coût croissant d'`OFFSET`. »
>
> *(Si non implémentée)* « Bonne remarque, ce n'est pas implémenté. Pour un site de recettes avec quelques milliers de recettes, ça reste viable mais ce n'est pas scalable. J'ajouterais un `LIMIT/OFFSET` en première itération, et un cursor sur l'`id` au-delà de quelques dizaines de milliers d'entrées. »

---

## Q13. Et si demain vous deviez ajouter le rôle « modérateur » entre user et admin ?

**Intention jury** — Tester l'adaptabilité (cahier des charges : « modifier son code en temps réel »).

**Comment répondre** (en montrant le code idéalement)

> « Trois endroits à toucher :
> 1. **Table `users`** : la colonne `role` est déjà un string, donc je rajoute simplement la valeur `moderator` au runtime — pas de migration de schéma nécessaire. Si c'était un ENUM strict, je rajouterais la valeur via une migration.
> 2. **Le JWT** contient déjà `role`, donc rien à faire côté token.
> 3. **Les middlewares d'autorisation** : je rajoute `requireRole('moderator')` ou je raffine `requireRole(['moderator', 'admin'])` sur les endpoints de modération qui doivent être accessibles aux deux. »
>
> *(Si on demande de coder en direct)* — ouvrir le middleware d'authz et faire la modif. Préparer mentalement le chemin du fichier.

---

## Q14. Vous n'avez pas mentionné Docker. Pourquoi ?

**Intention jury** — DevOps.

**Comment répondre**

> *(Si pas de Docker)* « Je n'ai pas containerisé le projet — j'aurais pu, mais ça aurait ajouté de la complexité de setup local sans bénéfice immédiat pour le développement seul. En revanche, pour le déploiement et la reproductibilité d'environnement, Docker est un standard, et je l'ajouterais dans un Dockerfile multistage pour le back et un autre pour le front. »
>
> *(Si Docker présent)* — décrire le Dockerfile et docker-compose.

---

## Q15. Si je vous demande d'ajouter une fonctionnalité maintenant, là, devant nous, vous le faites comment ?

**Intention jury** — Test d'adaptation en direct (explicitement exigé par le cahier des charges).

**Comment répondre** — Ne pas paniquer, suivre une démarche claire :

1. **Reformuler la demande** : « Si je comprends bien, vous voulez que… [reformuler]. C'est bien ça ? »
2. **Identifier la couche concernée** : « C'est une nouvelle règle métier → je vais devoir modifier le service `XxxService`. Si c'est aussi une nouvelle persistance, j'ajoute une méthode au repository. »
3. **Annoncer le plan** : « Je vais (1) ajouter la méthode dans le repo, (2) la méthode métier dans le service, (3) un test, (4) l'endpoint dans le contrôleur, (5) le DTO si nécessaire. »
4. **Coder calmement** : commencer par la signature, faire compiler, puis remplir.
5. **Tester** : lancer le test, ouvrir Postman pour appeler l'endpoint si possible.

**Ce qu'il ne faut PAS faire**
- ❌ Se précipiter et taper du code à l'aveugle.
- ❌ Dire « je ne sais pas » sans même essayer.
- ❌ Coder en silence — **expliquer ce qu'on fait** à voix haute.
