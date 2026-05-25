# 2. Compte utilisateur

Recipe Shelter demande un compte pour **mettre une recette en favori, commenter une recette, en publier une**. Le compte est gratuit et utilise une simple adresse email.

## 2.1 Créer un compte

> Capture suggérée : page `/sign-up`

Depuis le menu, cliquer sur `Inscription`. Le formulaire demande :

| Champ | Règles |
| --- | --- |
| Adresse email | Format `xxx@yyy.zz` valide. Doit être unique. |
| Pseudo | Visible publiquement. 3 à 30 caractères. |
| Mot de passe | Minimum 8 caractères, au moins une majuscule, une minuscule, un chiffre et un caractère spécial. |
| Confirmation mot de passe | Doit être identique au précédent. |
| Acceptation des conditions | Case obligatoire. |

À la validation :

- **Si tout est correct** : le compte est créé, un email de vérification est envoyé à l'adresse fournie. Un message à l'écran rappelle de vérifier la boîte mail (et les indésirables).
- **Si une règle n'est pas respectée** : le champ concerné est entouré et un message indique quoi corriger. Le formulaire reste rempli.

## 2.2 Activer son compte

Le mail d'activation contient un **lien unique**, valide un temps limité. Cliquer dessus :

- Si le lien est encore valide → le compte est marqué actif, on est redirigé vers la page de connexion.
- Si le lien a expiré → un message propose de **renvoyer un email d'activation** depuis la page de connexion.

Tant que le compte n'est pas activé, la **connexion est refusée** avec un message explicite : « Votre compte n'est pas encore activé. Vérifiez votre boîte mail. »

## 2.3 Se connecter

> Capture : `10-login.png`

Depuis le menu, cliquer sur `Connexion`. Le formulaire demande email et mot de passe.

Cas de figure :

| Situation | Comportement |
| --- | --- |
| Identifiants corrects, compte actif | Connexion validée. Le menu compte remplace les liens `Connexion` / `Inscription`. |
| Identifiants incorrects | Message générique « Identifiants invalides. » (volontairement vague pour ne pas indiquer si l'email existe). |
| Compte non activé | Message « Votre compte n'est pas encore activé. » avec lien pour renvoyer le mail. |
| Compte suspendu | Message « Votre compte a été suspendu. » sans accès possible. Contact admin via le formulaire. |

Après connexion, vous êtes redirigé vers l'accueil — ou vers la page que vous tentiez d'atteindre si vous aviez cliqué sur un lien protégé (`/profile`, `/me/recipes/submit`, etc.).

## 2.4 Mot de passe oublié

Depuis la page de connexion, lien `Mot de passe oublié ?`. Le formulaire demande l'adresse email.

- **Si l'email existe** : un mail contenant un lien de réinitialisation est envoyé. Le lien expire après un temps limité.
- **Si l'email n'existe pas** : un **message identique** s'affiche (volontaire, pour ne pas révéler quels emails sont enregistrés).

Le lien de réinitialisation ouvre un formulaire pour saisir et confirmer un nouveau mot de passe (mêmes règles qu'à l'inscription).

## 2.5 Profil personnel

> Capture : `06-profile-me.png`

Une fois connecté, le menu compte donne accès à `Mon profil`. La page affiche :

- **Vos informations personnelles** (modifiables) : pseudo, biographie, photo de profil.
- **Vos préférences** : email de notification, langue (si applicable).
- **Vos statistiques** : nombre de recettes publiées, brouillons en cours, favoris.
- **Le lien vers votre profil public** : ce que les autres membres voient.

Les modifications sont enregistrées par le bouton `Mettre à jour`. Une confirmation s'affiche brièvement après chaque enregistrement.

## 2.6 Changer son mot de passe

Depuis le profil, section `Sécurité`. Il faut saisir :

- Le mot de passe actuel,
- Le nouveau mot de passe,
- La confirmation du nouveau.

Si le mot de passe actuel est incorrect, le formulaire l'indique sans rien modifier.

## 2.7 Supprimer son compte (RGPD)

> Capture suggérée : section `Suppression de compte` du profil

Le profil propose une suppression de compte. Conformément au RGPD :

- L'action demande une **confirmation** (saisie du mot de passe ou case « Je confirme la suppression »).
- À la suppression : les données personnelles (email, mot de passe, informations de profil) sont effacées. Les recettes publiées peuvent être **conservées de façon anonymisée** (auteur remplacé par « Utilisateur supprimé ») si elles ont reçu des commentaires ou des favoris d'autres membres, pour ne pas casser l'historique communautaire.
- Une fois supprimé, le compte ne peut plus être restauré. L'adresse email peut être réutilisée pour une nouvelle inscription.

## 2.8 Se déconnecter

Depuis le menu compte, cliquer sur `Déconnexion`. La session est immédiatement fermée :

- Le menu redevient `Connexion` / `Inscription`.
- Les pages protégées (profil, création de recette, admin) ne sont plus accessibles : tentative d'y aller → redirection vers la page de connexion.

> Détail technique : la déconnexion supprime le cookie de session côté navigateur et invalide la session côté serveur. Aucune trace ne reste dans le navigateur après cette action.
