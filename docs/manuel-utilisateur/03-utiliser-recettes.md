# 3. Utiliser les recettes (membre connecté)

Ce chapitre suppose que vous avez un compte actif (voir [2. Compte utilisateur](02-compte-utilisateur.md)). Toutes les actions ci-dessous demandent d'être connecté. Si vous tentez de les déclencher sans session active, Recipe Shelter vous redirige vers la page de connexion puis vous ramène à l'endroit prévu.

## 3.1 Mettre une recette en favori

> Capture : `05-favorites.png`

Sur la **fiche d'une recette** ou sur une **carte de la liste**, l'icône cœur permet d'ajouter ou retirer la recette de vos favoris :

- Cœur vide → la recette n'est pas en favori. Cliquer pour l'ajouter.
- Cœur plein → la recette est déjà en favori. Cliquer pour la retirer.

Vos favoris sont consultables depuis le menu compte → `Mes favoris` (URL `/me/favorites`). On y retrouve la liste, avec les mêmes filtres et tris que la recherche principale.

**Note** : un favori reste valide même si l'auteur archive sa recette. Si la recette est supprimée définitivement par un administrateur, elle disparaît de vos favoris automatiquement.

## 3.2 Commenter une recette

> Capture : `04-comments.png`

Sur la fiche d'une recette, en bas, une zone `Commentaires` affiche la conversation. Pour participer :

1. Cliquer dans le champ `Votre commentaire`.
2. Saisir le texte (limite indiquée sous le champ).
3. Cliquer sur `Publier`.

Le commentaire apparaît immédiatement, avec votre pseudo et la date.

**Modération** : un administrateur peut **masquer** un commentaire qui ne respecte pas les conditions d'utilisation (insultes, spam, hors-sujet). Dans ce cas, l'auteur du commentaire reçoit une notification (ou voit son commentaire signalé en rouge dans la fiche). Voir aussi [4. Administration](04-administration.md).

Vous pouvez **supprimer vos propres commentaires** à tout moment via le menu `…` à côté de chaque commentaire.

## 3.3 Créer une recette

> Capture : `11-recipe-create.png`

Depuis le menu compte, cliquer sur `Soumettre une recette` (URL `/me/recipes/submit`). Le formulaire est découpé en sections :

### Informations principales

| Champ | Obligatoire | Détail |
| --- | --- | --- |
| Titre | Oui | Court, descriptif (ex. « Tarte aux poireaux et chèvre »). |
| Description courte | Oui | 1 à 2 phrases, sert d'accroche sur les vignettes. |
| Catégorie | Oui | Liste fermée : Entrée, Plat, Dessert, Boisson, Apéritif, etc. |
| Tags | Non | Cocher dans la liste (`végétarien`, `rapide`, `sans gluten`…). |
| Image principale | Non mais conseillé | Fichier JPG ou PNG, taille raisonnable (quelques centaines de Ko à 1–2 Mo). |

### Métadonnées culinaires

| Champ | Détail |
| --- | --- |
| Portions | Nombre, par défaut 4. |
| Temps de préparation | En minutes. |
| Temps de cuisson | En minutes. |
| Difficulté | Facile / Moyen / Difficile. |

### Ingrédients

Liste dynamique : un bouton `+ Ajouter un ingrédient` ajoute une ligne. Chaque ligne demande :

- Quantité (nombre),
- Unité (g, ml, c. à café, c. à soupe, pièce…),
- Nom de l'ingrédient.

Bouton `Supprimer` à droite de chaque ligne. On peut réordonner les ingrédients en faisant glisser la poignée (si l'interface le propose).

### Étapes

Liste dynamique aussi, chaque étape contient une zone de texte libre. L'ordre se gère avec les flèches `↑` / `↓` ou par glisser.

### Enregistrer

Deux boutons en bas :

- `Enregistrer le brouillon` → la recette est sauvegardée avec le statut **Brouillon**. Elle n'est pas visible publiquement.
- `Soumettre à la modération` → la recette passe au statut **En attente**. Voir section 3.4 ci-dessous.

Un message confirme l'enregistrement (par exemple `Brouillon enregistré.`) et la recette devient accessible depuis `/me/recipes/list`.

**Astuce** : il est possible d'enregistrer en brouillon plusieurs fois en cours de rédaction. Les modifications ne sont pas perdues si on quitte la page (à condition d'avoir cliqué sur `Enregistrer`).

## 3.4 Soumettre une recette à la modération

Une fois la recette complète, retourner sur sa fiche d'édition (`/me/recipes/submit/:id`). La page affiche une **checklist** : tous les champs obligatoires doivent être valides pour que le bouton `Soumettre` devienne actif.

Au clic sur `Soumettre` :

1. La recette passe au statut **En attente**.
2. Elle apparaît dans la liste des recettes à modérer côté administrateur.
3. Vous êtes redirigé vers `/me/recipes/list` avec un message de confirmation.
4. La recette n'est **pas encore visible publiquement** — il faut attendre l'approbation.

## 3.5 Suivre ses recettes

> URL : `/me/recipes/list`

La liste personnelle montre toutes vos recettes, quel que soit leur statut, avec un filtre par statut :

| Statut | Description | Visible publiquement ? |
| --- | --- | --- |
| `Brouillon` | En cours d'édition, non soumis. | Non |
| `En attente` | Soumis, en attente de modération. | Non |
| `Publié` | Approuvé par un administrateur, en ligne. | Oui |
| `Refusé` | Refusé par un administrateur (avec motif). | Non |
| `Archivé` | Retiré par vous ou par un administrateur. | Non |

Actions possibles selon le statut :

- **Brouillon** : modifier, supprimer, soumettre.
- **En attente** : modifier (la soumission est alors annulée et la recette repasse en brouillon), supprimer.
- **Publié** : voir la fiche publique, archiver (la retire du public).
- **Refusé** : lire le motif, modifier, re-soumettre.
- **Archivé** : restaurer (passe en brouillon).

## 3.6 Que faire si une recette est refusée

Quand un administrateur refuse une recette, le motif est affiché sur la fiche dans `/me/recipes/list`. Les motifs typiques :

- Titre vide ou trompeur,
- Étapes incomplètes ou incohérentes,
- Image inadaptée (qualité, droits),
- Contenu sans rapport (test, blague, recette copiée).

Vous pouvez **modifier la recette** et la **re-soumettre** sans recommencer à zéro.

## 3.7 Modifier ou supprimer une recette déjà publiée

- **Modifier une recette publiée** : depuis `/me/recipes/list`, cliquer sur `Modifier`. La recette passe alors en brouillon (elle disparaît temporairement du public) puis il faut la re-soumettre. Cela permet à un administrateur de revalider les changements.
- **Archiver** : retire la recette du public sans la supprimer. Elle reste dans votre liste personnelle au statut `Archivé`. Vous pouvez la restaurer plus tard.
- **Supprimer définitivement** : action irréversible. Les favoris et commentaires associés sont également retirés. Une confirmation est demandée.
