# Supervision

Un homelab qui rend service au quotidien doit prévenir quand quelque chose tombe, et le faire sans consommer la moitié de la machine.

## Ce qui est surveillé

- **Les conteneurs** : découverte automatique, état, redémarrages
- **Les services exposés** : test HTTP et TCP, y compris depuis l'extérieur
- **Les certificats TLS** : date d'expiration, avec alerte avant l'échéance
- **Les métriques de la machine** : charge, mémoire, disque
- **Les ports publiés** : le superviseur signale ce qui est joignable depuis l'extérieur

Ce dernier point est le plus utile en pratique : il transforme une politique de pare-feu en quelque chose de vérifiable.

## Comment je suis alerté

Les alertes partent sur Discord, dans un salon dédié. Un service qui tombe, un certificat qui approche de l'expiration, un disque qui se remplit : le message arrive sur le téléphone sans avoir à ouvrir une interface.

## Le remplacement qui m'a servi de leçon

La supervision était assurée par un outil très répandu dans le monde de l'auto-hébergement. En regardant la consommation réelle de la machine, il prenait **184 Mo de mémoire au repos** pour une vingtaine de vérifications. Sur 8 Go, c'est beaucoup pour une tâche qui consiste à envoyer des requêtes HTTP.

Je l'ai remplacé par un outil plus récent, en conservant les données de l'ancien au cas où. Résultat : **14 Mo au repos**, pour plus de fonctions. Découverte automatique des conteneurs via le socket Docker, tests HTTP, TCP et TLS, métriques, alertes, et une lecture des ports exposés.

L'ancienne configuration et ses données sont toujours sur le disque, dans une archive de 9 Mo, avec la commande de retour arrière documentée. Remplacer un outil ne doit jamais être un aller sans retour.

## Ce que l'incident m'a appris

**Mesurer avant de choisir.** Un outil qui « marche » n'est pas un outil adapté à la machine sur laquelle il tourne. Sur un mini PC, le chiffre qui compte est la mémoire au repos, pas le nombre de fonctions sur la page d'accueil.

**La supervision ne doit pas être le voisin bruyant.** Un outil de surveillance qui consomme plus que ce qu'il surveille est un mauvais calcul. En surveillant la machine, il ne doit pas la dégrader.

**Garder le retour arrière.** Après le remplacement, l'ancien service et ses données ont été conservés jusqu'à ce que le nouveau fasse ses preuves. La règle vaut pour toute migration : on ne supprime qu'après avoir observé.

## Ce que je surveillerai ensuite

- L'état SMART du SSD, qui porte le système et les conteneurs
- L'espace du disque USB, qui descend régulièrement avec les enregistrements
- La date d'expiration du certificat, en plus de l'alerte automatique du WAF
