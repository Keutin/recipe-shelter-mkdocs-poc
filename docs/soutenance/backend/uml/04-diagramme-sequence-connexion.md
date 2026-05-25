# Diagramme de séquence — Connexion (login)

> Flux complet de la connexion d'un utilisateur, depuis le formulaire Angular
> jusqu'à la pose du cookie de session JWT côté navigateur.
>
> Met en évidence les **points de sécurité** : hash bcrypt, JWT signé,
> cookie HttpOnly + SameSite, vérification du statut utilisateur.

---

## Séquence nominale

```mermaid
sequenceDiagram
    autonumber
    actor U as Utilisateur
    participant FE as Front Angular<br/>(sign-in.ts + AuthService)
    participant API as Express Router<br/>(POST /api/v1/auth/login)
    participant MW as Middlewares<br/>(rateLimiter, cors, cookieParser)
    participant CTRL as AuthController.login
    participant DTO as LoginDto
    participant SVC as AuthService.login
    participant REPO as UserRepository
    participant BC as bcrypt
    participant JWT as jsonwebtoken
    participant DB as MySQL

    U->>FE: Saisit mail + mot de passe
    FE->>API: POST /api/v1/auth/login<br/>{ mail, password }
    API->>MW: Pipeline middlewares
    MW->>MW: Vérifie le rate-limit<br/>(par IP)
    MW->>CTRL: req validé
    CTRL->>DTO: LoginDto.parse(req.body)
    DTO-->>CTRL: { mail, password } typés
    CTRL->>SVC: login({ mail, password })
    SVC->>REPO: findAuthByEmail(mail)
    REPO->>DB: SELECT * FROM Users WHERE Mail = ?
    DB-->>REPO: row
    REPO-->>SVC: UserWithPassword | null

    alt utilisateur introuvable
        SVC-->>CTRL: throw InvalidCredentials
        CTRL-->>FE: 401 { code: 'INVALID_CREDENTIALS' }
    else trouvé
        SVC->>BC: compare(password, user.password)
        BC-->>SVC: boolean

        alt mot de passe invalide
            SVC-->>CTRL: throw InvalidCredentials
            CTRL-->>FE: 401 { code: 'INVALID_CREDENTIALS' }
        else statut non actif
            SVC-->>CTRL: throw AccountNotActive
            CTRL-->>FE: 403 { code: 'ACCOUNT_NOT_ACTIVE' }
        else OK
            SVC->>JWT: sign({ sub: user.id, role }, JWT_SECRET, expiresIn)
            JWT-->>SVC: token
            SVC-->>CTRL: { user, token }
            CTRL->>CTRL: Construit le cookie<br/>HttpOnly, SameSite=lax,<br/>Secure (prod), Max-Age
            CTRL-->>FE: 200 Set-Cookie: rs_session=...<br/>+ { user }
            FE->>FE: SessionService.setUser(user)
            FE-->>U: Redirection /
        end
    end
```

---

## Vérification de la session sur une route protégée

```mermaid
sequenceDiagram
    autonumber
    actor U as Utilisateur connecté
    participant FE as Front Angular
    participant API as Express Router<br/>(ex: GET /api/v1/me/favorites)
    participant CP as cookieParser
    participant AUTH as requireAuth middleware
    participant JWT as jsonwebtoken
    participant REPO as UserRepository
    participant CTRL as FavoritesController.list
    participant DB as MySQL

    U->>FE: Clique "Mes favoris"
    FE->>API: GET /me/favorites<br/>Cookie: rs_session=...
    API->>CP: lit le cookie
    CP->>AUTH: req.cookies.rs_session
    AUTH->>JWT: verify(token, JWT_SECRET)

    alt token invalide ou expiré
        JWT-->>AUTH: throws
        AUTH-->>FE: 401 { code: 'UNAUTHORIZED' }
        FE->>FE: SessionService.clear()<br/>Redirige /sign-in
    else token valide
        JWT-->>AUTH: payload { sub, role }
        AUTH->>REPO: findById(payload.sub)
        REPO->>DB: SELECT * FROM Users WHERE Id = ?
        DB-->>REPO: user row
        REPO-->>AUTH: User
        AUTH->>AUTH: Vérifie user.status === 'active'

        alt user banni ou inactif
            AUTH-->>FE: 403 { code: 'ACCOUNT_NOT_ACTIVE' }
        else actif
            AUTH->>AUTH: req.auth = { userId, username,<br/>roleId, status }
            AUTH->>CTRL: next()
            CTRL->>CTRL: Lit req.auth.userId
            CTRL-->>FE: 200 { favorites: [...] }
        end
    end
```

---

## Notes de défense soutenance

**Sécurité — points à mettre en avant**

- **Hash bcrypt** avec coût configurable (défaut 12) → résistant aux attaques par force brute hors-ligne.
- **JWT signé HS256** avec un secret de 32 bytes minimum (`crypto.randomBytes(32)`).
- **Cookie HttpOnly** → inaccessible au JavaScript du navigateur → protection contre les vols de token par XSS.
- **SameSite=lax** par défaut → protection CSRF de base sur les requêtes cross-site. Si passage en `SameSite=none`, ajouter un token CSRF (à mentionner si questionné, c'est documenté dans le README).
- **Flag Secure** activé en production → le cookie ne transite qu'en HTTPS.
- **Re-vérification du statut user à chaque requête authentifiée** → un user banni est éjecté immédiatement, sans attendre l'expiration du JWT. Coût : un SELECT par requête authentifiée. Alternative possible : cache court (Redis), mais non implémenté car non nécessaire à l'échelle du projet.
- **Rate-limiting** sur les routes auth (`/login`, `/register`, `/forgot-password`) → limite les attaques par énumération de comptes et par brute-force en ligne.

**Pièges fréquents du jury**

- *"Pourquoi pas de refresh token ?"* → Pour un projet de cette taille, un JWT de 7 jours en cookie HttpOnly est un compromis acceptable. Le pattern access+refresh est plus pertinent quand on a besoin de tokens courts révocables (ex. mobile, multi-device). Mention possible en évolution.
- *"Et si je vole le cookie ?"* → C'est la limite. Mitigations : HttpOnly (pas d'accès JS), Secure (pas en HTTP), durée raisonnable, et révocation côté serveur possible via blacklist (non implémenté ici, à mentionner comme évolution).
- *"Pourquoi pas de bibliothèque type Passport.js ?"* → Cohérent avec l'esprit "from scratch" du Bloc 2 : on utilise les briques de base (`bcrypt`, `jsonwebtoken`) sans framework d'auth opinionated.
