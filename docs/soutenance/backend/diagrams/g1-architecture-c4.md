# G1 — Architecture C4 Container

Ce diagramme présente la vue **Container** (C4 niveau 2) de Recipe Shelter. L'objectif est de visualiser les briques exécutables du système et leurs interactions, sans détailler les composants internes.

Le périmètre couvre le frontend Angular avec rendu universel (SSR + browser), l'API backend Express, la base MySQL et le service SMTP utilisé pour les mails transactionnels.

```mermaid
C4Container
    title Recipe Shelter — Vue Container

    Person(visiteur, "Visiteur", "Non authentifie : consulte les recettes publiques")
    Person(membre, "Membre", "Authentifie : poste recettes, commentaires, favoris")
    Person(admin, "Administrateur", "Modere recettes, commentaires et utilisateurs")

    System_Boundary(rs, "Recipe Shelter") {
        Container(front, "Frontend Angular SSR", "Node 22, Angular 21", "Rendu universel : SSR ou CSR selon le render mode par route. Sert le bundle au navigateur.")
        Container(api, "API Backend", "Node 22, Express 5, TypeScript", "API REST /api/v1. Authentification JWT via cookie httpOnly. Architecture 3 couches : api / services / repositories.")
        ContainerDb(db, "Base MySQL", "MySQL 8, mysql2 pool", "Recettes, utilisateurs, commentaires, favoris, tags, ingredients, equipements.")
    }

    System_Ext(smtp, "SMTP", "Brevo en prod, nodemailer", "Mails transactionnels : reset password, validation email, contact.")

    Rel(visiteur, front, "Navigue", "HTTPS")
    Rel(membre, front, "Navigue et publie", "HTTPS")
    Rel(admin, front, "Modere", "HTTPS")

    Rel(front, api, "Appelle l API REST", "HTTP + cookie auth_token")
    Rel(api, db, "Lit et ecrit", "TCP, pool mysql2")
    Rel(api, smtp, "Envoie des mails", "SMTP TLS")

    UpdateLayoutConfig($c4ShapeInRow="3", $c4BoundaryInRow="1")
```

## Notes

- **Pas de container CDN ni de storage objet.** Les images de recettes sont stockees comme **URLs externes** (champ `coverImageUrl` cote DTO, `RecipeCoverImage` en base) — voir `backend/src/repositories/recipes/recipe.types.ts` et `backend/src/api/recipes/recipes.dto.ts`. Aucun upload binaire cote backend (pas de `multer`, pas de S3, pas de dossier `uploads/`). L'utilisateur fournit une URL deja hebergee ailleurs. Si plus tard on ajoute l'upload natif, il faudra introduire un container `Object Storage` (S3, R2, ou disque local) et le tracer ici.
- **Front unique, deux contextes d'execution.** "Frontend Angular SSR" represente un **seul bundle Angular** execute soit cote Node (rendu serveur sur la premiere requete, render mode SSR / prerender par route), soit cote navigateur (hydratation et navigations suivantes en CSR). En C4 Container on les regroupe car ils partagent le code, la build et le deploiement. Une vue Component pourrait separer le moteur SSR Node et le bundle browser si besoin.
- **JWT en cookie httpOnly.** L'API depose un cookie `auth_token` (nom configurable via `env.auth.sessionCookieName`) sur `/api/v1/auth/login`. Toutes les requetes authentifiees (depuis SSR ou browser) le re-transmettent automatiquement. CORS est configure avec `credentials: true` et une whitelist d'origines explicites.
