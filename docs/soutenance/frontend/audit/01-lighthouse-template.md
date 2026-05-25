# Lighthouse — Template d'audit à remplir

> **À compléter par Arthur** après exécution. Les scores ci-dessous sont des cases vides — il faut faire tourner l'audit pour les remplir.

## Procédure

### Préparer l'environnement

```bash
cd C:\DEV\ARTHUR\RECETTES\frontend
npm install                # si pas déjà fait
npm run build              # build production
npm run serve:ssr:recipe-shelter
# (vérifier la commande exacte dans package.json — peut être "serve:ssr" ou "start:prod")
```

L'application doit répondre sur `http://localhost:4000` (port SSR Angular) ou `http://localhost:4200`. Noter le port choisi.

> **Pourquoi le build de production** : auditer le mode `ng serve` est trompeur (sourcemaps, no minify, no tree-shake). Lighthouse pénalise le bundle gonflé du mode dev. **Toujours auditer la prod.**

### Lancer Lighthouse

#### Méthode A — DevTools (rapide)

1. Chrome navigation privée (Ctrl+Shift+N).
2. Désactiver les extensions (sinon Lighthouse refuse ou les pénalise).
3. Ouvrir l'URL cible.
4. F12 → onglet `Lighthouse`.
5. Mode : `Navigation (par défaut)`. Catégories : tout cocher. Appareil : `Mobile` (les scores mobiles sont plus exigeants, c'est le minimum à présenter).
6. `Analyser le chargement de la page`.
7. Une fois le rapport prêt : `⋮` en haut → `Enregistrer au format HTML` (pour preuve) + capture PNG du score.

#### Méthode B — CLI (reproductible)

```bash
npm install -g lighthouse
lighthouse http://localhost:4000 --view --preset=mobile --output=html --output-path=./lh-home.html
```

Reproduire pour chaque URL. Avantage : on peut scripter et comparer dans le temps.

### URLs cibles

Choix de 5 URLs pour couvrir les types de pages :

| # | URL | Pourquoi |
| --- | --- | --- |
| 1 | `/` | Page d'accueil, vitrine, hero image, LCP critique. |
| 2 | `/recipes` | Liste paginée, beaucoup d'images, recherche. |
| 3 | `/recipes/<slug>` | Fiche détail, SSR-rendu, commentaires async. |
| 4 | `/sign-in` | Formulaire simple, doit être proche du 100. |
| 5 | `/admin/dashboard` | Rendu client uniquement (pas SSR), nécessite session admin → noter dans le rapport que c'est attendu différent. |

## Grille de scores

> Remplir après audit. Les cases entre `[]` sont à compléter. Noter les chiffres exacts du rapport Lighthouse.

### Audit Mobile

| Page | Performance | Accessibility | Best Practices | SEO |
| --- | --- | --- | --- | --- |
| `/` | [__] | [__] | [__] | [__] |
| `/recipes` | [__] | [__] | [__] | [__] |
| `/recipes/<slug>` | [__] | [__] | [__] | [__] |
| `/sign-in` | [__] | [__] | [__] | [__] |
| `/admin/dashboard` | [__] | [__] | [__] | [__] |

### Audit Desktop

| Page | Performance | Accessibility | Best Practices | SEO |
| --- | --- | --- | --- | --- |
| `/` | [__] | [__] | [__] | [__] |
| `/recipes` | [__] | [__] | [__] | [__] |
| `/recipes/<slug>` | [__] | [__] | [__] | [__] |
| `/sign-in` | [__] | [__] | [__] | [__] |
| `/admin/dashboard` | [__] | [__] | [__] | [__] |

### Core Web Vitals (rapport Lighthouse Mobile)

> LCP = Largest Contentful Paint (vitesse d'apparition du plus gros élément). CLS = Cumulative Layout Shift (stabilité visuelle). TBT = Total Blocking Time (réactivité JS).

| Page | LCP (s) | CLS | TBT (ms) | Verdict |
| --- | --- | --- | --- | --- |
| `/` | [__] | [__] | [__] | [Bon / Moyen / À améliorer] |
| `/recipes` | [__] | [__] | [__] | [__] |
| `/recipes/<slug>` | [__] | [__] | [__] | [__] |

Seuils Google : LCP < 2.5s = bon, CLS < 0.1 = bon, TBT < 200ms = bon.

## Captures à joindre

| Capture | Fichier suggéré |
| --- | --- |
| Score Lighthouse home mobile | `documentation/soutenance/frontend/audit/lh-home-mobile.png` |
| Score Lighthouse home desktop | `documentation/soutenance/frontend/audit/lh-home-desktop.png` |
| Score Lighthouse recipe-detail | `documentation/soutenance/frontend/audit/lh-recipe-detail.png` |
| Score Lighthouse sign-in | `documentation/soutenance/frontend/audit/lh-sign-in.png` |
| Rapport HTML complet (au moins home) | `documentation/soutenance/frontend/audit/lh-home-report.html` |

## Pré-prédictions (basé sur l'audit statique)

> Ces estimations viennent de l'analyse statique du code. Elles servent de **repères** pour détecter une régression ou un faux positif. Les chiffres réels remplacent ces hypothèses dans la grille ci-dessus.

| Page | Performance attendue | Accessibility attendue | Best Practices attendue | SEO attendu |
| --- | --- | --- | --- | --- |
| `/` | 75–90 (hero préchargé, WebP, mais pas de NgOptimizedImage) | 85–92 (sémantique OK, mais skip link manquant, focus route absent) | 90–100 (HTTPS local OK, pas d'erreurs console) | 70–85 (title statique, OG manquants, robots OK) |
| `/recipes` | 70–85 (liste paginée, images lazy) | 88–95 | 95–100 | 75–85 |
| `/recipes/<slug>` | 75–90 (SSR + skeleton) | 85–95 | 95–100 | 65–80 (pas de title dynamique, pas de JSON-LD recipe) |
| `/sign-in` | 90–100 | 90–98 (forms OK mais aria-invalid manquant) | 95–100 | 80–90 |
| `/admin/dashboard` | N/A en SSR (CSR forcé), mesure différente | 80–90 | 90–100 | N/A (noindex souhaité) |

## Notes attendues du jury sur Lighthouse

**Si scores autour de 80–95** → c'est crédible et défendable. Au-dessus de 95 partout, le jury peut soupçonner un environnement trop favorable ou des seuils tordus.

**Si scores < 80 sur la production** → expliquer la cause précise (image lourde non optimisée, library tierce, etc.) plutôt que minimiser. Identifier l'écart entre `localhost` et la production déployée.

**Discours type** :

> « Sur la home, Lighthouse mobile renvoie [X] en performance et [Y] en accessibilité. Le score performance est tiré vers le bas par l'absence d'image responsive — on n'a pas branché `NgOptimizedImage` parce que [raison]. C'est la première amélioration sur ma roadmap, documentée dans le plan de remédiation. »

C'est plus solide qu'un « on a 99 partout grâce à Angular ».
