# Projet 5 : Site Web de Recettes de Cuisine Collaboratif

> Cahier des charges officiel de la certification RNCP d'Arthur.
> Conservé hors des repos git pour servir de contexte de travail.

## Introduction et Contexte

Un site web de recettes de cuisine collaboratif permet aux utilisateurs de partager leurs recettes, de commenter celles des autres et de sauvegarder leurs recettes préférées. Ce projet vise à développer les compétences des apprenants en conception et développement web en les défiant de créer un site web où les utilisateurs peuvent interagir autour de la cuisine. Le site doit inclure une recherche avancée par ingrédients, type de cuisine, temps de préparation, etc.

## Sujet Détaillé

Les apprenants sont chargés de créer un site web collaboratif de recettes de cuisine. Le site doit permettre aux utilisateurs de partager leurs recettes, de commenter celles des autres et de sauvegarder leurs recettes préférées. Il doit offrir une expérience utilisateur optimale sur des appareils de différentes tailles (smartphones, tablettes, ordinateurs de bureau) et respecter les principes d'accessibilité web.

---

## Bloc 1 : Front-End

### Outils et Technologies
- **Langages** : HTML5, CSS3, JavaScript
- **Frameworks (optionnel)** : Bootstrap ou autre framework CSS pour le design responsif, ARIA roles pour l'accessibilité
- **Outils de Test** : Google Lighthouse pour tester la responsivité et l'accessibilité, BrowserStack pour tester la compatibilité entre navigateurs

### Principales Fonctionnalités Attendues
- **Affichage des recettes** : Création de templates pour présenter les recettes avec des images, titres, descriptions, ingrédients et étapes.
- **Formulaire de soumission de recettes** : Intégration de formulaires permettant aux utilisateurs d'ajouter et de modifier des recettes.
- **Recherche avancée** : Mise en place d'une fonctionnalité de recherche avancée par ingrédients, type de cuisine, temps de préparation, etc.
- **Commentaires** : Implémentation de commentaires pour permettre aux utilisateurs de donner leur avis sur les recettes.
- **Responsivité** : Utilisation de media queries pour adapter l'affichage aux différentes tailles d'écran.
- **Accessibilité** : Implémentation des rôles ARIA et respect des normes WCAG pour assurer une navigation accessible.

### Livrables Attendus
- **Code Source** : Le code complet du site (HTML, CSS, JS) avec des commentaires appropriés.
- **Documentation** : Un document expliquant les choix de conception, l'approche de responsivité, les mesures prises pour assurer l'accessibilité, et les tests effectués.
- **Démo en Ligne** : Un lien vers une version en ligne du site pour une démonstration facile.

### Tests et Validation
- **Tests de Responsivité** : Utilisation de BrowserStack pour tester l'affichage sur différentes tailles d'écran et navigateurs.
- **Tests d'Accessibilité** : Utilisation de Google Lighthouse et d'autres outils d'accessibilité pour s'assurer que le site est conforme aux normes WCAG et RGAA.
- **Tests de Performance** : Vérification de la vitesse de chargement et optimisation des ressources (images, scripts).

---

## Bloc 2 : Développement Back-End d'applications Web

### Outils et Technologies
- **Langages** : PHP, Python, Node.js, SQL
- **Base de données** : MySQL
- **Outils** : éditeur de code (ex : VSCode), outil de modélisation (ex : UML)

### Principales Fonctionnalités Attendues

**Développement from scratch :**
- Développer les fonctionnalités du back-end sans utiliser de frameworks ou de librairies prédéfinies.
- Utiliser PHP, Python ou Node.js pour coder les contrôleurs, les modèles et les vues.
- Utiliser les concepts liés à la Programmation Orientée Objet (POO).
- Implémenter des fonctionnalités de gestion des recettes, des catégories, des commentaires et des utilisateurs.

**Fonctionnalités utilisateurs :**
- Créer un système de gestion des comptes utilisateurs avec différents rôles (administrateur, utilisateur).
- Implémenter des fonctionnalités d'inscription, de connexion, de modification de profil et de récupération de mot de passe.
- Gérer les permissions et les rôles pour les différentes sections du site.

**Gestion des recettes :**
- Ajouter, modifier et supprimer des recettes.
- Gérer les catégories et les tags des recettes.
- Afficher les recettes selon différentes catégories et tags.
- Recherche avancée par ingrédients, type de cuisine, temps de préparation, etc.

**Commentaires et interaction :**
- Permettre aux utilisateurs de commenter les recettes.
- Modérer les commentaires (ajout, modification, suppression).
- Permettre aux utilisateurs de sauvegarder leurs recettes préférées.

**Manipulation des données :**
- Utiliser MySQL pour gérer les données de l'application.
- Implémenter des opérations CRUD pour les recettes, les catégories, les commentaires et les utilisateurs.
- Optimiser les requêtes SQL pour améliorer les performances de l'application.

### Livrables Attendus
- Code source du back-end de l'application.
- Schémas fonctionnels et modèles de données.
- Documentation technique incluant les diagrammes UML et la description des fonctionnalités.
- Script SQL pour la création et la gestion de la base de données.

### Tests et Validation
- Tests unitaires pour les fonctionnalités du back-end.
- Tests d'intégration pour assurer la cohérence entre les différentes parties de l'application.
- Revues de code par les pairs pour vérifier la qualité et la maintenabilité du code.

---

## Bloc 3 : Framework

### Outils et Technologies
- **Frameworks** : React, Angular, Symfony, Laravel, Flask (au choix de l'apprenant)
- **Outils de Développement** : Webpack, Babel, TypeScript (pour Angular), Vite.js (pour React)

### Principales Fonctionnalités Attendues
- **Affichage dynamique des recettes** : Récupération et affichage des recettes via l'API.
- **Gestion de l'état** : Utilisation de hooks (React), services (Angular), ou composants contrôleurs (Symfony, Laravel) pour gérer l'état de l'application.
- **Routage** : Mise en place de la navigation entre les différentes pages de l'application (liste des recettes, détail d'une recette, soumission de recettes).
- **Interactivité** : Ajout d'interactions utilisateur, telles que la soumission et la gestion des recettes sans rechargement de la page (AJAX).

### Livrables Attendus
- **Code Source** : Le code complet de l'application développée avec le framework choisi, bien documenté et structuré.
- **Documentation** : Un guide utilisateur et une documentation technique expliquant l'utilisation des principales fonctionnalités et les choix technologiques.
- **Démo en Ligne** : Une version déployée de l'application accessible en ligne.

### Tests et Validation
- **Tests Unitaires** : Utilisation de frameworks de test comme Jest (React), Jasmine/Karma (Angular) pour tester les composants individuellement.
- **Tests Fonctionnels** : Vérification des fonctionnalités principales de l'application dans des environnements de production simulés.
- **Tests de Performance** : Optimisation du bundle, réduction du temps de chargement.

---

## Modalités d'évaluation dans le cadre de la certification RNCP

Mises en situation professionnelle sous forme de projets, conformément aux standards définis par France Compétences, les participants seront amenés à imaginer, maquetter (si cela est requis par le projet) et développer une application web spécifique. Ce développement inclura, si le projet l'exige, des aspects cruciaux tels que la sécurité, l'automatisation des processus et le déploiement de l'application, en utilisant les langages, technologies, framework, et logiciels divers étudiés durant la formation.

Le candidat présentera son projet devant un jury d'évaluation composé de deux professionnels (chefs d'entreprise) justifiant d'au moins trois ans d'expérience dans les métiers visés, tels que le développement web. Il argumentera ses choix de conception, de développement et d'implémentation, montrant sa capacité à répondre aux exigences techniques et fonctionnelles tout en garantissant la sécurité, la fiabilité et l'efficacité opérationnelle de l'application.

Le candidat devra être capable de modifier son code ou son processus de travail en temps réel pour répondre aux demandes imprévues des membres du jury. Il est également attendu que le candidat soit force de proposition, apportant des idées innovantes ou des solutions améliorées lorsque sollicité par le jury.

Pour mener à bien les projets, des éléments seront fournis aux candidats (cahier des charges, fonctionnalités, etc.) et des éléments précis seront demandés après réalisation des projets. Les éléments précis seront adaptés suite à l'entretien de positionnement et avec les modalités d'évaluation fixées par le certificateur. Pour en savoir plus, visitez le site officiel sur France Compétences. Concernant le stage obligatoire, le tuteur évalue la période de stage. Le jury de certification est piloté par le certificateur.
