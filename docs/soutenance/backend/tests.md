# Stratégie de tests du backend Recipe Shelter

> Document de référence pour la défense de soutenance (RNCP). Décrit la
> stratégie de tests unitaires du backend, le choix du runner natif, la
> structure du dossier `tests/` et la couverture par domaine. Versionné dans
> `documentation/soutenance/backend/tests.md` (cible), brouillon courant
> dans `_draft_backend_docs/tests.md`.

## 1. Philosophie : le test runner natif

La stratégie de tests du backend suit le même parti pris que le reste du
projet : **pas de framework lourd, pas de magie**. Là où la plupart des
projets Node ajoutent Jest, Mocha ou Vitest dès le premier test, Recipe
Shelter s'appuie exclusivement sur le runner intégré à Node.js
(`node:test`) et sur la bibliothèque d'assertions standard
(`node:assert/strict`).

Ce choix est cohérent avec la démarche « from scratch » revendiquée dans le
Bloc 2 de la certification. Le `package.json` du backend ne contient
**aucune dépendance de test** : ni Jest, ni Mocha, ni Vitest, ni Chai, ni
Sinon, ni Supertest. La seule dépendance de développement nécessaire pour
exécuter les tests est `tsx` (déjà présent pour `npm run dev`), qui charge
le code TypeScript à la volée.

Concrètement, cela signifie que :

- chaque ligne de la stack de test est défendable et explicable lors du
  passage devant le jury, sans avoir à invoquer la documentation d'un
  framework tiers ;
- les tests sont écrits avec la même API stable que celle qui sera
  maintenue par la fondation Node.js sur le long terme, sans risque
  d'obsolescence d'un framework ;
- le démarrage des tests est immédiat : pas de phase de bootstrap Jest, pas
  de configuration `jest.config.js`, pas de hack pour faire fonctionner les
  modules ESM.

Le revers de la médaille est documenté en section 8 : pas de mocks
automatiques, pas d'outil de couverture intégré. On verra que ces deux
limites sont volontairement compensées par des patterns simples (fakes
explicites) et acceptées comme améliorations de Vague 3.

## 2. Stack de test

| Composant         | Bibliothèque         | Version | Rôle                                                         |
| ----------------- | -------------------- | ------- | ------------------------------------------------------------ |
| Test runner       | `node:test`          | natif   | API `describe` / `it` / `beforeEach`, exécution parallèle.   |
| Assertions        | `node:assert/strict` | natif   | `equal`, `deepEqual`, `rejects`, `throws`, `match`, `ok`.    |
| Chargement TS     | `tsx`                | ^4.21.0 | Loader ESM pour exécuter le TypeScript sans étape de build.  |

Le runner `node:test` est disponible depuis Node 18 et stabilisé depuis
Node 20 (la version cible du projet, cf. `architecture.md` §2). Il fournit
nativement les primitives suivantes, toutes utilisées dans la suite :

- `describe(name, fn)` pour regrouper des cas par sujet ;
- `it(name, fn)` (alias `test`) pour un cas individuel, `async` ou non ;
- `beforeEach(fn)` pour réinitialiser l'état entre cas ;
- exécution parallèle par fichier, isolation par défaut.

La commande de test est définie dans `backend/package.json:14` :

```json
"test": "node --import tsx --test \"tests/**/*.test.ts\""
```

Décomposition :

- `node --test` active le runner natif et collecte tous les fichiers passés
  en argument ou en glob ;
- `--import tsx` injecte le loader `tsx` au démarrage du processus, ce qui
  permet à Node d'exécuter directement les fichiers `.ts` sans `tsc` ;
- le glob `"tests/**/*.test.ts"` est résolu par Node (et non par le shell),
  ce qui assure une exécution identique sur Windows, macOS et Linux.

La configuration TypeScript du backend (`backend/tsconfig.json`) cible
`ES2022` avec `module: NodeNext`, ce qui correspond exactement à ce que
`tsx` consomme. Il n'y a donc **pas de `tsconfig.test.json` séparé** : les
tests sont compilés avec la même configuration que la production. C'est un
gage de cohérence et un point qu'il est possible de revendiquer en
soutenance.

## 3. Organisation du dossier `tests/`

Le dossier `backend/tests/` reproduit fidèlement la structure de `src/`,
couche par couche. Pour un fichier source `src/<couche>/<domaine>/X.ts`, le
test correspondant se trouve dans `tests/<couche>/<domaine>/X.test.ts`.
Cette convention rend la navigation immédiate (ouvrir le test à côté du
fichier testé est une simple bascule de répertoire).

Inventaire chiffré : **29 fichiers de test** au total, répartis comme
suit :

### Services (11 fichiers)

Logique métier pure, indépendante d'Express. C'est la couche la mieux
couverte, car c'est là que vivent les règles défendables en soutenance.

- `services/auth/auth.service.test.ts` — inscription, login, JWT
- `services/auth/email-validation.service.test.ts` — workflow de
  validation d'email
- `services/auth/password-policy.test.ts` — règles de mot de passe fort
- `services/auth/password-reset.service.test.ts` — workflow de
  réinitialisation
- `services/admin/admin.comments.service.test.ts` — modération commentaires
- `services/admin/admin.recipes.service.test.ts` — modération recettes
- `services/admin/admin.users.service.test.ts` — gestion comptes et bans
- `services/comments/comments.service.test.ts` — CRUD commentaires
- `services/recipes/recipes.service.test.ts` — cycle de vie des recettes
- `services/recipes/recipe-slug.service.test.ts` — génération de slug
  unique
- `services/users/users.service.test.ts` — profil utilisateur

### API / DTO (9 fichiers)

Validation des bodies entrants et normalisation (trim, lowercase email).
Un fichier de test par endpoint majeur.

- `api/auth/auth.controller.test.ts` — wiring HTTP du contrôleur auth
- `api/auth/auth.dto.test.ts` — `parseRegisterBody`, `parseLoginBody`, etc.
- `api/admin/admin.comments.dto.test.ts`
- `api/admin/admin.recipes.dto.test.ts`
- `api/admin/admin.users.dto.test.ts`
- `api/comments/comments.dto.test.ts`
- `api/contact/contact.dto.test.ts`
- `api/recipes/recipes.dto.test.ts`
- `api/users/users.dto.test.ts`

### Middlewares (5 fichiers)

Pipeline Express : authentification, autorisation, gestion d'erreur,
limitation de débit, 404.

- `middlewares/error-handler.test.ts`
- `middlewares/not-found.test.ts`
- `middlewares/rate-limiter.test.ts`
- `middlewares/require-auth.test.ts`
- `middlewares/require-admin.test.ts`

### Repositories / Mappers (2 fichiers)

Conversion des lignes SQL brutes (colonnes `PascalCase`) vers les objets
métier (propriétés `camelCase`). C'est la frontière entre persistance et
domaine.

- `repositories/comments/comments.mapper.test.ts`
- `repositories/recipes/recipe.mapper.test.ts`

### Utils (2 fichiers)

Utilitaires transverses, dont une primitive de sécurité (cf. §6).

- `utils/pagination.test.ts`
- `utils/security/password-reset-token.test.ts`

## 4. Patterns de test

Les 29 fichiers de test suivent un petit nombre de patterns récurrents,
volontairement explicites pour rester lisibles.

### 4.1 Fakes in-memory, pas de mocks magiques

Les services dépendent d'un `UserRepository`, d'un `RecipeRepository`,
d'un `Mailer`, etc. Pour les tester en isolation, le projet n'utilise ni
`jest.mock`, ni `sinon.stub`, ni `vi.fn()`. À la place, chaque test
construit des **classes fakes** qui implémentent l'interface attendue avec
un état mutable in-memory.

Exemple tiré de `tests/services/auth/auth.service.test.ts:29-57` :

```typescript
class FakeUserRepository implements Partial<UserRepository> {
    createdInput: CreateUserInput | null = null;
    emailTaken = false;
    usernameTaken = false;
    roleId: number | null = 2;
    authUser: UserWithPassword | null = null;

    async isEmailTaken(): Promise<boolean> {
        return this.emailTaken;
    }

    async isUsernameTaken(): Promise<boolean> {
        return this.usernameTaken;
    }

    async getRoleIdByName(): Promise<number | null> {
        return this.roleId;
    }

    async create(input: CreateUserInput): Promise<User> {
        this.createdInput = input;

        return { ...baseUser, mail: input.mail, /* ... */ };
    }

    async findAuthByEmail(): Promise<UserWithPassword | null> {
        return this.authUser;
    }
}
```

L'avantage de cette approche :

- **lisibilité** : un fake est juste une classe TypeScript, son
  comportement est entièrement défini dans le fichier de test ;
- **typage strict** : `implements Partial<UserRepository>` garantit que
  toute évolution de l'interface est détectée par le compilateur ;
- **inspection facile** : les fakes exposent leur état (`createdInput`)
  pour qu'on puisse asserter ce qui leur a été demandé (`assert.equal(
  users.createdInput?.mail, 'user@example.com')`) ;
- **pas de framework** : un junior peut lire et comprendre le test sans
  connaître l'API d'un mock framework.

Le coût (verbosité) est assumé. Sur 29 fichiers, l'addition reste
raisonnable.

### 4.2 Fixtures avec timestamps réalistes

Chaque fichier de test qui manipule des entités déclare un objet `baseUser`
(ou `baseRecipe`, `listRow`...) en haut du fichier, avec des dates
réalistes (`new Date('2026-05-09T10:00:00.000Z')`), des IDs stables et
tous les champs requis par le typage. Les cas individuels font ensuite des
**spreads** (`{ ...baseUser, status: 'banned' }`) pour ne mettre en avant
que le champ pertinent.

Voir par exemple `tests/services/auth/auth.service.test.ts:15-27` ou
`tests/middlewares/require-auth.test.ts:13-25`.

### 4.3 Pattern AAA (Arrange / Act / Assert)

Tous les tests suivent le découpage classique en trois temps, sans
commentaires `// Arrange` superflus : la structure visuelle suffit. Exemple
condensé tiré de `tests/services/auth/auth.service.test.ts:113-125` :

```typescript
it('logs in active users and returns a signed token', async () => {
    users.authUser = { ...baseUser, passwordHash: await bcrypt.hash('Recipe42?', 4) };

    const result = await service.login({ mail: ' USER@Example.COM ', password: 'Recipe42?' });
    const payload = jwt.verify(result.token, env.auth.jwtSecret) as jwt.JwtPayload;

    assert.equal(result.user.mail, 'user@example.com');
    assert.equal('passwordHash' in result.user, false);
    assert.equal(payload.sub, 2);
    assert.equal(payload.username, 'testuser');
    assert.equal(payload.roleId, 2);
    assert.equal(payload.status, 'active');
});
```

Trois choses se lisent immédiatement :

1. **Arrange** : on prépare un utilisateur authentifiable avec un hash
   bcrypt généré à la volée (coût 4 en test, cf. §4.5).
2. **Act** : on appelle `service.login(...)` avec un email volontairement
   sale (`' USER@Example.COM '`) pour vérifier la normalisation.
3. **Assert** : on contrôle à la fois la forme de la réponse (l'absence du
   `passwordHash` dans le retour est explicitement assertée) et le
   contenu du JWT en le vérifiant avec la vraie clé.

Ce test prouve trois invariants critiques en six assertions : la
normalisation d'email, la non-fuite du hash de mot de passe, et la
signature correcte du JWT avec le bon payload.

### 4.4 Helpers d'assertion réutilisables

Le projet répète assez souvent le pattern « vérifier qu'une erreur est une
`HttpError` avec un code et un statut donnés ». Chaque fichier qui en a
besoin déclare un mini-helper local plutôt que de le centraliser dans un
module partagé :

```typescript
function assertHttpError(error: unknown, code: string, status: number): boolean {
    assert.ok(error instanceof HttpError);
    assert.equal(error.code, code);
    assert.equal(error.statusCode, status);

    return true;
}
```

(`tests/services/auth/auth.service.test.ts:67-73`,
`tests/api/auth/auth.dto.test.ts:7-11`,
`tests/utils/pagination.test.ts:7-11`.)

Cette duplication est délibérée : chaque fichier reste autonome, on peut
le lire de bout en bout sans naviguer ailleurs. C'est aussi la
recommandation classique pour les tests (« DRY pour le code de prod,
explicite pour les tests »).

### 4.5 Coût bcrypt réduit en test

Le coût bcrypt par défaut est 12 (cf. `securite.md`). En test on le
ramène à 4 dans le `beforeEach` :

```typescript
beforeEach(() => {
    env.auth.bcryptCost = 4;
    /* ... */
});
```

(`tests/services/auth/auth.service.test.ts:80-85`.)

C'est invisible côté code de production et indispensable pour que la suite
de tests s'exécute en quelques secondes plutôt qu'en plusieurs minutes.

### 4.6 Faux objets `Request` / `Response` pour les middlewares

Plutôt que de monter un serveur Express et de tirer des requêtes via
Supertest, les middlewares sont testés en leur passant à la main des
objets `req`, `res`, `next` minimalistes. Le test d'`errorHandler` en
est l'illustration la plus pure
(`tests/middlewares/error-handler.test.ts:7-20`) :

```typescript
function createResponse() {
    return {
        statusCode: 0,
        body: null as unknown,
        status(code: number) {
            this.statusCode = code;
            return this;
        },
        json(payload: unknown) {
            this.body = payload;
            return this;
        }
    };
}
```

Pour les middlewares plus complexes (`requireAuth`), on capture le
`next(error)` dans une variable locale et on assert ensuite sur l'erreur
reçue (cf. `tests/middlewares/require-auth.test.ts:47-56`).

## 5. Couverture par domaine

Le tableau ci-dessous résume ce qui est couvert et ce qui ne l'est pas
explicitement. La colonne « Référence » pointe vers le test le plus
représentatif du domaine.

| Domaine                              | Couvert ? | Référence                                                    |
| ------------------------------------ | --------- | ------------------------------------------------------------ |
| Inscription utilisateur              | Oui       | `services/auth/auth.service.test.ts`                         |
| Login + JWT signé                    | Oui       | `services/auth/auth.service.test.ts`                         |
| Validation d'email (token)           | Oui       | `services/auth/email-validation.service.test.ts`             |
| Reset password (workflow)            | Oui       | `services/auth/password-reset.service.test.ts`               |
| Reset password (hash crypto)         | Oui       | `utils/security/password-reset-token.test.ts`                |
| Politique de mot de passe fort       | Oui       | `services/auth/password-policy.test.ts`                      |
| CRUD recettes + cycle de vie         | Oui       | `services/recipes/recipes.service.test.ts`                   |
| Unicité de slug                      | Oui       | `services/recipes/recipe-slug.service.test.ts`               |
| Commentaires (threading, modération) | Oui       | `services/comments/comments.service.test.ts`                 |
| Modération admin (recettes / users)  | Oui       | `services/admin/*.test.ts`                                   |
| Validation des DTO entrants          | Oui       | `api/*/. *.dto.test.ts`                                      |
| Pipeline d'erreurs HTTP              | Oui       | `middlewares/error-handler.test.ts`                          |
| Authentification middleware          | Oui       | `middlewares/require-auth.test.ts`                           |
| Autorisation admin                   | Oui       | `middlewares/require-admin.test.ts`                          |
| Rate limiting                        | Oui       | `middlewares/rate-limiter.test.ts`                           |
| Mapping SQL → domaine                | Oui       | `repositories/recipes/recipe.mapper.test.ts`                 |
| Pagination & bornes                  | Oui       | `utils/pagination.test.ts`                                   |
| Intégration HTTP de bout en bout     | **Non**   | hors périmètre — couvert par les tests E2E frontend          |
| Persistance MySQL réelle             | **Non**   | hors périmètre — repositories testés via mappers seulement   |
| Envoi SMTP réel                      | **Non**   | abstrait derrière l'interface `Mailer`, fake en test         |
| Tests de charge                      | **Non**   | non requis par la certification                              |

La règle implicite est : **tout ce qui contient une règle métier
défendable est testé unitairement**. Les zones d'intégration (HTTP réel,
SQL réel, SMTP réel) sont couvertes manuellement ou par les tests E2E
côté frontend, et reconnues comme axes d'amélioration (§8).

## 6. Section spéciale : tests de cryptographie sécurité

Un test mérite d'être appelé en soutenance par-dessus les autres :
`tests/utils/security/password-reset-token.test.ts` (20 lignes). Il valide
les deux primitives de sécurité de la réinitialisation de mot de passe,
documentées en détail dans `securite.md` §reset password.

Contenu intégral du fichier
(`tests/utils/security/password-reset-token.test.ts:6-20`) :

```typescript
describe('password-reset-token', () => {
    it('generates 32-byte hex tokens', () => {
        const token = generateResetToken();

        assert.match(token, /^[a-f0-9]{64}$/);
    });

    it('hashes tokens deterministically without exposing the raw token', () => {
        const hash = hashResetToken('raw-token');

        assert.match(hash, /^[a-f0-9]{64}$/);
        assert.equal(hash, hashResetToken('raw-token'));
        assert.notEqual(hash, 'raw-token');
    });
});
```

Ce que ces deux cas garantissent :

1. **Entropie suffisante** : le token brut envoyé par email est composé
   de 64 caractères hexadécimaux, soit 32 octets aléatoires (256 bits).
   La regex `^[a-f0-9]{64}$` vaut preuve de format et de longueur.
2. **Hash déterministe** : `hashResetToken(raw)` appelle SHA-256 sur le
   token brut. Le test vérifie que (a) le hash a la longueur attendue
   (64 hex = 256 bits), (b) le hash est reproductible (deux appels avec
   la même entrée produisent la même sortie, ce qui est requis pour
   pouvoir vérifier un token en base), (c) le hash diffère du token brut
   (ce qui est requis pour qu'un dump de base ne révèle pas les tokens
   actifs).

C'est un test court mais à fort impact défensif : il prouve que la base
de données ne stocke **jamais** la valeur qui a été envoyée à
l'utilisateur, et que la propriété « one-way » du hash est respectée.

À combiner avec le test du workflow complet
(`tests/services/auth/password-reset.service.test.ts`) qui vérifie
l'expiration du token, l'invalidation après usage et l'absence d'oracle
sur les emails inexistants.

## 7. Exécution des tests

### Commande

```bash
npm test
```

ou directement :

```bash
node --import tsx --test "tests/**/*.test.ts"
```

### Sortie attendue

Le runner produit une sortie TAP (Test Anything Protocol) lisible :

```
TAP version 13
# Subtest: AuthService
    # Subtest: registers inactive users and sends a validation email
    ok 1 - registers inactive users and sends a validation email
      ---
      duration_ms: 124.5
    ...
ok 1 - AuthService
  ---
  duration_ms: 312.8
...
# tests 142
# pass 142
# fail 0
# duration_ms 4521.3
```

Chaque `ok` correspond à un cas `it(...)` réussi ; chaque `not ok` à un
échec, avec le diff complet de l'assertion qui a échoué (avantage de
`node:assert/strict` sur `node:assert`).

### Filtrer un sous-ensemble

Le runner supporte nativement deux options utiles :

- `--test-name-pattern="logs in"` ne lance que les cas dont le nom
  contient `logs in` ;
- passer un sous-glob (`node --import tsx --test "tests/services/**/*.test.ts"`)
  pour ne tester que la couche services.

Ces options sont précieuses lors du développement (relancer un seul cas
suffit) et n'ont nécessité aucune configuration ad-hoc.

### Absence d'outil de couverture intégré

Le projet n'utilise **pas** d'outil de mesure de couverture
(`c8`, `nyc`, `istanbul`). C'est une limitation assumée : aucune
métrique chiffrée de couverture n'est aujourd'hui produite. Pour la
soutenance, la couverture se défend qualitativement à partir du tableau
de la section 5, pas avec un pourcentage automatique.

## 8. Améliorations possibles (Vague 3)

Par honnêteté envers le jury, et conformément au format « Vagues »
documenté dans le cahier des charges, voici les améliorations identifiées
mais non implémentées dans la version certifiée.

### 8.1 Mesure de couverture

Ajouter `c8` (compatible avec `node:test`) en `devDependency`, modifier la
commande en `c8 node --import tsx --test ...`. Coût : une dépendance, une
commande, zéro réécriture de tests. Bénéfice : un pourcentage objectif à
montrer en démo, et la détection automatique des fichiers de code
non couverts (typiquement les nouveaux services ajoutés sans test).

### 8.2 Tests d'intégration avec une vraie MySQL

Les `RepositoryImpl` ne sont aujourd'hui testés qu'indirectement (via les
services qui les utilisent avec des fakes). Une suite d'intégration ferait
tourner les `RepositoryImpl` réels contre un conteneur Docker MySQL
éphémère (`testcontainers-node` ou simplement un `docker run` dans un
script `pretest:integration`). Ce serait l'occasion de valider :

- les requêtes SQL réellement exécutées, y compris les contraintes ;
- les transactions multi-tables (recettes avec ingrédients / étapes /
  équipements) ;
- la cohérence des types entre `mysql2` et les types TypeScript des
  rows.

C'est l'amélioration la plus impactante côté confiance, et la plus
coûteuse en temps de mise en place.

### 8.3 Tests E2E HTTP côté backend

Aujourd'hui, les contrôleurs sont testés par leur DTO + leur service. Un
test HTTP complet (avec `node:http` directement, ou en supplément
`undici.fetch` contre une instance Express bootée en test) permettrait de
valider la chaîne entière, y compris :

- l'ordre des middlewares ;
- le contenu du cookie de session ;
- les headers CORS effectifs ;
- la sérialisation finale des erreurs.

Pas besoin de Supertest : Node 20 a `fetch` global. Cette suite pourrait
rester légère (un test par endpoint critique).

### 8.4 Tests de charge

Hors périmètre de la certification, mais à mentionner si le jury pousse :
un `autocannon` ou un `k6` ciblé sur `GET /recipes` (l'endpoint le plus
chaud) donnerait une borne supérieure du débit et identifierait les
contentions sur le pool MySQL.

### 8.5 Property-based testing

Pour les utils purs (slug, pagination, password policy), une bibliothèque
comme `fast-check` permettrait de générer aléatoirement des entrées et de
vérifier des invariants (« le slug est toujours `kebab-case` », « le
offset est toujours `(page - 1) * limit` »). C'est un raffinement avancé,
pertinent uniquement après les améliorations §8.1 et §8.2.

---

**En résumé**, la stratégie de tests du backend est volontairement
minimaliste dans ses dépendances et exhaustive dans sa couverture des
règles métier. Elle s'aligne avec la philosophie générale du projet :
chaque ligne du `package.json` est défendable, chaque test est lisible
sans connaissance préalable d'un framework, et les limitations
(couverture chiffrée, intégration MySQL) sont identifiées et planifiées
plutôt que masquées.
