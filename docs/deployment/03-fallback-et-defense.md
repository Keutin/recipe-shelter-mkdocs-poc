# Plan B et défense jury

## Si le déploiement n'est pas prêt à J-day

Trois plans de repli, classés du plus crédible au moins crédible. Chacun a un discours associé pour que ce ne soit pas vécu comme un échec.

### Plan B1 — Démo locale projetée + vidéo de secours

**Quand** : si tu as essayé Railway mais le déploiement a planté la veille (bug, mauvaise variable, etc.) et que tu n'as pas eu le temps de réparer.

**Setup** :

- Brancher ton laptop sur le projecteur de la salle.
- Lancer `npm run dev` côté back + `npm start` côté front en local (ou la version SSR : `npm run build && npm run serve:ssr:recipe-shelter`).
- Avoir une **vidéo screencast de 3-5 minutes** déjà enregistrée la veille montrant le scénario de démo bout-en-bout. Si le live casse, fallback vidéo.

**Discours au jury** :

> « Pour la démo, je projette depuis ma machine locale, en mode production. Le site est dans la phase finale de déploiement sur Railway — j'ai voulu privilégier la stabilité de la démo aujourd'hui plutôt qu'une mise en ligne dans la précipitation. L'URL sera fournie en complément après la soutenance. »

C'est défendable si tu as **réellement** travaillé sur Railway et que tu peux montrer l'URL en construction (un service Railway même non finalisé est visible). Mauvais si c'est de l'invention pure.

**Risque** : le jury peut insister (« montrez-moi l'URL »). Si tu n'as rien, c'est gênant.

### Plan B2 — Tunnel ngrok / cloudflared

**Quand** : si tu n'as pas eu le temps de déployer mais tu as une démo locale qui tourne.

**Principe** : ngrok ou cloudflare tunnel expose ton localhost via une URL publique HTTPS. Le jury clique le lien, son navigateur se connecte à ton laptop via le tunnel.

**Setup ngrok** :

```bash
# Installer ngrok : https://ngrok.com/download
ngrok config add-authtoken <token>

# Lancer le backend
cd backend && npm run dev

# Lancer le frontend SSR
cd frontend && npm run build && npm run serve:ssr:recipe-shelter

# Dans deux terminaux séparés, exposer les deux
ngrok http 3000     # backend → URL1
ngrok http 4000     # frontend → URL2

# Mettre à jour environment.prod.ts avec URL1, rebuild, relancer SSR
```

**Setup cloudflared (alternative gratuite illimitée)** :

```bash
# Installer cloudflared : https://github.com/cloudflare/cloudflared
cloudflared tunnel --url http://localhost:4000     # tunnel ad-hoc, URL générée
```

**Discours au jury** :

> « La démo est servie depuis ma machine via un tunnel sécurisé Cloudflare. Vous pouvez ouvrir l'URL sur n'importe quel appareil, c'est la même URL publique. C'est un setup de démonstration — le déploiement permanent sur Railway est documenté dans le repo et sera actif après la soutenance. »

**Risque** : si le wifi de la salle ou ton 4G coupe, le tunnel casse → site down devant le jury. Avoir un partage de connexion 5G prêt en repli.

**Avantage** : ça donne une URL publique cliquable, ce qui satisfait formellement la ligne 34 du cahier.

### Plan B3 — Localhost pur, pas d'URL publique

**Quand** : dernier recours, démo très tardive sans préparation.

**Discours** :

> « Pour des raisons de timing, le déploiement final est planifié post-soutenance. Je vous propose une démo locale en production simulée sur ma machine. Tous les éléments de déploiement sont écrits dans la documentation. »

**C'est le plus risqué**. Le jury va probablement questionner. Préparer des réponses claires (effort estimé, choix de provider, raison du retard).

## Discours à tenir si le déploiement EST fait

Tu n'es pas tiré d'affaire juste parce que c'est en ligne. Le jury va te poser des questions sur **pourquoi ce choix**.

### Q. « Pourquoi Railway et pas un VPS classique ? »

> « Trois raisons. Premièrement, le temps de mise en route : Railway m'a permis de partir de zéro à une démo en ligne en environ trois heures, là où un VPS m'aurait demandé d'installer Node, MySQL, nginx, configurer Let's Encrypt et systemd — un jour entier minimum. Deuxièmement, MySQL est managé : Railway gère les backups et la haute dispo, je n'ai pas à m'en occuper. Troisièmement, c'est un workflow Git push → deploy qui correspond à ce qu'on rencontre en entreprise sur des PaaS comme Heroku, Render ou Fly. Pour un VPS, j'aurais voulu plus de temps pour proprement gérer la conteneurisation Docker. »

### Q. « Et si Railway change ses prix ou ferme ? »

> « Risque réel mais limité dans le périmètre de ce projet. Pour pérenniser au-delà de la soutenance, deux options : migrer vers OVH avec un VPS — j'ai la stack documentée, c'est une journée de travail. Ou passer en self-hosted Docker sur un serveur dédié. Le code est portable, il dépend uniquement de Node 20 + MySQL 8, deux standards stables. »

### Q. « Vous avez du HTTPS ? »

> « Oui, géré automatiquement par Railway via Let's Encrypt. Tu peux le vérifier sur l'URL : le certificat est valide. C'est aussi pour ça que le cookie session porte le flag Secure en prod — il n'est envoyé que sur HTTPS. »

### Q. « Vous avez du monitoring ? »

> « Railway fournit des métriques basiques de mémoire/CPU et les logs en temps réel sur le dashboard. Pour aller plus loin — uptime alerting, error tracking — j'aurais ajouté Sentry pour les erreurs front et UptimeRobot pour le ping. Pas implémenté pour ce projet, mais documenté dans les améliorations possibles. »

### Q. « Comment vous gérez les secrets ? »

> « Variables d'environnement Railway, chiffrées au repos, injectées au runtime. Le repo contient `backend/.env.example` avec la liste des variables attendues mais aucune valeur réelle. Le `.gitignore` exclut explicitement `.env`. Je peux te montrer : aucune clé JWT ni mot de passe dans l'historique Git. »

### Q. « C'est quoi votre stratégie de backup MySQL ? »

> « Railway fait des backups automatiques quotidiens conservés sept jours sur leur plan Hobby. Pour aller plus loin, j'aurais ajouté un dump nocturne via `mysqldump` poussé sur S3 ou un service de backup tiers comme SimpleBackups. Pour ce projet, le backup Railway est suffisant. »

### Q. « Vous avez une CI/CD ? »

Honnête :

> « Pas de CI/CD complète à ce stade. Le déploiement Railway est lui-même Git-push driven, donc à chaque push sur main le service redéploie automatiquement. Mais je n'ai pas ajouté de GitHub Actions pour les tests ou le lint pré-merge. C'est dans le plan d'amélioration : `.github/workflows/test.yml` qui lance `npm test` et `npm run lint` sur chaque PR avant merge. »

### Q. « Vous avez testé la charge ? »

> « Pas de test de charge formel. L'application est dimensionnée pour un usage individuel et démo, pas pour de la production réelle. Pour aller plus loin, j'aurais utilisé k6 ou Artillery pour simuler quelques centaines de users concurrent, et identifié les goulots — probablement la requête de recherche `LIKE` sur les recettes, qui mériterait un index full-text. »

## Anti-patterns à éviter dans le discours

- **« J'ai pas eu le temps »** → faux ton, jury défavorable. Préférer « j'ai priorisé X par rapport au déploiement, voici le plan pour la suite. »
- **« C'est juste un projet d'école »** → tue ton positionnement professionnel.
- **« Heroku c'est mort donc Railway »** → vrai mais grossier ; préférer « PaaS moderne avec MySQL managé inclus. »
- **« Le déploiement c'est easy en 2026 »** → minimise les apprentissages et le travail. À éviter.

## Mantra déploiement

> « Mieux vaut **un lien moche qui marche** qu'un beau domaine vide. »

Le jury veut cliquer et voir l'application. Une URL `recipe-shelter-frontend-production-1a2b.up.railway.app` qui fonctionne **vaut mieux** que `recipe-shelter.fr` qui retourne `ECONNREFUSED`.

## Lien avec les autres livrables jury

- **`_draft_jury/05-pieges-et-defense.md`** contient probablement déjà une Q sur le déploiement et l'absence de Docker. À harmoniser avec ce qui est ici.
- **`_draft_jury/00-questions-transversales.md`** Q sur l'organisation projet → mentionner le déploiement comme jalon final.
- **`_draft_slides/soutenance-slides.md`** slide "Démo" doit afficher l'URL réelle (zone `[À COMPLÉTER]`).
- **`_draft_jury/06-ameliorations-innovantes.md`** Q "ce qu'on ferait avec plus de temps" → CI/CD complète + monitoring + tests de charge.
