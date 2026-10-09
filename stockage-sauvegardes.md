# Stockage et sauvegardes

Deux disques, deux rôles : un SSD pour le système et les conteneurs, un disque USB pour les médias et les sauvegardes.

## Les disques

| Disque | Taille | Utilisation | Contenu |
|---|---|---|---|
| SSD SATA 240 Go | 233 Go | 29 pour cent | système, conteneurs, volumes Docker |
| HDD USB 2 To | 1,8 To | 67 pour cent | médias, photos, rétro-jeux, sauvegardes |

Tout est en ext4. Le disque USB est monté sur `/mnt/HOMESTORAGE` et il est le seul stockage de masse : le SSD ne contient que le système et les données des conteneurs, environ 4,6 Go dans `/srv/docker`.

## Ce qu'il y a sur le disque de 2 To

| Dossier | Taille |
|---|---|
| Séries | 757 Go |
| Films | 354 Go |
| Photos | 29 Go |
| Rétro-jeux | 13 Go |
| Sauvegardes | 2,4 Go |
| Formation | 1,8 Go |

La bibliothèque média est organisée en dossiers par série, avec les métadonnées gérées par le serveur média. Les fichiers sont déposés par le réseau local, jamais depuis l'extérieur.

## Les sauvegardes

| Quoi | Où | Quand | Détails |
|---|---|---|---|
| Configuration et données de l'agent | disque USB + dépôt Git privé | quotidien à 04:00 | archive compressée, empreinte SHA256, 5 versions conservées |
| Configuration du serveur média | disque USB | avant chaque mise à jour majeure | archive complète, 2,1 Go pour la dernière |
| Données de l'ancienne supervision | disque USB | ponctuel | 9 Mo, conservés pour un retour arrière |

Le script de sauvegarde est simple et volontairement lisible : il crée l'archive, en calcule l'empreinte, la compare, puis purge la plus ancienne. Chaque exécution laisse une ligne dans un journal.

```
[2026-10-06 04:00:52] SUCCESS | hermes-backup-20261006_040048.tar.gz | 34M | sha256 ok
[2026-10-06 04:00:53] PRUNED  | hermes-backup-20260818_040052.tar.gz
```

Deux destinations plutôt qu'une : le disque local pour la restauration rapide, un dépôt Git privé distant au cas où le disque disparaît avec la machine. Une sauvegarde qui vit sur le disque qu'elle protège n'est pas une sauvegarde.

La restauration est documentée et testée :

```bash
tar xzf hermes-backup-20261006_040048.tar.gz -C ~/
```

## Ce qui n'est pas encore sauvegardé

Et c'est la vraie faiblesse de l'installation.

Les configurations des conteneurs montées en bind dans `/srv/docker` ne sont pas incluses dans la sauvegarde automatique. Concrètement : si le SSD meurt, je perds la configuration du serveur DNS, du WAF et du tableau de bord. Les médias sont sur le disque USB et survivraient, mais la reconstruction demanderait de tout refaire à la main.

La correction prévue est simple : inclure les fichiers de configuration (pas les données média) dans l'archive quotidienne. Quelques mégaoctets, et ça change complètement la reprise après incident.

## Les limites assumées

- **Disque USB pour la bibliothèque.** Pas de redondance, un seul disque. C'est un choix de coût, et il est assumé d'autant plus facilement que les médias sont remplaçables, contrairement aux photos.
- **Photos comprises dans la même logique.** 29 Go de photos de famille qui ne sont ni redondées ni sauvegardées à distance. C'est le point que je corrigerai en premier, avant la configuration des conteneurs.
- **Pas de RAID, pas de grappe.** Un mini PC, un SSD, un disque USB. La panne est certaine un jour, la seule vraie question est ce qu'on peut restaurer.
