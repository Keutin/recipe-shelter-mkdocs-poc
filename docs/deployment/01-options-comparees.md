# Options de déploiement comparées

Quatre options envisageables, classées par adéquation au contexte (étudiant, court délai, stack Node.js + MySQL + Angular SSR, exigence d'une URL publique HTTPS pour le jury).

## Vue d'ensemble

| Critère | Railway | Render | Fly.io | OVH VPS |
| --- | --- | --- | --- | --- |
| Free tier / coût démarrage | ~$5 crédit gratuit | Free tier (sleep après inactivité) | Trial ~$5 crédit | ~3€/mois mini |
| Effort de mise en route | ⭐⭐⭐⭐⭐ Très simple | ⭐⭐⭐⭐ Simple | ⭐⭐⭐ Moyen | ⭐⭐ Complexe |
| MySQL managé inclus | ✅ Oui | ❌ PG préféré, MySQL via marketplace ou externe | ❌ Externe (PlanetScale RIP, Aiven, AWS RDS) | ⚠️ À installer soi-même |
| HTTPS auto | ✅ | ✅ | ✅ | ⚠️ Let's Encrypt à configurer |
| Deploy via Git push | ✅ | ✅ | ✅ (`fly deploy`) | ⚠️ `git pull` + `npm` à scripter |
| SSR Angular (Express custom) | ✅ Détecté auto | ✅ Détecté auto | ✅ Détecté auto | ✅ Possible avec PM2 |
| Custom domain (`.fr`) | ✅ | ✅ | ✅ | ✅ |
| Maintenance | ✅ Quasi nulle | ✅ Quasi nulle | ⚠️ Docker à comprendre | ❌ Update OS, MySQL, nginx |
| **Verdict** | **🏆 Recommandé** | Bon plan B | Pour qui aime Docker | Si Arthur a déjà un VPS |

## Option 1 — Railway (recommandée)

**URL** : `https://railway.app`

**Pourquoi c'est le meilleur choix dans ce contexte** :

- Une seule plateforme pour les 3 composants (backend, MySQL, frontend SSR).
- MySQL managé en 2 clics, variables d'environnement injectées automatiquement.
- Deploy via GitHub : Railway connecte le repo, build sur push.
- Logs centralisés, redéploiement en un clic.
- HTTPS automatique avec sous-domaine `*.up.railway.app`.

**Modèle de coûts (mai 2026, à vérifier)** :

- Plan Trial : ~5$ de crédit gratuit, consomme à l'usage.
- Plan Hobby : ~5$/mois plat pour usage modeste — largement dans le budget d'un étudiant si tu paies un mois.
- Pour la durée de l'évaluation jury (quelques jours / semaines), le trial suffit probablement.

**Inconvénients** :

- Pas un provider français (US). Sur le plan certif, sans incidence — le cahier ne demande pas un hébergeur FR.
- En cas de pic d'usage massif (TikTok virality), coût grimpe. Pas le cas en cert.
- Peut avoir des sleep states sur trial, à vérifier.

**Limite vraiment bloquante** : aucune connue dans ce contexte.

## Option 2 — Render

**URL** : `https://render.com`

**Atouts** :

- Free tier intéressant pour les Web Services Node.
- Interface très propre.

**Frictions pour Recipe Shelter** :

- MySQL n'est pas first-class chez Render : il préfère PostgreSQL. Pour MySQL, il faut soit un add-on tiers (Aiven, PlanetScale alternative, Railway externe) soit migrer le schéma vers PG — gros chantier.
- Free tier : le service Web s'endort après ~15 min d'inactivité → premier hit du jury = 50s de cold start. Mauvais signal.

**Verdict** : Bon plan B si Railway ne convient pas, mais MySQL te complique la vie. Si tu pars sur Render, prévois soit l'upgrade payant (sans sleep), soit un service de keep-alive (UptimeRobot ping).

## Option 3 — Fly.io

**URL** : `https://fly.io`

**Atouts** :

- Très bon pour des apps Node + Docker.
- Régions multiples (peux mettre EU si tu veux).
- Postgres managé excellent.

**Frictions pour Recipe Shelter** :

- Demande de **comprendre Docker** (écrire un `Dockerfile`, déployer via `fly deploy`).
- MySQL pas first-class : Postgres préféré, MySQL via Aiven / AWS RDS externe.
- Trial limité dans le temps.

**Verdict** : Bonne option si tu veux apprendre Docker pour Bloc 2, mais c'est un détour qui peut coûter 1-2 jours pour rien si tu débutes.

## Option 4 — OVH VPS

**URL** : `https://www.ovhcloud.com/fr/vps/`

**Atouts** :

- Hébergeur français — argument bonus très léger pour un jury français (anecdotique, le cahier n'exige rien à ce sujet).
- VPS d'entrée à ~3-4€/mois.
- Contrôle total (tu peux montrer `nginx`, `systemd`, `mysql` au jury → argument "j'ai déployé moi-même").

**Frictions** :

- **Tout à faire à la main** : installer Node, MySQL, nginx, Let's Encrypt, configurer systemd, gérer les logs.
- Si quelque chose casse en prod la veille, debug à 23h sur SSH.
- Sans CI/CD, chaque déploiement = `git pull && npm ci && npm run build && pm2 restart`.

**Verdict** : Option viable seulement si tu as déjà un VPS qui tourne et où tu sais ce que tu fais. Pour partir de zéro à J-X, c'est trop de surface d'erreur.

## Décision recommandée

**Railway** pour les raisons suivantes :

1. **Temps de mise en route minimal** (~3h pour un débutant motivé).
2. **MySQL managé inclus** — pas de chantier d'install à coté.
3. **Un seul tableau de bord pour les 3 services** → simplifie la démo au jury si on doit montrer la config.
4. **Git push → deploy** → tu prends l'habitude du workflow CI/CD sans installer Jenkins.

Pour pousser au-delà (si soutenance dans plusieurs semaines et envie de produire un livrable plus impressionnant pour le jury), considérer **Fly.io avec Dockerfile écrit à la main** comme argument "j'ai compris la conteneurisation". Mais c'est un projet de quelques jours, à équilibrer.

## Coût total estimé pour la durée de l'évaluation

- Railway trial + 1 mois hobby si nécessaire : **5 à 15 $**.
- Si tu prends un domaine custom `recipe-shelter.fr` chez OVH ou Gandi : **~12 €/an**. **Optionnel** — un sous-domaine `*.up.railway.app` suffit pour le jury.

Total réaliste pour passer la certif : **moins de 30 €**.

## Ce qu'il faut savoir AVANT de cliquer "Deploy"

Trois éléments doivent être prêts dans le code avant déploiement (sinon ça plante en prod) :

1. **Le backend doit écouter sur `process.env.PORT`** (et pas un port hardcodé). `.env.example:6` indique déjà `PORT=3000` — vérifier dans `src/server.ts` que `app.listen(process.env.PORT || 3000)` est bien utilisé. Si Railway injecte un port, le code doit le respecter.

2. **Les CORS doivent autoriser le domaine frontend en prod**. Variable `CORS_ALLOWED_ORIGINS` (`backend/.env.example:30`) à mettre à jour avec l'URL Railway frontend.

3. **Le cookie session doit avoir `Secure=true` en prod**. Variable `AUTH_SESSION_COOKIE_SECURE` (`.env.example:21`). Sans HTTPS, le cookie n'est pas envoyé.

Ces trois points sont validés dans le plan pas-à-pas du fichier suivant.
