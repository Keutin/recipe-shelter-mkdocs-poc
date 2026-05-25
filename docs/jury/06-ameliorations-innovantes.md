# 06 — Améliorations innovantes

Le cahier des charges l'écrit noir sur blanc : « **le candidat soit force de proposition, apportant des idées innovantes ou des solutions améliorées lorsque sollicité par le jury**. » C'est presque toujours posé à la fin de l'entretien (« Comment iriez-vous plus loin ? »). Avoir 3-4 idées **prêtes à dégainer**, avec à chaque fois un bénéfice concret, le coût d'implémentation, et l'ordre de priorité.

---

## Q1. Comment iriez-vous plus loin sur la sécurité ?

**Comment répondre**

1. **Rate limiting** sur les endpoints sensibles (login, register, forgot-password) — middleware Express custom basé sur Redis ou en mémoire. Bloque les attaques par brute force.
2. **Headers de sécurité** : ajouter `helmet`-equivalent custom (CSP, HSTS, X-Frame-Options, X-Content-Type-Options).
3. **Refresh tokens** : accès court (15 min) + refresh long (7 jours), permet la révocation effective.
4. **2FA** (OTP par email ou TOTP type Google Authenticator) pour les comptes admin.
5. **Audit log** centralisé : chaque opération sensible (login, modification d'admin, suppression) loggée avec contexte (IP, user agent).
6. **Scan automatisé** des dépendances : `npm audit` dans la CI, Snyk ou Dependabot pour les alertes.

> **Comment formuler à l'oral** : « Trois priorités si je devais reprendre demain : un, le rate limiting sur les endpoints d'auth — c'est la mesure de sécu avec le meilleur rapport effort/protection. Deux, des refresh tokens pour résoudre le problème de révocation du JWT. Trois, des headers de sécu via un middleware custom — CSP, HSTS, etc. »

---

## Q2. Comment iriez-vous plus loin sur les performances ?

**Comment répondre**

1. **Cache HTTP** : headers `Cache-Control` agressifs sur les assets statiques (immutable + hash dans le nom), `ETag` sur les réponses API.
2. **Cache applicatif** : Redis devant les requêtes lourdes (recherches, listes paginées) avec invalidation à la création/modif.
3. **Optimisation des images** : pipeline d'optimisation à l'upload (resize, conversion WebP, lazy-loading). Service externe type imgproxy ou Cloudinary.
4. **Index DB** : profilage avec `EXPLAIN` sur les requêtes les plus fréquentes, ajout d'index ciblés.
5. **Pagination cursor-based** pour les listes longues (plus rapide qu'`OFFSET` au-delà de quelques milliers d'entrées).
6. **SSR + hydration partielle** : Angular 21 supporte le defer/hydration partielle pour ne charger le JS que des composants visibles.
7. **CDN** devant les assets statiques.

> **À l'oral** : « Le premier levier serait un cache Redis sur les requêtes de recherche — c'est l'endpoint le plus appelé, et les résultats varient peu d'une seconde à l'autre. Le deuxième serait l'optimisation des images uploadées par les utilisateurs : c'est ce qui plombe le score Lighthouse aujourd'hui. »

---

## Q3. Quelles fonctionnalités utilisateurs aimeriez-vous ajouter ?

**Comment répondre** — Choisir 3-4 idées qui **enrichissent l'expérience** :

1. **Recommandations personnalisées** : « les utilisateurs qui ont aimé cette recette ont aussi aimé… » — algorithme simple basé sur les favoris communs.
2. **Notation des recettes** (étoiles) avec moyenne affichée + filtre « les mieux notées ».
3. **Listes de courses générées** depuis une ou plusieurs recettes — agréger les ingrédients avec les quantités.
4. **Plan de repas hebdomadaire** : drag-and-drop pour organiser les recettes sur 7 jours.
5. **Photos par étape** : permettre d'ajouter une image à chaque étape de préparation.
6. **Système de tags utilisateurs** : « végétarien », « sans gluten », « rapide ».
7. **Mode hors-ligne** : Service Worker + cache des recettes consultées récemment (PWA).
8. **Partage social** : open graph pour avoir une carte propre sur les réseaux.
9. **Conversions d'unités** automatiques (cup → g, F° → C°) selon le profil utilisateur.
10. **Internationalisation** : multi-langues via `@angular/localize`.

> **À l'oral** : « Côté UX, ce qui me semble le plus impactant ce serait les listes de courses générées automatiquement à partir des recettes sauvegardées — c'est un cas d'usage concret qui transformerait le site d'un catalogue en outil de planification. »

---

## Q4. Comment iriez-vous plus loin sur la qualité du code ?

**Comment répondre**

1. **Tests E2E avec Playwright** : valider les parcours critiques (inscription → connexion → création recette → commentaire) bout en bout.
2. **Tests d'intégration en DB réelle** : Testcontainers pour spawner une MySQL Docker dans les tests.
3. **CI/CD complète** : GitHub Actions qui lint + teste + déploie, branche `main` protégée, status checks obligatoires.
4. **Couverture de code mesurée** et un seuil minimum (ex : 80 %) bloquant en CI.
5. **Code review** systématique sur PR (même en projet solo, faire des PR pour relire à froid).
6. **Documentation OpenAPI/Swagger** générée depuis le code et exposée sur `/docs`.
7. **Génération de types front depuis le back** via OpenAPI generator pour éviter la duplication.
8. **Linter plus strict** : eslint-plugin-security, eslint-plugin-jsdoc, niveau `recommended-strict`.

---

## Q5. Comment iriez-vous plus loin sur le déploiement et l'exploitation ?

**Comment répondre**

1. **Containerisation Docker** : Dockerfile multistage pour back et front, docker-compose pour le local.
2. **Infrastructure as Code** : Terraform ou Pulumi pour décrire l'infra (DB, hébergement, DNS).
3. **Monitoring** : Prometheus + Grafana, alertes sur les latences et taux d'erreur.
4. **Logging centralisé** : Pino → Loki ou Datadog.
5. **Endpoint `/health`** pour les health checks de la plateforme.
6. **Migrations DB versionnées** : scripts numérotés appliqués automatiquement au démarrage, ou outil dédié (Flyway, dbmate).
7. **Backups automatiques** + test de restauration trimestriel.
8. **CDN + cache edge** devant les assets statiques (Cloudflare, Fastly).

---

## Q6. Quelles évolutions techniques côté front ?

**Comment répondre**

1. **PWA** : manifest + Service Worker → installable sur mobile, cache hors-ligne.
2. **Lighthouse > 95 partout** : optimiser images, lazy-loading, defer JS.
3. **Composants Material ou PrimeNG** en complément de Bootstrap pour des composants riches (datepicker, autocomplete avancé).
4. **Animations** : `@angular/animations` pour des transitions fluides (entrée carte recette, fade-in modal).
5. **Storybook** pour documenter les composants visuellement et tester en isolation.
6. **Tests E2E Playwright** plutôt que Cypress pour la perf et le support natif TS.

---

## Q7. Et côté intelligence artificielle / fonctionnalités modernes ?

**Intention jury** — Question parfois posée pour voir si Arthur suit les tendances.

**Comment répondre** — Avec prudence et concret :

1. **Génération de recette à partir d'une liste d'ingrédients** : un endpoint qui prend `[œufs, farine, lait]` → appelle une API LLM → retourne une recette suggérée. Use case fort pour découvrir des combinaisons.
2. **Description automatique de recette à partir d'une image** : multimodal — l'utilisateur upload sa photo de plat fini, le LLM propose un titre + une description.
3. **Recherche sémantique** : embeddings sur les recettes + index vectoriel (pgvector ou similaire) pour des recherches type « plat froid d'été léger ».
4. **Modération automatique** : appel à un LLM pour détecter les commentaires injurieux et les router vers la file de modération avec une priorité.

> **À l'oral** : « Le plus pertinent pour ce produit serait probablement la recherche sémantique : aujourd'hui ma recherche est lexicale, ce qui force l'utilisateur à connaître les bons mots-clés. Une recherche par embeddings permettrait des requêtes en langage naturel. C'est un chantier réaliste — pgvector + un modèle d'embedding open-source. »

**Pièges**
- ❌ Tomber dans le buzzword. Rester concret sur **quel bénéfice utilisateur**.

---

## Q8. Comment gérez-vous l'évolution du schéma de base de données ?

**Intention jury** — Connaissance des migrations.

**Comment répondre**

> *(Si pas de système de migration)* « J'ai un script SQL initial dans `database/` qui crée tout le schéma. Je n'ai pas d'outil de migration versionné — c'est une vraie limite, surtout dès qu'il y a plusieurs environnements. Je rajouterais soit un outil dédié (Flyway, dbmate, node-pg-migrate adapté MySQL), soit un système maison : des fichiers `001_init.sql`, `002_add_favorites.sql`, etc. + une table `migrations` qui trace ce qui a été appliqué. À l'app start, on lance les migrations manquantes. »

---

## Q9. Comment surveilleriez-vous votre app en production ?

**Comment répondre**

1. **Logs structurés** (JSON) envoyés à un agrégateur (Loki, ELK).
2. **Métriques** : nombre de requêtes par endpoint, latence p50/p95/p99, taux d'erreur. Exposées via `/metrics` (Prometheus).
3. **Traces distribuées** (OpenTelemetry) si on évolue vers du multi-service.
4. **Alertes** : taux d'erreur > 5 % → notification Slack/email.
5. **Dashboard** Grafana avec les courbes clés.
6. **Real User Monitoring** côté front (Sentry pour les JS errors, web vitals).

---

## Q10. Si vous deviez ouvrir le projet à des contributions externes, que feriez-vous ?

**Intention jury** — Question parfois posée pour évaluer la maturité collaborative.

**Comment répondre**

1. **README accueillant** avec install rapide + lien vers `CONTRIBUTING.md`.
2. **CONTRIBUTING.md** : convention de commits, workflow de PR, checklist de relecture.
3. **CODE_OF_CONDUCT.md** standard.
4. **Issue templates** + PR templates GitHub.
5. **Labels** : `good first issue`, `help wanted`, `bug`, `enhancement`.
6. **CI obligatoire** sur PR.
7. **Releases tagged** (SemVer) + changelog.
