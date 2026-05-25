# Déploiement — diagnostic et plan d'action

Ce dossier traite le point #9 de la roadmap : **mettre Recipe Shelter en ligne avant la soutenance**. C'est le seul point restant qualifié d'**éliminatoire** : le cahier des charges l'exige explicitement comme livrable.

## Diagnostic (2026-05-25)

| Élément | État | Évidence |
| --- | --- | --- |
| Domaine `recipe-shelter.fr` | ❌ Non enregistré au DNS | `nslookup recipe-shelter.fr` → `Non-existent domain` |
| Domaine `api.recipe-shelter.fr` | ❌ Non résolu | `nslookup api.recipe-shelter.fr` → `Non-existent domain` |
| Backend déployé | ❌ Aucune trace | Pas de Dockerfile, Procfile, fly.toml, render.yaml dans `backend/` |
| Frontend déployé | ❌ Aucune trace | Pas de vercel.json, netlify.toml, Dockerfile dans `frontend/` |
| CI/CD | ❌ Aucun workflow | Pas de `.github/workflows/` dans les 3 repos |
| Variables prod configurées | ⚠️ Côté code OK, côté infra inconnu | `frontend/src/environments/environment.prod.ts:apiBaseUrl='https://api.recipe-shelter.fr/api/v1'` (cible inexistante) |
| `comptes-test.md` / `scenario-demo.md` | ⚠️ Mentionnent un site déployé | `Sinon le site déployé sur internet avec la base de données de démonstration chargée.` (`documentation/soutenance/demo/scenario-demo.md:5`) |

**Conclusion** : la documentation parle d'un déploiement qui **n'existe pas**. À ce stade, soutenance avec démo locale uniquement → risque d'élimination, ou en tout cas demande systématique du jury (« pourquoi pas en ligne ? »).

## Ce qu'exige le cahier des charges

- Ligne 34 (Bloc 1) : « **Démo en Ligne** : Un lien vers une version en ligne du site pour une démonstration facile. »
- Ligne 107 (Bloc 3) : « **Démo en Ligne** : Une version déployée de l'application accessible en ligne. »
- Ligne 118 (modalités) : « le déploiement de l'application » est explicitement cité comme aspect évalué.

Le cahier ne précise **pas** le provider ni les performances attendues — il faut juste que le jury puisse cliquer sur un lien et voir l'application fonctionner.

## Périmètre minimal à mettre en ligne

| Composant | Obligatoire ? | Pourquoi |
| --- | --- | --- |
| Backend Express + MySQL | Oui | Sans backend, le frontend ne montre rien. |
| Frontend Angular SSR | Oui | C'est l'objet du Bloc 3. |
| Base de données de démo seedée | Oui | Comptes de test (`paul.bernard`, `sophie.leclerc`, `admin_demo`) doivent fonctionner. |
| Emails (SMTP) | Optionnel | Si SMTP en prod ne fonctionne pas, ce n'est pas éliminatoire — mais alors la fonctionnalité activation email tombe. Solution : SMTP transactionnel gratuit (Brevo, Resend, Mailtrap free tier). |
| Domaine custom `.fr` | **Non** | Une URL `*.up.railway.app` ou `*.vercel.app` suffit pour le jury. Domaine custom = bonus de présentation, pas exigence. |
| HTTPS | Oui | Cookie session avec `Secure=true` en prod. Tous les providers modernes fournissent HTTPS automatiquement. |

## Structure de ce dossier

| Fichier | Contenu |
| --- | --- |
| [01-options-comparees.md](01-options-comparees.md) | 4 options de déploiement (Railway, Render, Fly.io, OVH VPS) comparées sur 8 critères. Recommandation finale. |
| [02-plan-railway.md](02-plan-railway.md) | Plan pas-à-pas avec commandes exactes pour l'option recommandée (Railway pour backend+DB+frontend). ~3 heures de travail. |
| [03-fallback-et-defense.md](03-fallback-et-defense.md) | Plan B si déploiement impossible avant D-day + ce qu'on dit au jury dans chaque cas. |

## Walkthrough pour Arthur

### Étape 1 — Décider du provider (15 min)

Lire [01-options-comparees.md](01-options-comparees.md). La recommandation par défaut est **Railway**. Raisons :
- 1 seule plateforme pour les 3 composants (back + MySQL + front),
- Free trial / hobby plan suffisant pour la durée de l'évaluation,
- Deploy via Git push, pas de Docker à écrire,
- MySQL managed inclus.

Si tu as déjà un OVH/VPS en cours d'utilisation, l'option 4 peut être plus simple.

### Étape 2 — Déployer (~3 h)

Suivre [02-plan-railway.md](02-plan-railway.md). Étapes :

1. Créer compte Railway
2. Provisionner MySQL service → récupérer les variables d'environnement
3. Charger le dump `database/reset_demo.sql`
4. Déployer le backend (repo `recipe-shelter-backend`)
5. Configurer les ENV (JWT_SECRET, SMTP, CORS_ALLOWED_ORIGINS, FRONTEND_BASE_URL)
6. Déployer le frontend (repo `recipe-shelter-frontend`) avec `apiBaseUrl` mis à jour
7. Tester end-to-end avec les comptes de démo

### Étape 3 — Mettre à jour `environment.prod.ts` et `scenario-demo.md`

Une fois les URLs Railway connues, remplacer dans le code :

- `frontend/src/environments/environment.prod.ts` → `apiBaseUrl: 'https://<backend>.up.railway.app/api/v1'`
- `backend/.env` Railway → `CORS_ALLOWED_ORIGINS=https://<frontend>.up.railway.app`, `FRONTEND_BASE_URL=https://<frontend>.up.railway.app`
- `documentation/soutenance/demo/scenario-demo.md` ligne 5 → indiquer l'URL exacte
- `documentation/soutenance/demo/comptes-test.md` → idem
- Slides soutenance (`_draft_slides/soutenance-slides.md`) → remplir la zone `[À COMPLÉTER]` avec les URLs

### Étape 4 — Tester depuis un appareil autre que le tien

Idéalement : téléphone en 4G (pas Wi-Fi maison qui pourrait avoir un cache DNS). Faire le scénario de démo complet, prendre des captures. Si quelque chose casse, le réparer **avant** la soutenance, pas le jour J.

### Étape 5 — Si tu ne peux pas déployer

Voir [03-fallback-et-defense.md](03-fallback-et-defense.md). Trois plans de repli, du plus crédible au moins crédible :

- **B1** : démo locale projetée + vidéo de secours (acceptable, mais demande une justification crédible du « pourquoi pas en ligne »).
- **B2** : ngrok/cloudflared sur ton poste pendant la soutenance (URL publique tunnel vers ton localhost) — risqué, mais accessible.
- **B3** : repli sur localhost pur — mauvaise option, présenter comme dernier recours.

## Délai critique

- Si la soutenance est dans **plus de 2 semaines** → option Railway recommandée, large marge.
- Si la soutenance est dans **moins de 7 jours** → option Railway possible mais sans marge, ou tunnel ngrok en repli.
- Si la soutenance est **demain** → option B2 (ngrok) ou B3 (localhost).

À adapter selon la date réelle (à jour : 2026-05-25 — confirmer la date jury).

## Lien avec les autres livrables

- **Slides** (`_draft_slides/soutenance-slides.md`) : la zone `[À COMPLÉTER]` URLs repos/site doit être remplie après déploiement.
- **Scénario démo** (`documentation/soutenance/demo/scenario-demo.md`) : indique déjà "site déployé" en repli — il faut que ce site existe vraiment.
- **Manuel utilisateur** (`_draft_user_guide/`) : ne mentionne pas d'URL spécifique, OK.
- **Banque jury** (`_draft_jury/05-pieges-et-defense.md`) : contient probablement déjà une Q sur le déploiement — synchroniser avec la version finale.
