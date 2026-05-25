# ADR-003 — JWT en cookie HttpOnly (vs header Authorization)

## Statut

Accepté · 2026-05-25

## Contexte

L'application nécessite une authentification persistante entre les
requêtes. Deux familles de stratégies :

1. **Token dans le header `Authorization: Bearer <token>`** — stockage
   côté client en `localStorage` ou en mémoire.
2. **Token dans un cookie HttpOnly** — posé par le serveur, renvoyé
   automatiquement par le navigateur.

Le frontend est Angular SSR servi sur le même domaine que l'API (cf.
`recipe-shelter.fr` et `api.recipe-shelter.fr`).

## Décision

Utiliser un **cookie HttpOnly nommé `rs_session`** contenant le JWT
signé HS256.

Paramètres :

- `HttpOnly` : true (toujours)
- `Secure` : true en production (HTTPS uniquement)
- `SameSite` : `lax` par défaut (configurable via env)
- `Max-Age` : aligné sur `JWT_EXPIRES_IN` (défaut 7 jours)
- `Domain` : optionnel, défini par env pour les déploiements
  multi-sous-domaines

À la connexion, le serveur :

1. Vérifie email + mot de passe (bcrypt compare).
2. Vérifie le statut user (`active` requis).
3. Signe un JWT avec `{ sub: userId, username, roleId, status }`.
4. Pose le cookie via `Set-Cookie`.
5. Renvoie le profil utilisateur en JSON (sans le token — il n'est
   jamais exposé au JavaScript du navigateur).

À chaque requête authentifiée, le middleware `requireAuth`
(`src/middlewares/require-auth.ts`) :

1. Lit le cookie.
2. Vérifie la signature JWT.
3. **Re-charge le user depuis MySQL** pour vérifier `status === 'active'`.
4. Attache `req.auth = { userId, username, roleId, status }`.

## Alternatives considérées

### JWT dans `Authorization: Bearer`

- **Pour** : pattern standard des APIs publiques (CORS-friendly,
  stateless).
- **Contre** : si stocké en `localStorage`, **vulnérable à XSS** —
  n'importe quel script tiers compromis peut lire le token.
- **Contre** : doit être ajouté manuellement à chaque requête côté
  client (intercepteur Angular nécessaire).

### Session serveur classique (cookie ID + table `Sessions`)

- **Pour** : révocation triviale (supprimer la ligne).
- **Contre** : état serveur, charge BDD supplémentaire à chaque requête
  (déjà payée ici par la vérif de statut, mais cumulable).
- **Contre** : moins idiomatique pour une API REST.

## Conséquences

### Positives

- **Protection XSS** : un script malveillant injecté dans la page ne
  peut pas lire le cookie HttpOnly.
- **Protection CSRF basique** : `SameSite=lax` bloque les requêtes
  cross-site déclenchées par un lien malveillant.
- **Simplicité côté frontend** : le navigateur attache automatiquement
  le cookie, pas d'intercepteur explicite à maintenir pour l'auth
  (l'intercepteur Angular existe mais traite d'autres cas).
- **Révocation effective sans table de sessions** : le middleware
  re-vérifie le statut user à chaque requête. Un user banni est éjecté
  immédiatement, sans attendre l'expiration du JWT.

### Négatives

- **Coût** : 1 `SELECT * FROM Users WHERE Id = ?` par requête
  authentifiée. À l'échelle d'un projet pédagogique, négligeable. À
  l'échelle d'une production massive, ajouter un cache court (Redis
  TTL 30s) résout le problème.
- **CORS complexe** : `credentials: true` nécessite des origines
  explicites (pas de wildcard). C'est implémenté dans `src/app.ts` ligne
  77.
- **Si passage en `SameSite=none`** (front et API sur domaines vraiment
  cross-site) : nécessite d'ajouter une protection CSRF
  (synchronizer token). Documenté dans le README backend.

### À surveiller

- Si le projet évolue vers du mobile (app native), il faudra basculer
  sur `Authorization: Bearer` car les apps mobiles ne gèrent pas les
  cookies aussi naturellement qu'un navigateur.
