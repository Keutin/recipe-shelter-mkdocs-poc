# Guide utilisateur — Recipe Shelter

Ce dossier contient un **manuel d'utilisation** de Recipe Shelter, rédigé du point de vue de l'utilisateur final (visiteur, membre, administrateur). Il est conçu comme livrable de certification pour le Bloc 3 et complète le scénario de démonstration (`documentation/soutenance/demo/scenario-demo.md`) qui, lui, vise le jury.

## À quoi sert ce guide

- **Livrable de soutenance** : montrer au jury que l'application a été pensée pour ses utilisateurs, pas seulement pour les développeurs.
- **Support de démo** : aide-mémoire si on doit reprendre la main pendant la présentation.
- **Documentation utilisateur** : si l'application va plus loin que la certification, c'est la base d'un manuel public.

## Pourquoi on rédige côté utilisateur, pas côté développeur

La certification valide aussi la **capacité à se mettre à la place de l'utilisateur final**. Une doc qui parle seulement de routes Angular et d'API REST passe à côté. Ici on parle :

- de ce que l'utilisateur veut faire (« trouver une recette aux poireaux »),
- de ce qu'il voit à l'écran (le bouton, le message, le statut),
- de ce qui se passe quand quelque chose échoue (compte non activé, recette refusée).

Le vocabulaire technique reste accessible (URL, formulaire, statut) mais on évite les noms internes (route, guard, repository, etc.).

## Structure

| Fichier | Pour qui | Contenu |
| --- | --- | --- |
| [01-decouverte.md](01-decouverte.md) | Visiteur (pas de compte) | Page d'accueil, exploration libre, recherche, lecture d'une recette, pages légales. |
| [02-compte-utilisateur.md](02-compte-utilisateur.md) | Membre | Inscription, activation email, connexion, mot de passe oublié, profil personnel, déconnexion. |
| [03-utiliser-recettes.md](03-utiliser-recettes.md) | Membre connecté | Favoris, commentaires, création de recette, brouillon, soumission à la modération, suivi du statut. |
| [04-administration.md](04-administration.md) | Administrateur | Tableau de bord, modération des recettes soumises, modération des commentaires, gestion des comptes. |
| [05-faq-incidents.md](05-faq-incidents.md) | Tous | Questions fréquentes, messages d'erreur courants, contact. |

## Walkthrough pour Arthur

### Étape 1 — Relire (15 min)

Parcours rapide en se mettant dans la peau de quelqu'un qui n'a jamais vu le site. Repérer :

- Les **captures manquantes** (chaque section référence une image dans `documentation/soutenance/frontend/Screenshots/`). Ce qui existe déjà : `01-home.png`, `02-recipes-list.png`, `03-recipe-detail.png`, `04-comments.png`, `05-favorites.png`, `06-profile-me.png`, `06-profile-user.png`, `07-admin-dashboard.png`, `08-admin-moderation.png`, `09-admin-review-recipe.png`, `10-login.png`, `11-recipe-create.png`. Si une référence pointe vers une capture absente, soit on la prend, soit on retire la mention.
- Les **différences avec le réel** : libellés de boutons, textes d'erreur, URLs. Le guide a été écrit en croisant les routes Angular et le scénario de démo, mais certains messages exacts peuvent diverger. Corriger au passage.

### Étape 2 — Compléter les captures manquantes (30 min)

Pour chaque section, idéalement une capture par page principale. Format recommandé : PNG, largeur 1280–1920px, navigateur en mode normal (pas devtools ouverts). Nommer les fichiers en continuité avec ceux existants (`12-...`, `13-...`).

### Étape 3 — Copier dans le repo `documentation`

Cible : créer `documentation/manuel-utilisateur/` à côté de `documentation/soutenance/`. Cette séparation matérialise la différence d'audience (jury vs utilisateur final). Commandes (à exécuter par Arthur, dans `C:\DEV\ARTHUR\RECETTES\documentation`) :

```bash
mkdir manuel-utilisateur
copy ..\_draft_user_guide\*.md manuel-utilisateur\
git checkout -b docs/manuel-utilisateur
git add manuel-utilisateur
git commit -m "docs: ajoute le manuel utilisateur (Bloc 3)"
```

### Étape 4 — Mentionner le guide depuis le `README.md` racine de `documentation`

Une ligne du type :

```markdown
- [Manuel utilisateur](manuel-utilisateur/README.md) : guide d'utilisation côté visiteur, membre et administrateur.
```

dans la section `Sommaire` ou équivalente.

### Étape 5 — S'en servir en soutenance

Le manuel n'est probablement pas projeté tel quel pendant la soutenance — il sert :

- **À l'oral** : pour répondre à « comment un utilisateur ferait pour faire X ? » sans improviser.
- **En slide unique** : éventuellement glisser une mention « Manuel utilisateur fourni dans la documentation » dans le deck `_draft_slides/`.
- **Dans le dossier remis au jury** : imprimable, lisible sans accès au site.

## Principes éditoriaux qu'on a suivis

1. **Parler à l'utilisateur**, pas de lui (« vous cliquez sur… » et non « l'utilisateur clique sur… »).
2. **Décrire l'écran avant l'action** : où regarder, puis quoi faire.
3. **Anticiper les blocages** : pour chaque action, mentionner ce qui peut échouer et comment s'en sortir.
4. **Pas d'argot technique** : « page de connexion » plutôt que `/sign-in`, sauf si on indique explicitement l'URL.
5. **Pas de promesse de fonctionnalité future** : on documente ce qui existe.

## Lien avec les autres livrables

- [Scénario de démo](../soutenance/demo/scenario-demo.md) → fil rouge pour le jury (8–10 min de manipulation).
- [Comptes de test](../soutenance/demo/comptes-test.md) si présents → identifiants à utiliser pour tester chaque rôle.
- [Banque de questions jury](../jury/README.md) → certains items de FAQ peuvent recouper des questions Bloc 3 sur l'expérience utilisateur.
- [Slides soutenance](../slides/soutenance-slides.md) → Slide « Démo » peut renvoyer au manuel pour la version écrite.
