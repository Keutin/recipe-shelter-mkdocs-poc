# Plan déploiement Railway pas-à-pas

Procédure complète pour mettre Recipe Shelter en ligne sur Railway. Effort estimé : **3 heures** si rien ne dérape, **5 heures** avec debug.

> Pré-requis : compte GitHub avec les 3 repos `arthur-lagenebre/recipe-shelter-*` accessibles. Compte email pour Railway.

## Phase 1 — Préparer le code pour la prod (30 min)

Avant de cliquer Deploy, valider que le code accepte une config prod.

### 1.1 Vérifier le port d'écoute du backend

```bash
cd C:\DEV\ARTHUR\RECETTES\backend
```

Ouvrir `src/server.ts` (ou équivalent). Chercher `app.listen(...)`. Doit ressembler à :

```ts
const port = process.env.PORT ?? 3000;
app.listen(port, () => console.log(`API listening on ${port}`));
```

Si c'est en dur (`app.listen(3000)`), corriger. Railway injecte `$PORT` dynamiquement, refuser cette injection = service down.

### 1.2 Vérifier la connexion MySQL

Ouvrir le fichier de config DB (probablement `src/infrastructure/database/` ou `src/config/`). Doit lire :

```ts
host: process.env.DB_HOST,
port: parseInt(process.env.DB_PORT ?? '3306'),
user: process.env.DB_USER,
password: process.env.DB_PASSWORD,
database: process.env.DB_NAME,
```

Railway MySQL fournit `MYSQLHOST`, `MYSQLPORT`, `MYSQLUSER`, `MYSQLPASSWORD`, `MYSQLDATABASE`. Tu peux soit :

- Aliaser via les variables Railway (`DB_HOST=${{MYSQL.MYSQLHOST}}`),
- Soit ajouter du code de fallback.

Recommandation : aliaser dans Railway (interface graphique) → pas de modif code.

### 1.3 Build le backend localement pour valider

```bash
npm install
npm run build           # devrait produire dist/server.js
node dist/server.js     # doit démarrer sans erreur (modulo MySQL pas accessible)
```

Si le build casse → fix avant de continuer.

### 1.4 Build le frontend prod localement

```bash
cd ..\frontend
npm install
npm run build           # produit dist/recipe-shelter/{browser,server}
npm run serve:ssr:recipe-shelter   # doit servir sur :4000
```

Ouvrir `http://localhost:4000` → home s'affiche.

### 1.5 Commit les éventuels fixes

```bash
git checkout -b chore/prepare-prod
git commit -am "chore: prepare backend for production env"
git push -u origin chore/prepare-prod
# Merge la PR sur main ensuite
```

## Phase 2 — Provisionner MySQL sur Railway (15 min)

1. Aller sur `https://railway.app` → `New Project` → `Provision MySQL`.
2. Une fois créé, ouvrir le service MySQL → onglet `Variables` → noter :
   - `MYSQLHOST`
   - `MYSQLPORT`
   - `MYSQLUSER`
   - `MYSQLPASSWORD`
   - `MYSQLDATABASE`
3. Onglet `Connect` → récupérer la commande `mysql -h <host> -P <port> -u <user> -p<password> <database>`.

## Phase 3 — Charger le seed démo dans Railway MySQL (15 min)

Le fichier `backend/database/reset_demo.sql` contient le schéma + données de démo (comptes utilisateurs, recettes, commentaires).

Depuis ton poste local, avec un client MySQL :

```bash
# Option A — client mysql en ligne de commande (si installé)
mysql -h <MYSQLHOST> -P <MYSQLPORT> -u <MYSQLUSER> -p<MYSQLPASSWORD> <MYSQLDATABASE> < C:\DEV\ARTHUR\RECETTES\backend\database\reset_demo.sql

# Option B — MySQL Workbench / DBeaver (GUI)
# Connecter avec les credentials Railway, ouvrir le SQL editor, charger et exécuter reset_demo.sql.
```

Validation :

```sql
SELECT COUNT(*) FROM users;          -- doit retourner > 0
SELECT email FROM users WHERE email LIKE '%admin%';   -- doit voir admin_demo@recipe-shelter.fr
```

## Phase 4 — Déployer le backend (45 min)

1. Dans le projet Railway, `New Service` → `GitHub Repo` → autoriser → choisir `recipe-shelter-backend`.
2. Railway détecte Node automatiquement (présence de `package.json`).
3. Onglet `Settings` du service backend :
   - **Build command** : `npm ci && npm run build`
   - **Start command** : `npm start` (qui lance `node dist/server.js`)
   - **Root directory** : laisser vide (repo entier)
4. Onglet `Variables` → ajouter (en s'inspirant de `backend/.env.example`) :

```env
NODE_ENV=production
PORT=3000                  # Railway override en interne, mais doit être présent
DB_HOST=${{MySQL.MYSQLHOST}}
DB_PORT=${{MySQL.MYSQLPORT}}
DB_NAME=${{MySQL.MYSQLDATABASE}}
DB_USER=${{MySQL.MYSQLUSER}}
DB_PASSWORD=${{MySQL.MYSQLPASSWORD}}
DB_CONNECTION_LIMIT=10

JWT_SECRET=<générer 64 caractères aléatoires — utiliser https://passwordsgenerator.net/ ou openssl rand -hex 32>
JWT_EXPIRES_IN=7d
AUTH_SESSION_COOKIE_NAME=rs_session
AUTH_SESSION_COOKIE_SAME_SITE=none      # cross-site (back et front sur 2 domaines)
AUTH_SESSION_COOKIE_SECURE=true         # HTTPS prod
BCRYPT_COST=12
AUTH_DEFAULT_ROLE_NAME=user
AUTH_RATE_LIMIT_MAX_ATTEMPTS=5
AUTH_RATE_LIMIT_WINDOW_MS=900000

CORS_ALLOWED_ORIGINS=https://<frontend-railway-domain>.up.railway.app
FRONTEND_BASE_URL=https://<frontend-railway-domain>.up.railway.app

# SMTP — voir phase 6 pour les valeurs réelles
SMTP_HOST=smtp.brevo.com
SMTP_PORT=587
SMTP_SECURE=false
SMTP_USER=<à remplir>
SMTP_PASSWORD=<à remplir>
SMTP_FROM=no-reply@recipe-shelter.up.railway.app

CONTACT_RECIPIENT_EMAIL=<ton email perso>
```

5. Onglet `Settings` → `Networking` → `Generate Domain` → noter l'URL `https://<backend-railway>.up.railway.app`.
6. Onglet `Deployments` → attendre que le build passe au vert.
7. Tester : `curl https://<backend-railway>.up.railway.app/api/v1/health` (si une route health existe ; sinon `curl ...` la route d'une liste de recettes publiques).

### Si le build casse — diagnostics fréquents

| Erreur log | Cause probable | Fix |
| --- | --- | --- |
| `ERR_MODULE_NOT_FOUND` | Build TypeScript pas exécuté | Vérifier `npm run build` produit `dist/server.js` |
| `ECONNREFUSED ::1:3306` | DB_HOST pointe localhost | Variables MySQL pas reliées au service MySQL Railway |
| `Access denied for user` | Credentials MySQL incorrects | Re-copier depuis l'onglet Variables MySQL |
| `Cannot find module 'tsx'` | Tentative de lancer en dev | Vérifier que start = `node dist/server.js`, pas `tsx` |

## Phase 5 — Déployer le frontend (45 min)

1. Mettre à jour `frontend/src/environments/environment.prod.ts` :

```ts
export const environment = {
  production: true,
  apiBaseUrl: 'https://<backend-railway>.up.railway.app/api/v1',
};
```

Commit + push sur `main`.

2. Dans Railway, `New Service` → `GitHub Repo` → `recipe-shelter-frontend`.
3. Settings :
   - **Build command** : `npm ci && npm run build`
   - **Start command** : `npm run serve:ssr:recipe-shelter`
   - **Root directory** : vide
4. Variables :

```env
NODE_ENV=production
PORT=4000                  # le serveur SSR écoute sur PORT (à vérifier dans dist/server.mjs)
```

Si le SSR Angular écoute en dur sur 4000, c'est OK Railway redirige. Sinon vérifier `frontend/src/server.ts`.

5. `Generate Domain` → noter l'URL `https://<frontend-railway>.up.railway.app`.
6. **Retour côté backend** : mettre à jour les variables `CORS_ALLOWED_ORIGINS` et `FRONTEND_BASE_URL` avec l'URL frontend réelle, redéployer.

7. Tester : ouvrir `https://<frontend-railway>.up.railway.app` → home Angular doit s'afficher avec données réelles (recettes seedées).

## Phase 6 — SMTP transactionnel (30 min)

L'app envoie des emails (activation compte, mot de passe oublié, contact). Sans SMTP, ces flux sont cassés.

### Choix recommandé : Brevo (ex-Sendinblue)

- Free tier : 300 emails/jour — largement assez pour la cert.
- Account creation : `https://www.brevo.com/`.
- Une fois inscrit, aller dans `SMTP & API` → générer une clé SMTP.

Variables à mettre côté backend Railway :

```env
SMTP_HOST=smtp-relay.brevo.com
SMTP_PORT=587
SMTP_SECURE=false
SMTP_USER=<email-account-brevo>
SMTP_PASSWORD=<smtp-key-générée>
SMTP_FROM=no-reply@<un-domaine-validé-chez-brevo>
```

Brevo demande de valider le `FROM`. Soit :
- Tu valides ton domaine `recipe-shelter.fr` (si tu l'achètes plus tard).
- Soit tu valides ton email perso (rapide) et utilises `arthur.lagenebre@gmail.com` comme FROM — moins propre mais ça marche.

### Alternative : Resend (`https://resend.com`)

- Free tier : 100 emails/jour, 3000/mois.
- API plus simple, mais nécessite un domaine validé (pas d'envoi depuis Gmail).

### Test rapide

Une fois SMTP configuré, créer un compte de test sur le site déployé → vérifier que l'email d'activation arrive.

## Phase 7 — Mettre à jour la documentation (15 min)

Une fois tout en ligne, modifier :

### `documentation/soutenance/demo/scenario-demo.md` ligne 5

```diff
-> Pré-requis : frontend lancé sur `http://localhost:4200`, backend lancé sur `http://localhost:3000`, base de données de démonstration chargée. Sinon le site déployé sur internet avec la base de données de démonstration chargée.
+> Pré-requis : site en ligne à `https://<frontend-railway>.up.railway.app` (recommandé), ou démo locale avec frontend sur `http://localhost:4200`, backend sur `http://localhost:3000` et base de démo chargée.
```

### `documentation/soutenance/demo/comptes-test.md`

Ajouter une section en tête :

```markdown
## Accès

- **Site déployé** : `https://<frontend-railway>.up.railway.app`
- **API** : `https://<backend-railway>.up.railway.app/api/v1`
- **Mot de passe commun des comptes de test** : `Password123!`
```

### `_draft_slides/soutenance-slides.md`

Remplir la zone `[À COMPLÉTER]` URLs avec les URLs Railway, et l'email de contact réel.

### `frontend/README.md` (optionnel)

Ajouter une section `Déploiement` qui pointe vers l'URL prod et explique brièvement la stack (Railway).

## Phase 8 — Test end-to-end (45 min)

Du téléphone (4G, pas Wi-Fi), suivre le **scénario de démo complet** (`documentation/soutenance/demo/scenario-demo.md`) :

1. Tentative login compte inactif → message correct.
2. Tentative login compte banni → message correct.
3. Login user `sophie.leclerc@yahoo.fr` → menu compte apparaît.
4. Recherche recette → résultats.
5. Ajout favori → cœur change d'état.
6. Création recette → brouillon enregistré.
7. Soumission recette → statut "En attente".
8. Login admin → tableau de bord.
9. Validation recette → publication.
10. Modération commentaire.
11. Déconnexion.

Si une étape échoue → fix avant J-day.

Prendre des **captures du site déployé** pour les slides.

## Récap

| Phase | Temps | Risque |
| --- | --- | --- |
| 1. Préparer code | 30 min | Faible (l'app est déjà conçue pour des envs) |
| 2. Provisionner MySQL | 15 min | Faible |
| 3. Charger seed démo | 15 min | Moyen (à valider client MySQL) |
| 4. Déployer backend | 45 min | Moyen (variables ENV à bien câbler) |
| 5. Déployer frontend | 45 min | Moyen (SSR Angular, à tester) |
| 6. SMTP | 30 min | Faible (Brevo bien doc) |
| 7. Doc | 15 min | Faible |
| 8. Test E2E | 45 min | Critique — c'est ici qu'on découvre les bugs |
| **Total** | **3 h 30** | |

Prévoir **un samedi entier** (5-6h) pour être tranquille, pas un soir après un cours.
