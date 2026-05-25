# Recipe Shelter — Documentation

Site de documentation complet du projet **Recipe Shelter** (certification RNCP — 3 blocs : front HTML/CSS/JS, back from-scratch, front Angular).

## Sections

### Soutenance
- **[Backend](soutenance/backend/)** — architecture, sécurité, catalogue d'erreurs, OpenAPI (53 endpoints), diagrammes (C4, pipeline, séquences), ADRs, UML, collections Postman, schéma BDD
- **[Frontend](soutenance/frontend/)** — captures responsive (desktop/iPad/iPhone), audit Lighthouse + RGAA, plan de remédiation
- **[Demo](soutenance/demo/)** — comptes de test + scénario de démonstration

### Manuel utilisateur
- **[Manuel utilisateur](manuel-utilisateur/)** — guide visiteur / membre / admin, FAQ, incidents

### Mise en production
- **[Déploiement](deployment/)** — options comparées (Railway/Render/Fly/OVH), plan Railway pas-à-pas, fallback + défense jury

### Préparation soutenance
- **[Slides](slides/)** — deck Marp ~20 min, 28 slides
- **[Banque de questions jury](jury/)** — ~104 questions (transversales, Bloc 1/2/3, sécurité, pièges, améliorations) + plan d'entraînement

### Méta
- **[Project context (LLM)](meta/project-context/)** — contextes backend/frontend pour assistants IA
- **[Plan MkDocs](meta/mkdocs/)** — architecture du site, POC, walkthrough handoff

## Conventions

- **Branches & commits** : voir [GIT_CONVENTION](GIT_CONVENTION.md).
- **Base URL backend** : `http://localhost:3000` en local, préfixe `/api/v1` pour les routes documentées.
