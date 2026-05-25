# Synthèse exécutive — Recipe Shelter / Cahier des charges

> Lecture cible : ≤ 5 min. Pour le détail ligne par ligne, voir [`compliance-matrix.md`](./compliance-matrix.md).
> Snapshot : 2026-05-25.

## Feu tricolore par bloc

| Bloc | Couleur | Lecture rapide |
|------|---------|---------------|
| **Bloc 1 — Frontend** | 🟠 Orange | Code OK, responsive OK. **Manque démo en ligne** (commun aux 3 blocs) + Lighthouse à exécuter + BrowserStack absent (défense alternative possible). |
| **Bloc 2 — Backend from scratch** | 🟢 Vert | Implémentation très solide. 53 endpoints, 16 tables, 29 tests, schéma SQL versionné. Toute la doc tech est rédigée mais en draft — il faut que les handoffs soient exécutés par Arthur. |
| **Bloc 3 — Framework** | 🟠 Orange | Angular 21 SSR complet, 64 specs Vitest, guide utilisateur prêt. **Manque démo en ligne** (idem 1.12 / 3.9). Doc tech rédigée, handoff `_draft_frontend_docs/` à finaliser (README walkthrough à créer). |
| **Transverse / Soutenance** | 🟡 Jaune | Slides Marp (28 slides), banque ~100 Q&A, ADRs (10), narration from-scratch — tout en place mais à personnaliser (date, jurés, scores réels). |

**Taux livrable** (OK + Ready, hors N/A) = **84 %** (48/57) si Arthur exécute tous les handoffs draft → repos. Taux soutenable (incluant gaps avec défense préparée) = **95 %**.

## Top 5 risques classés

1. **🚨 Démo en ligne absente — ÉLIMINATOIRE.** Le cahier l.34 (Bloc 1), l.107 (Bloc 3) et l'évaluation RNCP l.118 exigent une appli accessible. `nslookup recipe-shelter.fr` → NXDOMAIN. Aucun Dockerfile, fly.toml, render.yaml, .github/workflows/ dans les 3 repos. **Plan détaillé** : [`_draft_deployment/02-plan-railway.md`](../deployment/02-plan-railway.md) (~3h30, Railway recommandé). **Plan B** si délai trop court : tunnel ngrok/cloudflared documenté dans [`03-fallback-et-defense.md`](../deployment/03-fallback-et-defense.md).

2. **🟠 Audit Lighthouse non exécuté.** Le cahier l.21, 38 nomme Lighthouse explicitement. Template prêt [`01-lighthouse-template.md`](../soutenance/frontend/audit/01-lighthouse-template.md), Arthur doit exécuter (5 URLs × mobile+desktop, ~30 min après `npm run build`). À faire impérativement *avant* la soutenance, idéalement après le P1 du [plan de remédiation](../soutenance/frontend/audit/04-plan-remediation.md) (~1h20) pour des scores plus présentables.

3. **🟡 Handoffs draft → documentation repo non exécutés.** 10 livrables prêts en `_draft_*/` mais pas encore copiés dans le repo `documentation` :
   - `_draft_adr/` (5 ADRs backend) → `documentation/soutenance/backend/adr/`
   - `_draft_adr_frontend/` (5 ADRs Angular) → `documentation/soutenance/frontend/adr/`
   - `_draft_backend_docs/` Vague 1 + 2 (architecture, errors, securite, data-dict, tests, env, 8 diagrammes Mermaid, OpenAPI) → `documentation/soutenance/backend/`
   - `_draft_bloc1_audit/` (RGAA grid, Lighthouse template, plan rem, défense) → `documentation/soutenance/frontend/audit/`
   - `_draft_user_guide/` (5 fichiers) → `documentation/manuel-utilisateur/`
   - `_draft_frontend_docs/frontend-architecture.md` → `documentation/soutenance/frontend/architecture.md` (README walkthrough à produire — point 4 ci-dessous)

   Chaque draft a un README walkthrough sauf `_draft_frontend_docs/`. Toujours sur le modèle : branche `docs/<short-name>` → copie → commit conventional → PR.

4. **🟡 `_draft_frontend_docs/` orphelin.** Document de 1018 lignes, **précis à 8/9** sur audit factuel, mais sans README de handoff. Une seule petite inexactitude à corriger : « 15 modules Bootstrap » → 16 (en 2 endroits). README walkthrough à créer en complément (modèle existant dans `_draft_backend_docs/README.md`).

5. **🟠 BrowserStack non couvert** (cahier l.21, 37). Pas de compte ni de scripts. **Défense préparée** : « Tests responsivité via Chrome DevTools mobile/desktop + captures matérielles iPhone/iPad ; voir 18 captures `Responsive/` ». Acceptable mais le jury peut y revenir — réponse à drillrer.

## Plan d'action minimal pré-soutenance

Hypothèse : il reste **2 semaines** avant la soutenance (à adapter selon date réelle).

### J-14 à J-10 — déblocage éliminatoire
1. **Lancer le déploiement** ([`02-plan-railway.md`](../deployment/02-plan-railway.md)) — provider Railway, 3 prérequis code (PORT env, CORS, Cookie Secure), SMTP Brevo. Risque #1 ⇒ J0.
2. Si trop court, basculer plan B tunnel ([`03-fallback-et-defense.md`](../deployment/03-fallback-et-defense.md)).

### J-10 à J-7 — handoffs documentation
3. Copier les drafts dans le repo `documentation` selon leurs walkthroughs respectifs (compter ~3h cumulées). Ordre suggéré :
   - Backend docs (Vague 1 + 2) → branche `docs/backend-soutenance-vague1`/`vague2`
   - ADRs backend + frontend → branches `docs/adr-architecture-decisions` + `docs/frontend-adrs`
   - Frontend architecture (créer README walkthrough avant) → branche `docs/frontend-architecture`
   - Audit Bloc 1 → `docs/bloc1-audit`
   - User guide → `docs/manuel-utilisateur`
4. **Pousser branches + ouvrir PR** sur `arthur-lagenebre/recipe-shelter-documentation`.
5. Pousser le commit UML `1acdf99` après ré-attribution patch (voir notes session_state Claude Code côté Quentin).

### J-7 à J-4 — audit qualité
6. Appliquer P1 du plan de remédiation accessibilité (~1h20, [`_draft_bloc1_audit/04-plan-remediation.md`](../soutenance/frontend/audit/04-plan-remediation.md)) : skip link, Title/Meta, page 404, `role=list`→`ul`, `fr-FR` locale.
7. **Exécuter Lighthouse** (5 URLs × mobile+desktop sur build prod en ligne), remplir le template, capturer scores.
8. Mettre à jour `environment.prod.ts` + `scenario-demo.md` + slides avec URLs réelles.

### J-4 à J-0 — défense
9. Personnaliser les [À COMPLÉTER] des slides Marp.
10. Drill quotidien des questions [`_draft_jury/`](../jury/README.md) (104 Q&A, plan 2 semaines fourni en [`07-mode-entrainement.md`](../jury/07-mode-entrainement.md)).
11. Simulation à blanc 20 min en conditions réelles.

## Discours de défense pour les gaps non comblables

| Gap | Pitch jury |
|-----|-----------|
| Express utilisé alors que cahier dit « from scratch » | « Express est ma couche HTTP, pas un framework. Pas d'ORM, pas de DI, pas de scaffolding — toute l'architecture est câblée manuellement dans `app.ts`. Détails dans l'ADR-001 et la narration from-scratch. » |
| BrowserStack absent | « J'ai testé sur DevTools mobile/desktop (3 breakpoints) et appareils physiques iPhone/iPad — 18 captures versionnées dans `documentation/soutenance/frontend/Responsive/`. BrowserStack reste une amélioration identifiée. » |
| Pas de tests d'intégration auto | « Tests unitaires Node:test (29) sur DTOs/services/middlewares, tests d'intégration via 12 collections Postman versionnées + 11 scripts `.http` reproductibles. CI Postman = évolution identifiée. » |
| Pas de revues par pairs | « Projet solo, mais auto-revue formalisée via 10 ADRs + suite `eslint`/`stylelint`/`husky lint-staged` qui force la qualité à chaque commit. » |
| RGPD : pas de `DELETE /users/me` | « Le ban admin agit comme une suspension d'effet immédiat (re-check en base à chaque requête), la suppression définitive est une opération admin manuelle — évolution identifiée pour exposer un endpoint self-service. » |
| Démo en ligne (si plan B tunnel actif) | « Démo accessible via tunnel cloudflared pour la séance, pipeline de déploiement Railway pré-validé en parallèle. Trade-off conscient sur la persistance du domaine. » |

---

**Mantra de défense** (`_draft_deployment/03-fallback-et-defense.md`) :
> « Mieux vaut un lien moche qui marche qu'un beau domaine vide. »
