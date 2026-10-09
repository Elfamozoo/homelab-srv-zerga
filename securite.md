# Sécurité

Sur une machine exposée à Internet, la sécurité tient à trois choses : une politique de pare-feu par défaut restrictive, une mise à jour disciplinée, et le cloisonnement de ce qui sert à administrer.

## Pare-feu

La politique par défaut est `deny` en entrée :

```
Default: deny (incoming), allow (outgoing), deny (routed)
```

Concrètement :

- **Ouvert à tout le monde** : uniquement les ports du reverse proxy, pour le HTTPS et la redirection
- **Ouvert au réseau local seulement** : DNS, tableau de bord, supervision, veille, boîte à outils, OSINT, SSH, Cockpit, partage de fichiers
- **Rien d'autre**

Les règles sont explicites et nommées une par une, y compris la restriction au sous-réseau local. Une règle qu'on ne sait plus expliquer est une règle qu'on rouvrira par erreur un jour.

## Cloisonnement

Le serveur DNS filtrant, la supervision, le tableau de bord et l'outil OSINT partagent la même règle : accessibles depuis le réseau local, jamais depuis l'extérieur. L'outil OSINT en particulier ne doit pas être joignable, pas parce qu'il est dangereux, mais parce qu'il raconte tout ce qu'on lui demande sur une cible.

Les conteneurs sont sur des réseaux Docker distincts. Le WAF joint les services par leurs adresses internes. Aucun service applicatif n'est publié directement sur une interface publique : c'est tout l'intérêt de la porte d'entrée unique.

## Comptes

- Un compte personnel, avec les droits d'administration
- Un compte dédié à l'agent, distinct du compte personnel, avec un accès restreint à ce dont il a besoin pour ses tâches
- Aucun service ne tourne en root quand l'image prévoit un utilisateur dédié

Les secrets (jetons, mots de passe d'interface) sont dans un fichier d'environnement en permissions `600`, hors du dépôt Git et hors des images. Quand un jeton passe dans une conversation, il est considéré comme compromis et remplacé.

## Mise à jour

**Les images Docker sont épinglées par digest SHA256.** Aucune n'est en `latest`, donc aucune mise à jour ne peut arriver sans qu'on la décide. La procédure est toujours la même : tirer la nouvelle image, relever son digest, mettre à jour la référence, inspecter le conteneur en cours, recréer, vérifier.

Ce que ça protège : une image qui change sous vos pieds, une régression non voulue, ou un incident chez l'éditeur qui pousse une version cassée. Le prix à payer : il faut penser à mettre à jour, et le faire exprès.

**Le système** est mis à jour automatiquement pour les correctifs de sécurité, avec un redémarrage planifié.

**Surveillance et blocage** : fail2ban est actif et bannit les adresses qui accumulent les échecs d'authentification. Le WAF bloque et journalise les comportements anormaux.

**Le réseau** est filaire uniquement, l'interface Wi-Fi est désactivée.

## Ce qui reste à faire

- **SSH en clé uniquement.** C'est la première chose que je fermerai : le mot de passe reste accepté aujourd'hui.
- **Segmentation réseau.** Tout est sur le même sous-réseau, appareils personnels compris. Un VLAN séparé pour les objets connectés limiterait les dégâts en cas de compromission d'un de ces appareils.
- **Sauvegarder les configurations.** Une machine bien configurée mais non sauvegardée reste fragile, voir [stockage-sauvegardes.md](stockage-sauvegardes.md).

## La leçon

C'est un homelab, pas un datacenter, et la sécurité y est proportionnée : l'objectif n'est pas de résister à une attaque ciblée, c'est de ne pas laisser de porte ouverte par négligence. La partie la plus utile a été de comprendre ce que la machine expose vraiment, plutôt que ce que je croyais qu'elle exposait : une politique de pare-feu se relit, se compte, et se documente.
