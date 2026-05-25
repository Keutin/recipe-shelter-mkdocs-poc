# 1. Découvrir Recipe Shelter (sans compte)

Recipe Shelter est ouvert à la consultation : on peut explorer les recettes, en lire le détail et utiliser la recherche **sans créer de compte**. Certaines actions (mettre en favori, commenter, publier sa propre recette) demandent ensuite une inscription, traitée au chapitre [2. Compte utilisateur](02-compte-utilisateur.md).

## 1.1 Page d'accueil

> Capture : `01-home.png`

À l'ouverture de `https://recipe-shelter.fr` (ou de l'application en local), la page d'accueil affiche :

- **L'en-tête** avec le logo Recipe Shelter, la barre de navigation et les liens `Connexion` / `Inscription` (à droite). Une fois connecté, ces deux liens sont remplacés par un menu compte.
- **Une zone de mise en avant** : recettes récemment publiées et recettes les mieux notées.
- **Le pied de page** avec les liens vers les pages légales (mentions, confidentialité, conditions) et le formulaire de contact.

**Ce qu'on peut faire depuis l'accueil sans être connecté :**

- Cliquer sur une vignette de recette → ouverture de la fiche détaillée.
- Cliquer sur `Rechercher` dans le menu → ouverture du moteur de recherche.
- Cliquer sur `Inscription` ou `Connexion` → création ou ouverture de compte.

## 1.2 Parcourir la liste des recettes

> Capture : `02-recipes-list.png`

L'URL `/recipes` affiche **toutes les recettes publiées**, avec pagination. Chaque carte de recette montre :

- Le titre,
- Une miniature,
- Le pseudo de l'auteur (cliquable pour voir son profil public),
- Les tags principaux,
- Le temps total et la note moyenne.

La pagination est en bas de la liste : un nombre de pages et des flèches `Précédent` / `Suivant`. Le numéro de page courant est indiqué dans l'URL pour pouvoir partager un lien direct vers une page précise.

## 1.3 Rechercher une recette

> Capture suggérée : page `/search`

Depuis le menu, cliquer sur `Rechercher` ouvre la page de recherche. Les filtres disponibles :

| Filtre | Comment l'utiliser | Exemple |
| --- | --- | --- |
| Texte libre | Tape un mot du titre ou de l'ingrédient principal. | `tarte` |
| Catégorie | Sélectionner dans la liste déroulante. | `Dessert` |
| Tags | Cocher un ou plusieurs tags. | `végétarien`, `rapide` |
| Ingrédients | Saisir un ou plusieurs ingrédients. | `poireau`, `crème` |
| Temps total maximum | Indiquer une durée en minutes. | `45` |

Les filtres se combinent. À chaque modification, l'URL est mise à jour avec les paramètres choisis (par exemple `/search?text=tarte&category=Dessert`). Cela permet de **partager une recherche** : il suffit d'envoyer l'URL.

**Si la recherche ne renvoie rien**, retirer un filtre à la fois ou tester un mot plus simple.

## 1.4 Lire une recette

> Capture : `03-recipe-detail.png`

La fiche d'une recette montre :

- Le titre et une image,
- Les **métadonnées** : auteur, date de publication, temps total, difficulté, portions, catégorie, tags,
- La **liste des ingrédients** (quantités exactes),
- Les **étapes de préparation** (numérotées),
- Les **commentaires** des autres membres (chapitre [3. Utiliser les recettes](03-utiliser-recettes.md)).

Boutons en haut de fiche (visibles selon connexion) :

- `Ajouter aux favoris` (icône cœur, demande la connexion).
- `Commenter` (descend jusqu'à la zone commentaires, demande la connexion).

## 1.5 Voir le profil d'un membre

> Capture : `06-profile-user.png`

Depuis n'importe quelle carte de recette ou commentaire, le pseudo de l'auteur est cliquable et ouvre `/users/<pseudo>`. La page publique d'un membre montre :

- Son pseudo et sa photo,
- Sa biographie (facultative),
- Ses recettes publiées,
- Sa date d'inscription.

Aucune donnée personnelle sensible (email, vrai nom) n'apparaît sur ce profil public.

## 1.6 Pages légales et formulaire de contact

Liens dans le pied de page :

- **Mentions légales** : éditeur, hébergeur, contact.
- **Politique de confidentialité** : données collectées, durée de conservation, droits RGPD.
- **Conditions d'utilisation** : règles de publication, comportement attendu, suspension de compte.
- **Contact** : formulaire libre pour écrire à l'équipe (motif + message).

Le formulaire de contact ne demande pas d'être connecté. Il envoie un email vers l'administrateur du site et confirme l'envoi par un message à l'écran.
