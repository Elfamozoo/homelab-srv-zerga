# Incidents et leçons

Ce qui a cassé, ce que j'ai compris, et ce que j'ai changé pour que ça ne recommence pas. C'est la partie la plus utile de cette documentation : la configuration se réécrit, les leçons se paient une fois.

## La coupure DNS du réseau entier

**Ce qui s'est passé.** Une commande de recréation de la pile Docker a recréé le conteneur du serveur DNS sans ses volumes de configuration. Le fichier Compose ne correspondait plus à ce qui tournait réellement. Résultat : plus de résolution de noms pour tout le réseau de la maison, et l'obligation de basculer en urgence sur un résolveur public pour retrouver un accès à Internet.

**La cause.** Le fichier Compose n'avait pas été mis à jour quand les volumes avaient été ajoutés au conteneur. La recréation a donc appliqué fidèlement une configuration incomplète.

**Ce que j'ai changé.** Avant toute recréation, j'inspecte le conteneur en cours (montages, variables, ports, réseau) et je compare avec le fichier. Et le résolveur du serveur ne dépend plus de lui-même : l'adresse locale reste en premier, un résolveur public prend le relais.

**La leçon.** Le fichier de déploiement n'est pas la vérité, l'état réel l'est. Et un service qui sert tout le réseau ne doit pas être le seul à pouvoir résoudre les noms.

## Banni par son propre pare-feu applicatif

**Ce qui s'est passé.** Après une montée de version majeure du serveur média, un client mobile gardait un jeton d'authentification périmé et appelait la même URL en boucle. Le service répondait une erreur d'authentification. Le WAF, qui bannit automatiquement après une série de réponses d'erreur, a conclu à une attaque et bloqué l'adresse publique pendant 24 heures.

**La cause.** Deux logiques correctes qui se contredisent : un client qui réessaie, et un module anti-abus qui compte les erreurs. Un serveur média produit naturellement des réponses d'erreur, c'est dans son fonctionnement.

**Ce que j'ai changé.** Le blocage automatique sur erreurs a été désactivé pour ce service, en gardant tout le reste du filtrage : règles OWASP, listes de blocage, limitation de débit. La procédure de débannissement est documentée : lister les bannissements, puis lever celui du service concerné, parce que supprimer la clé en base ne suffit pas, le WAF garde le bannissement en mémoire.

**La leçon.** Une protection automatique doit être réglée pour le service qu'elle protège, pas pour un cas général. Et après une montée de version majeure côté serveur, il faut penser aux clients : vider le cache, se reconnecter.

## La mise à jour qui refuse de démarrer

**Ce qui s'est passé.** Le service de cloud faisait l'objet d'une mise à jour majeure. La nouvelle version a refusé de s'ouvrir sur les données existantes : deux versions d'écart, chemin de migration non prévu. Le conteneur a été remis sur la version précédente, l'accès est revenu en quelques minutes.

**La cause.** Une marche de deux versions majeures sur un service qui porte une base de données. Le chemin propre n'existait que version par version.

**Ce que j'ai changé.** L'image a été conservée sur une version figée, et le chemin de migration est écrit dans la documentation interne : version intermédiaire, migration, puis version suivante. Aucun saut de version ne se fait plus sans sauvegarde préalable.

**La leçon.** Épingler les versions rend le retour arrière trivial. C'est exactement au moment où une mise à jour casse qu'on mesure ce que vaut cette discipline.

## L'image officielle devenue intéléchargeable

**Ce qui s'est passé.** Le registre officiel a commencé à refuser le téléchargement d'une image (erreur 403). Le service ne pouvait plus être reconstruit.

**Ce que j'ai changé.** Recherche d'une alternative maintenue sur un autre registre, vérification de l'antériorité et du sérieux du dépôt, puis bascule. L'image retenue est elle aussi épinglée par digest.

**La leçon.** Ne pas dépendre d'un seul registre, et ne pas prendre la première image trouvée : regarder d'abord qui la maintient et à quelle fréquence.

## La supervision qui consommait plus que les services surveillés

**Ce qui s'est passé.** En relevant la consommation réelle de la machine, l'outil de supervision affichait 184 Mo de mémoire au repos pour une vingtaine de vérifications, sur une machine de 8 Go.

**Ce que j'ai changé.** Remplacement par un outil plus léger, découverte automatique des conteneurs par le socket Docker, 14 Mo au repos pour davantage de fonctions. L'ancien outil et ses données sont conservés, avec la procédure de retour arrière.

**La leçon.** Mesurer, pas supposer. La supervision doit être discrète, sinon elle fait partie du problème.

## Ce que ces incidents m'ont appris en général

**Documenter au moment où l'on répare.** C'est la seule fois où on a toutes les informations en tête. Aucun de ces incidents n'a d'abord été corrigé dans un fichier, ils l'ont tous été dans la précipitation, avec la compréhension qui vient après.

**Le retour arrière d'abord.** Sauvegarder avant, conserver l'ancien après. C'est le pire moment pour se demander si on aurait dû.

**Écrire ce qu'on croit avoir configuré, puis comparer avec la réalité.** L'écart entre les deux est exactement là où arrivent les incidents. Cette documentation est née de cette comparaison.
