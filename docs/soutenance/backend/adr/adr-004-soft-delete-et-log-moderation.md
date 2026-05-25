# ADR-004 — Soft-delete et log de modération

## Statut

Accepté · 2026-05-25

## Contexte

L'application contient du contenu créé par les utilisateurs (recettes,
commentaires) qui peut être modéré ou supprimé. Trois acteurs peuvent
déclencher une "disparition" du contenu :

- L'auteur (suppression de son propre commentaire, archivage de sa
  recette).
- Un administrateur (modération).
- Le système (cascade lors de la suppression d'un user banni).

Il faut décider si on supprime physiquement les lignes (`DELETE`) ou si
on les marque comme inactives (soft-delete).

## Décision

**Soft-delete par défaut sur les contenus modérables**, avec colonnes
explicites pour qui/quand/pourquoi :

### Recipes

- `Status ENUM('draft', 'pending', 'published', 'rejected', 'archived')`
- `ModeratedAt`, `ModeratedByUserId`, `RejectionReason`
- `ArchivedAt`
- Pas de colonne `DeletedAt` : l'archivage joue ce rôle. La suppression
  physique est réservée à l'admin via une action distincte (`DELETE
  /admin/recipes/:id`).

### Comments

- `ModeratedAt`, `ModeratedByUserId` (commentaire masqué pour
  modération)
- `DeletedAt`, `DeletedByUserId` (commentaire supprimé)
- Suppression physique uniquement par l'admin sur action explicite
  (`hardDelete`).

### Users

- `Status ENUM('inactive', 'active', 'banned')`
- `BannedByUserId`, `BannedReason`, `BannedAt`
- Pas de suppression physique : un user banni reste en base pour
  conserver l'auteur des recettes/commentaires qu'il a produits.

### Log de modération

Table dédiée `UserModerationLogs` qui trace **chaque action** (ban /
unban) avec son auteur, sa cible, son motif, sa date. Indépendante du
statut courant du user — si on dé-bannit puis re-bannit, l'historique
des deux actions est conservé.

## Alternatives considérées

### Hard-delete partout

- **Pour** : simple, RGPD-friendly de base, pas de logique de filtrage.
- **Contre** : perd l'audit trail. Impossible de répondre à *"qui a
  supprimé ce commentaire et quand ?"*.
- **Contre** : cascade lourde sur les FK (commentaires d'un user
  supprimé → recettes orphelines → favoris orphelins).

### Colonne `IsDeleted BOOLEAN` simple

- **Pour** : minimal.
- **Contre** : perd l'information "qui" et "quand". Pas de
  différenciation entre modération et suppression auteur.

## Conséquences

### Positives

- **Traçabilité complète** des actions modérateur : la table
  `UserModerationLogs` est immuable (pas d'`UPDATE`), elle constitue
  un audit trail légalement défendable.
- **Réversibilité** : un commentaire modéré peut être restauré
  (`AdminCommentService.unmoderate`), un user banni peut être
  dé-banni — sans perte d'information.
- **Cohérence référentielle** : les recettes/commentaires d'un user
  banni restent visibles (s'ils étaient publiés), avec l'auteur
  identifié. Pas de "Auteur supprimé" cryptique côté UI.

### Négatives

- **Filtrage permanent** : toutes les requêtes "lecture publique"
  doivent filtrer par statut (`WHERE Status = 'published' AND
  DeletedAt IS NULL`). Mitigation : les méthodes du repository
  encapsulent ce filtrage (`findPublished`, `findPublishedBySlug`).
- **RGPD** : un user qui demande l'effacement de ses données nécessite
  une procédure manuelle (hard-delete + nettoyage des FK
  `ModeratedByUserId`, `DeletedByUserId`, `BannedByUserId`). Non
  implémenté ici, à mentionner comme évolution.
- **Volume BDD** : les contenus modérés ne sont jamais purgés
  automatiquement. À l'échelle du projet, négligeable.

### À surveiller

- Risque de "fuite" via une requête qui oublie le filtre de
  soft-delete. Mitigation à terme : passer par une vue SQL
  `Comments_Public` qui pré-filtre, et n'utiliser que cette vue dans
  les requêtes de lecture publique.
