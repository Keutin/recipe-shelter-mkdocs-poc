---
title: Collections Postman
description: Collections Postman pour tester l'API Recipe Shelter
tags:
  - bloc-2
  - backend
  - api
---

# Collections Postman

> 12 collections Postman couvrant les **53 endpoints** de l'API Recipe Shelter
> (voir [OpenAPI](../openapi/README.md) pour la spec complète). Chaque
> collection couvre un domaine fonctionnel, avec exemples de payloads et
> assertions de base.

## Import dans Postman

1. Ouvrir Postman → **Import** (bouton en haut à gauche).
2. Glisser-déposer le fichier `.json` souhaité depuis ce dossier
   (ou via l'URL GitHub directement).
3. Configurer l'environnement : variable `baseUrl` =
   `http://localhost:3000/api/v1` (local) ou l'URL Railway (prod).
4. Pour les routes authentifiées, exécuter d'abord `auth.login` —
   le cookie de session `rs_session` est posé automatiquement et réutilisé
   par les requêtes suivantes.

## Collections disponibles

| Collection | Domaine | Endpoints couverts |
| --- | --- | --- |
| [`root.postman_collection.json`](root.postman_collection.json) | Racine API | `GET /`, `GET /api/v1` |
| [`health.postman_collection.json`](health.postman_collection.json) | Santé / readiness | `GET /health`, `GET /ready` |
| [`auth.postman_collection.json`](auth.postman_collection.json) | Authentification | `register`, `login`, `logout`, `validate-email`, `reset-password`, … |
| [`users.postman_collection.json`](users.postman_collection.json) | Utilisateurs (self) | `GET /users/me`, profil public, modification email/username/password |
| [`recipes.postman_collection.json`](recipes.postman_collection.json) | Recettes publiques | listing, recherche, fiche détail, recettes d'un user |
| [`comments.postman_collection.json`](comments.postman_collection.json) | Commentaires | poster, répondre, modifier, supprimer |
| [`favorites.postman_collection.json`](favorites.postman_collection.json) | Favoris | ajouter, retirer, lister ses favoris |
| [`contact.postman_collection.json`](contact.postman_collection.json) | Formulaire de contact | `POST /contact` |
| [`reference-data.postman_collection.json`](reference-data.postman_collection.json) | Données de référence | catégories, unités, listes statiques |
| [`admin-recipes.postman_collection.json`](admin-recipes.postman_collection.json) | Admin — recettes | modération (approve, reject), suppression |
| [`admin-comments.postman_collection.json`](admin-comments.postman_collection.json) | Admin — commentaires | modération, suppression |
| [`admin-users.postman_collection.json`](admin-users.postman_collection.json) | Admin — utilisateurs | bannir, débannir, lister |

## Authentification

L'API utilise un **cookie de session HttpOnly + Secure + SameSite=Lax**
(JWT HS256 signé, voir
[ADR-003](../adr/adr-003-jwt-cookie-httponly.md)).

Pour les collections admin, le compte exécutant doit avoir le rôle
`admin` (voir [comptes de test](../../demo/comptes-test.md) pour le seed
de dev).

## Liens

- [Spec OpenAPI 3.1 (Redoc)](../openapi/README.md) — référence exhaustive.
- [Catalogue d'erreurs](../errors.md) — ~140 codes d'erreur retournés par l'API.
- [Architecture backend](../architecture.md) — pipeline Express,
  middlewares, gestion des erreurs.
