---
title: Accueil
description: Documentation Recipe Shelter — Bloc 1 / Bloc 2 / Bloc 3 RNCP
---

# Recipe Shelter — Documentation

Site de documentation complet du projet **Recipe Shelter** (certification
RNCP — 3 blocs : front HTML/CSS/JS, back from-scratch, front Angular).

---

## Alignement avec les blocs de certification

| Bloc | Livrable demandé | Où c'est dans le site |
| --- | --- | --- |
| **Bloc 1** — Front HTML/CSS/JS | Audit qualité (Lighthouse + RGAA), responsive | [Audit Lighthouse + RGAA](soutenance/frontend/audit/README.md) · [Captures responsive](soutenance/frontend/Responsive/README.md) |
| **Bloc 1** — Front HTML/CSS/JS | Plan de remédiation accessibilité | [04 — Plan de remédiation](soutenance/frontend/audit/04-plan-remediation.md) |
| **Bloc 2** — Back from-scratch | Architecture (couches, câblage manuel) | [Architecture backend](soutenance/backend/architecture.md) |
| **Bloc 2** — Back from-scratch | Sécurité (auth, OWASP, RGPD) | [Sécurité backend](soutenance/backend/securite.md) |
| **Bloc 2** — Back from-scratch | UML (classes, séquences, cas d'usage) | [Diagrammes UML](soutenance/backend/uml/README.md) |
| **Bloc 2** — Back from-scratch | Diagrammes techniques (C4, pipeline, séquences) | [Diagrammes](soutenance/backend/diagrams/g1-architecture-c4.md) |
| **Bloc 2** — Back from-scratch | Schéma SQL + dictionnaire de données | [Base de données](soutenance/backend/database/README.md) |
| **Bloc 2** — Back from-scratch | API (CRUD, auth, rôles) | [OpenAPI Redoc](soutenance/backend/openapi/README.md) · [Postman](soutenance/backend/postman/README.md) · [Catalogue d'erreurs](soutenance/backend/errors.md) |
| **Bloc 2** — Back from-scratch | Décisions techniques justifiées | [ADRs](soutenance/backend/adr/README.md) · [Narration from-scratch](soutenance/backend/adr/00-narration-from-scratch.md) |
| **Bloc 3** — Framework (Angular) | Architecture frontend + choix tech | [Architecture frontend](soutenance/frontend/architecture.md) |
| **Bloc 3** — Framework (Angular) | Guide utilisateur | [Manuel utilisateur](manuel-utilisateur/README.md) |
| **Transverse** | Démo en ligne | [Plan de déploiement Railway](deployment/README.md) · [Scénario démo](soutenance/demo/scenario-demo.md) |
| **Transverse** | Soutenance orale (slides ~20 min) | [Slides Marp](slides/README.md) |
| **Transverse** | Préparation Q&A jury (~104 questions) | [Banque de questions jury](jury/README.md) |

---

## Sections du site

### Soutenance
- **[Backend](soutenance/backend/README.md)** — architecture, sécurité,
  catalogue d'erreurs (~140 codes), OpenAPI (53 endpoints), diagrammes
  Mermaid (C4, pipeline, séquences), UML, ADRs, collections Postman,
  schéma BDD.
- **[Frontend](soutenance/frontend/architecture.md)** — architecture
  Angular 21 (Signals, standalone, SSR), audit Lighthouse + RGAA, plan
  de remédiation, captures responsive.
- **[Démo](soutenance/demo/scenario-demo.md)** — comptes de test +
  scénario de démonstration jury.

### Pour l'utilisateur final
- **[Manuel utilisateur](manuel-utilisateur/README.md)** — guide visiteur,
  membre, admin · FAQ · incidents.

### Mise en production
- **[Déploiement](deployment/README.md)** — options comparées
  (Railway / Render / Fly / OVH), plan Railway pas-à-pas, fallback +
  défense jury.

### Préparation soutenance
- **[Slides](slides/README.md)** — deck Marp ~20 min, 28 slides.
- **[Banque de questions jury](jury/README.md)** — ~104 questions
  (transversales, Bloc 1/2/3, sécurité, pièges, améliorations) + plan
  d'entraînement.

### Méta
- **[Plan MkDocs](meta/mkdocs/README.md)** — architecture du site, POC,
  walkthrough handoff.
- **[Contextes LLM](meta/project-context/backend-context.md)** — règles
  pour assistants IA sur les repos backend & frontend.
- **[Revue de conformité](meta/review/synthesis.md)** — matrice cahier des
  charges ⇄ état du projet, top 5 risques, plan d'action pré-soutenance.

---

## Conventions

- **Branches & commits** : voir [Convention Git](GIT_CONVENTION.md).
- **Base URL backend** : `http://localhost:3000` en local, préfixe
  `/api/v1` pour les routes documentées.
- **Diagrammes** : Mermaid par défaut (rendu natif), PlantUML en source
  alternative pour l'UML strict (voir
  [03 — Cas d'utilisation](soutenance/backend/uml/03-diagramme-cas-utilisation.md)).
