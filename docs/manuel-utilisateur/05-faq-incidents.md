# 5. FAQ et messages d'erreur courants

Cette section regroupe les questions fréquentes et les messages d'erreur visibles à l'écran, avec leur signification et la marche à suivre. Pour les questions techniques approfondies (architecture, sécurité, RGPD côté implémentation), voir plutôt la documentation jury (`_draft_jury/`) ou le dossier `documentation/`.

## 5.1 Questions sur le compte

### Je n'ai pas reçu l'email d'activation

1. Vérifier le dossier **spam / indésirables**.
2. Vérifier l'orthographe de l'adresse saisie lors de l'inscription.
3. Depuis la page de connexion, cliquer sur le lien `Renvoyer l'email d'activation`.
4. Si toujours rien, contacter l'administrateur via le formulaire de contact.

### Le lien d'activation dit « expiré »

Le lien est volontairement valide un temps limité (typiquement 24 à 48 heures) pour des raisons de sécurité. Depuis la page de connexion, demander un nouvel email d'activation.

### J'ai oublié mon mot de passe

Lien `Mot de passe oublié ?` sur la page de connexion. Saisir son adresse email. Un mail avec un lien de réinitialisation est envoyé si l'adresse est connue. Voir [2.4 Mot de passe oublié](02-compte-utilisateur.md#24-mot-de-passe-oublié).

### Mon compte a été suspendu, pourquoi ?

La suspension est décidée par un administrateur pour comportement contraire aux conditions d'utilisation (spam, contenu inapproprié, harcèlement). Vous pouvez contacter l'administrateur via le formulaire de contact pour demander des explications. La suspension peut être levée par un administrateur.

### Je veux supprimer mon compte

Depuis votre profil, section `Sécurité` ou `Suppression de compte`. La suppression efface vos données personnelles. Les recettes que vous avez publiées peuvent être conservées de façon anonymisée pour ne pas casser la communauté. Voir [2.7 Supprimer son compte](02-compte-utilisateur.md#27-supprimer-son-compte-rgpd).

## 5.2 Questions sur les recettes

### Ma recette a été refusée, je peux faire quoi ?

Le motif de refus est affiché sur la fiche dans `/me/recipes/list`. Modifier la recette en conséquence et la re-soumettre. Le bouton `Modifier` reste actif tant que la recette n'est pas supprimée.

### Combien de temps pour qu'une recette soit validée ?

L'objectif est sous 48 heures, sans garantie. Un administrateur examine la recette manuellement.

### Puis-je publier la recette de quelqu'un d'autre ?

Non, sauf si vous en avez l'autorisation explicite et que vous citez la source dans la description. Les recettes copiées sans crédit peuvent être refusées ou retirées.

### Mes images sont-elles redimensionnées ?

Selon la version de l'application, les images peuvent être stockées telles quelles ou redimensionnées côté serveur. Recommandation : envoyer des fichiers de quelques centaines de Ko à 1–2 Mo, en JPG ou PNG, pour garder un site rapide à charger.

### Puis-je dupliquer une recette pour en faire une variante ?

Non, il n'y a pas de bouton `Dupliquer` en l'état. Il faut recréer une recette depuis le formulaire et ré-écrire les ingrédients et étapes. La fonctionnalité fait partie des améliorations possibles.

## 5.3 Questions sur les favoris et commentaires

### Pourquoi je ne peux pas mettre une recette en favori ?

Il faut être **connecté**. Le bouton cœur ouvre la page de connexion si vous ne l'êtes pas. Une fois revenu sur la recette après connexion, le clic ajoute le favori.

### Mon commentaire a disparu

Deux cas :

1. Un administrateur a **masqué** le commentaire (modération). Vous devriez recevoir une notification ou voir le commentaire signalé.
2. Vous avez vous-même supprimé le commentaire via le menu `…`.

Si vous pensez que la modération est injustifiée, contactez l'administrateur via le formulaire de contact.

### Combien de commentaires puis-je publier ?

Il n'y a pas de quota strict pour un usage normal. Un compte qui poste de façon massive ou répétitive peut être suspendu pour spam.

## 5.4 Messages d'erreur fréquents

| Message | Signification | Que faire |
| --- | --- | --- |
| `Identifiants invalides.` | Email ou mot de passe incorrect (volontairement vague). | Vérifier l'orthographe. Si oubli, utiliser `Mot de passe oublié ?`. |
| `Votre compte n'est pas encore activé.` | Le compte existe mais n'a pas cliqué sur le lien d'activation. | Vérifier la boîte mail, demander un renvoi. |
| `Votre compte a été suspendu.` | Compte banni par un administrateur. | Voir 5.1 « Mon compte a été suspendu ». |
| `Cette adresse email est déjà utilisée.` | Une inscription précédente existe avec cette adresse. | Utiliser `Mot de passe oublié ?` si vous avez perdu l'accès. |
| `Ce pseudo est déjà pris.` | Un autre membre utilise déjà ce pseudo. | En choisir un autre (3–30 caractères). |
| `Le mot de passe ne respecte pas les règles.` | Longueur ou complexité insuffisante. | Voir [2.1 Créer un compte](02-compte-utilisateur.md#21-créer-un-compte). |
| `Session expirée, merci de vous reconnecter.` | La session a dépassé sa durée de vie. | Se reconnecter depuis `/sign-in`. |
| `Vous devez être connecté pour effectuer cette action.` | Action protégée déclenchée sans session. | Se connecter et reprendre l'action. |
| `Accès refusé.` | Tentative d'ouvrir une page admin sans le rôle. | Se connecter avec un compte administrateur. |
| `Brouillon enregistré.` | Confirmation positive : la sauvegarde a fonctionné. | Aucune action à faire, c'est normal. |
| `Une erreur est survenue, veuillez réessayer.` | Erreur générique côté serveur. | Recharger la page. Si récurrent, contacter l'administrateur. |

## 5.5 Performance et navigateur

### Recipe Shelter est lent à s'afficher

Vérifier la connexion internet et la version du navigateur. L'application est optimisée pour les versions récentes de Chrome, Firefox, Safari, Edge.

### L'application ne s'affiche pas du tout

Vérifier que JavaScript est activé dans le navigateur. Sans JavaScript, le site peut afficher une partie du contenu grâce au rendu côté serveur (SSR), mais les interactions (recherche, formulaires, favoris) demandent JavaScript actif.

### Les images ne se chargent pas

Vérifier les bloqueurs de publicité ou extensions de confidentialité qui peuvent filtrer les images. Recharger la page.

## 5.6 Contact

Pour toute question non couverte ici, utiliser le formulaire de contact accessible depuis le pied de page. Indiquer :

- Votre pseudo ou email (si vous êtes membre, sinon une adresse de réponse),
- Une description claire du problème,
- L'URL de la page où le problème survient,
- Le navigateur utilisé (Chrome 123, Firefox 124, etc.) si erreur d'affichage.

Une réponse est envoyée par email sous quelques jours ouvrés.
