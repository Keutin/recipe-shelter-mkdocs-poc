# 4. Administration

Ce chapitre concerne les **comptes administrateur**. L'accès aux pages décrites ci-dessous est protégé : un membre standard qui tente d'ouvrir une URL `/admin/...` est redirigé vers la page de connexion (ou voit une page « Accès refusé »).

> Capture : `07-admin-dashboard.png`

## 4.1 Se connecter en tant qu'administrateur

L'identification est la même que pour un compte standard, via la page `/sign-in`. Le caractère « admin » est attaché au compte lui-même : il n'y a **pas de page de connexion séparée**. Une fois connecté, le menu compte montre une entrée supplémentaire `Administration`.

En soutenance, le compte de test admin est documenté dans `documentation/soutenance/demo/comptes-test.md`.

## 4.2 Tableau de bord

URL `/admin/dashboard`. Le tableau de bord regroupe :

| Bloc | Donnée affichée |
| --- | --- |
| État serveur | Statut, uptime, version backend. |
| Recettes en attente | Nombre de recettes à modérer, lien vers `/admin/recipes`. |
| Commentaires modérés | Nombre de commentaires masqués, lien vers `/admin/comments/moderated`. |
| Commentaires supprimés | Nombre de commentaires soft-deleted, lien vers `/admin/comments/soft-deleted`. |
| Utilisateurs suspendus | Nombre de comptes bannis, lien vers la gestion utilisateurs. |

Tous les compteurs sont cliquables et ouvrent la file correspondante.

## 4.3 Modérer une recette

> Capture : `08-admin-moderation.png` (liste) et `09-admin-review-recipe.png` (fiche)

### Lister les recettes à modérer

URL `/admin/recipes`. La liste montre toutes les recettes au statut `En attente`, triées par date de soumission (la plus ancienne en haut, pour ne pas faire trop attendre l'auteur).

Pour chaque ligne : titre, auteur, date de soumission, bouton `Examiner`.

### Examiner une recette

URL `/admin/recipes/:id`. La page affiche :

- La fiche **telle qu'elle apparaîtra publiquement** (ingrédients, étapes, image, métadonnées),
- Une **zone de décision** en bas, avec quatre actions :

| Action | Effet | Confirmation demandée |
| --- | --- | --- |
| `Approuver` | La recette passe à `Publié`, devient visible publiquement. | Non |
| `Refuser` | La recette passe à `Refusé`. Un champ `Motif` est obligatoire, transmis à l'auteur. | Oui (saisie du motif) |
| `Archiver` | La recette passe à `Archivé`. À utiliser pour les cas limites (ni clairement valide, ni clairement à refuser). | Oui |
| `Supprimer` | La recette est définitivement effacée. À réserver aux contenus inappropriés (spam, contenu illégal). | Oui, double confirmation |

Chaque action est **journalisée** côté serveur (qui a fait quoi, quand, sur quelle recette). Cela répond aux exigences de traçabilité de la modération.

## 4.4 Modérer un commentaire

### Files disponibles

| URL | Contenu |
| --- | --- |
| `/admin/comments/moderated` | Commentaires actuellement masqués au public mais conservés en base. |
| `/admin/comments/soft-deleted` | Commentaires supprimés (soft-delete) par leur auteur ou par un administrateur. Conservés pour traçabilité. |

### Actions par commentaire

Sur chaque ligne, le menu d'actions propose :

- **Masquer** (depuis la liste des commentaires publics) → bascule en `moderated`.
- **Annuler la modération** → re-publie le commentaire.
- **Restaurer** (depuis `soft-deleted`) → restaure le commentaire visible.
- **Supprimer définitivement** → action irréversible, demande confirmation.

Les actions sont également **journalisées**.

## 4.5 Gérer les utilisateurs

URL `/admin/users` (le libellé exact peut varier dans le menu admin).

La liste des comptes affiche :

- Pseudo,
- Email,
- Statut (`actif`, `non activé`, `suspendu`),
- Date d'inscription,
- Nombre de recettes publiées et de commentaires.

### Actions sur un utilisateur

| Action | Effet |
| --- | --- |
| `Suspendre` | Le compte ne peut plus se connecter. Ses recettes restent en ligne. À utiliser pour comportement inapproprié (spam, harcèlement). |
| `Réactiver` | Lève la suspension. |
| `Supprimer` | Efface le compte. Les recettes publiées sont anonymisées (auteur = « Utilisateur supprimé ») pour ne pas casser la communauté. À utiliser exceptionnellement, sur demande RGPD ou cas grave. |

> Conseil d'usage : préférer la **suspension** à la **suppression**. La suspension est réversible et laisse la trace si l'utilisateur revient pour s'expliquer.

## 4.6 Bonnes pratiques de modération

1. **Délai de réponse** : viser une modération sous 48 h pour les recettes soumises. Au-delà, l'expérience auteur se dégrade.
2. **Motifs de refus explicites** : un refus avec un motif clair (« étapes 3 et 4 contradictoires, merci de préciser ») est plus utile qu'un motif vague (« incomplète »).
3. **Privilégier l'archivage au refus** quand on doute : l'archivage est neutre, le refus est plus négatif côté auteur.
4. **Documenter les cas litigieux** : si on n'est pas sûr, mettre la recette en archivage et noter le cas dans un canal admin pour en discuter.
5. **Ne jamais supprimer sans trace** : utiliser les actions journalisées de l'application plutôt qu'une intervention manuelle en base de données.

## 4.7 Limites volontaires

L'application **ne propose pas** :

- De promotion d'utilisateur en administrateur depuis l'interface (passage par modification directe en base de données, intentionnel pour limiter les escalades de privilèges accidentelles),
- D'export massif de données utilisateurs depuis l'interface (les exports RGPD passent par une demande explicite),
- De connexion par réseaux sociaux (choix de simplicité et de souveraineté des données).
