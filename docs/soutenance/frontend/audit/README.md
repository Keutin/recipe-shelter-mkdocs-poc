---
title: Audit Lighthouse + RGAA
description: Audit qualité frontend Bloc 1 — Lighthouse et grille RGAA/WCAG
tags:
  - bloc-1
  - frontend
  - audit
  - accessibilite
---

# Bloc 1 — Audit Lighthouse + grille RGAA/WCAG

Ce dossier contient le **livrable de qualité frontend** attendu pour le Bloc 1 de la certification : un audit Lighthouse (performance, accessibilité, bonnes pratiques, SEO) et une grille RGAA/WCAG avec statut critère par critère.

L'audit RGAA a été **pré-rempli par lecture statique du code** (Angular templates + CSS + config). Lighthouse, lui, demande de lancer l'application — Arthur doit faire tourner les mesures lui-même et reporter les chiffres dans le template fourni.

## Pourquoi ce livrable

- Le Bloc 1 (HTML/CSS/JS et accessibilité) doit montrer que l'application respecte les standards d'accessibilité (RGAA 4.x / WCAG 2.1 AA) et les bonnes pratiques de qualité web (Lighthouse).
- Le jury attend des **chiffres** (scores Lighthouse) **et** une **analyse critique** (où on est conforme, où on ne l'est pas, et pourquoi).
- Une grille rigoureuse vaut mieux qu'un Lighthouse à 100 partout sans recul : la honnêteté impressionne, l'auto-satisfaction agace.

## Structure

| Fichier | Contenu |
| --- | --- |
| [01-lighthouse-template.md](01-lighthouse-template.md) | Template à remplir : 5 URLs cibles, 4 catégories, commandes exactes (CLI ou DevTools), captures à prendre. |
| [02-rgaa-grille.md](02-rgaa-grille.md) | Grille RGAA 4.1 par thématique (13 thématiques), pré-remplie avec les findings issus de l'audit statique. Chaque critère a un statut : ✅ conforme / ⚠️ partiellement / ❌ non conforme / N/A. |
| [03-findings-detailles.md](03-findings-detailles.md) | Inventaire complet des findings (≈ 120) avec `file:line`, ce qui est bon, ce qui manque, et la correction concrète. C'est la base technique des deux fichiers précédents. |
| [04-plan-remediation.md](04-plan-remediation.md) | Plan d'action priorisé : 10 corrections à fort impact, avec effort estimé et fichier exact à modifier. Pour montrer au jury qu'on sait où aller ensuite. |
| [05-defense-jury.md](05-defense-jury.md) | Notes pour la soutenance : comment parler du score, comment justifier les écarts, questions probables et leurs réponses. |

## Walkthrough pour Arthur

### Étape 1 — Relire les findings (30 min)

Ouvrir [03-findings-detailles.md](03-findings-detailles.md). Pour chaque thématique, vérifier en ouvrant les fichiers pointés que les findings correspondent à la réalité. Quelques-uns peuvent être faux positifs (audit statique → pas d'exécution réelle). Corriger la grille si besoin.

### Étape 2 — Faire tourner Lighthouse (1 h)

Suivre [01-lighthouse-template.md](01-lighthouse-template.md). Procédure résumée :

1. Build de production : `cd frontend && npm run build`
2. Servir : `npm run serve:ssr:recipe-shelter` (ou la commande SSR du `package.json`).
3. Pour chaque URL cible (home, /recipes, /recipes/:slug, /sign-in, /admin/dashboard) :
   - Ouvrir Chrome en navigation privée (extensions désactivées).
   - DevTools → Lighthouse → Mobile → toutes catégories → Analyser.
   - Remplir la grille de scores.
   - Faire une capture du rapport (PNG ou PDF).

Si possible, lancer aussi un audit **Desktop** : les scores sont souvent plus flatteurs.

### Étape 3 — Compléter la grille RGAA (30 min)

[02-rgaa-grille.md](02-rgaa-grille.md) est déjà rempli sur la majorité des critères techniques. Restent à valider manuellement :

- **Contraste** : utiliser l'extension Chrome `axe DevTools` ou `WAVE` sur chaque page cible. Reporter les contrastes < 4.5:1.
- **Navigation clavier** : tester chaque page en Tab/Shift+Tab/Entrée/Échap. Confirmer ce que dit la grille.
- **Lecteur d'écran** (optionnel mais valorisé) : faire une passe rapide avec NVDA (Windows, gratuit) ou Voice Over (Mac). 5 min sur la home + une fiche recette suffisent à étoffer le discours.

### Étape 4 — Décider du périmètre de remédiation

Lire [04-plan-remediation.md](04-plan-remediation.md). Si l'épreuve est proche, **ne pas chercher à tout corriger**. Choisir 3 à 5 corrections symboliques (les plus visibles : skip link, fr-FR locale, 404 page, alt text, title/meta SEO) — c'est l'amélioration entre l'avant et l'après qui parle au jury, pas la perfection.

Documenter les corrections faites dans un commit `feat: améliorations accessibilité et SEO Bloc 1` (ou plusieurs petits commits sémantiques).

### Étape 5 — Préparer le discours

Lire [05-defense-jury.md](05-defense-jury.md). Les questions Bloc 1 de la banque jury (`_draft_jury/01-bloc1-frontend.md`) recoupent en partie ce livrable — l'audit fournit les **ancres factuelles** (« on a 92 en accessibilité parce que… »).

### Étape 6 — Copier dans le repo `documentation`

Cible : créer `documentation/soutenance/frontend/audit/`. Commandes (à exécuter par Arthur dans `C:\DEV\ARTHUR\RECETTES\documentation`) :

```bash
mkdir soutenance\frontend\audit
copy ..\_draft_bloc1_audit\*.md soutenance\frontend\audit\
git checkout -b docs/bloc1-audit
git add soutenance/frontend/audit
git commit -m "docs: ajoute audit Lighthouse et grille RGAA (Bloc 1)"
```

Ne PAS copier `03-findings-detailles.md` si on préfère garder l'inventaire technique privé — il révèle aussi les défauts par fichier:ligne. Le garder dans `_draft_bloc1_audit/` comme note de travail.

## Périmètre de l'audit statique

Ce qu'on a pu analyser **sans lancer l'app** :

- Structure HTML sémantique (landmarks, headings, listes)
- Attributs `alt`, `lang`, `aria-*`, `role`
- Liaison label/input dans les formulaires
- Présence de `outline:none`, focus visible, skip links
- Configuration Angular (lazy routes, SSR, Title/Meta, OnPush, `@defer`, NgOptimizedImage)
- Tokens CSS et calcul approximatif de contraste
- Robots.txt, sitemap, OpenGraph
- Gestion d'erreurs et états vides

Ce qui demande **l'application en marche** :

- Scores Lighthouse réels (Performance dépend du temps de réponse serveur)
- Contraste exact des images de fond (hero) au runtime
- Performance hors localhost (Core Web Vitals en conditions réelles)
- Comportement focus après navigation routée (visible à l'œil)
- Lecteur d'écran (NVDA/VoiceOver)

## Lien avec les autres livrables

- [Banque de questions jury, Bloc 1](../../../jury/01-bloc1-frontend.md) — questions sur HTML/CSS/JS/responsive/a11y/Lighthouse.
- [ADRs frontend potentielles](../../backend/adr/README.md) — ADR-006 (Signals + standalone), ADR-007 (SSR), ADR-010 (Bootstrap 5).
- [Slides soutenance, slide Bloc 1](../../../slides/soutenance-slides.md) — les scores Lighthouse ont une zone `[À COMPLÉTER]`.
- [Manuel utilisateur](../../../manuel-utilisateur/README.md) — recoupe accessibilité côté discours utilisateur.

## Référentiel utilisé

- **RGAA 4.1.2** (Référentiel Général d'Amélioration de l'Accessibilité, version 2023) — version officielle française.
- **WCAG 2.1 niveau AA** — base internationale, ce que Lighthouse mesure aussi.
- **Lighthouse v11+** — score / 100 sur 4 catégories.

Sources officielles : `https://accessibilite.numerique.gouv.fr/methode/criteres-et-tests/`, `https://www.w3.org/TR/WCAG21/`.
