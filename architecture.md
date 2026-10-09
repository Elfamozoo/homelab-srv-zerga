# Architecture et réseau

Le principe est simple : une seule porte d'entrée depuis l'extérieur, et tout ce qui sert à administrer reste à l'intérieur du réseau local.

## Le trafic entrant

Internet arrive sur la box, qui fait la translation d'adresses. Sur le serveur, seuls les ports du reverse proxy sont ouverts, et derrière lui un WAF.

- **HTTPS en 443** : le WAF termine le TLS, filtre la requête, puis la transmet au service concerné
- **HTTP en 80** : uniquement pour la redirection vers HTTPS et la validation du certificat
- **Tout le reste est refusé par défaut** : la politique du pare-feu est `deny` en entrée

Le certificat est émis automatiquement par Let's Encrypt, renouvelé par le WAF.

## Le WAF

C'est lui qui expose le service média à distance. Il est là pour deux raisons : ne pas exposer un service applicatif directement sur Internet, et centraliser le TLS.

- Terminaison TLS et redirection HTTP vers HTTPS
- ModSecurity avec le jeu de règles OWASP CRS
- Journalisation et blocage automatique des comportements anormaux
- Limitation du débit par client

Le choix de l'image tout-en-un (proxy, ordonnanceur et interface dans un seul conteneur) est assumé : sur un N100 avec 8 Go de RAM, la version éclatée en trois conteneurs n'apportait rien. Le prix est une empreinte mémoire plus élevée, environ 700 Mo, c'est le plus gros consommateur de la machine.

## Ce qui n'est pas exposé

Tout ce qui touche à l'administration, à la supervision ou au stockage est en réseau local uniquement, et les règles du pare-feu le disent explicitement : DNS, tableau de bord, supervision, boîte à outils, veille, OSINT, SSH, Cockpit, partage de fichiers. Aucun de ces services n'a de règle d'entrée ouverte à tout le monde.

Le serveur DNS filtrant mérite une mention : il est utilisé par tout le réseau et fait aussi office de résolveur pour le serveur lui-même. Le fichier de résolution est donc figé sur l'adresse locale, avec un résolveur public en secours, pour que le serveur garde un accès DNS même si le conteneur de filtrage tombe. C'est la conséquence directe de l'incident décrit dans [incidents-et-lecons.md](incidents-et-lecons.md).

## Le réseau interne

| | |
|---|---|
| Sous-réseau | 192.168.1.0/24 |
| Adressage | bail statique réservé pour le serveur |
| DNS | AdGuard Home, annoncé à tout le réseau |
| IPv6 | actif |

Les conteneurs vivent sur des réseaux Docker séparés, créés par Compose. Le WAF joint les services par leurs adresses internes, jamais par une exposition publique. L'interface Wi-Fi du mini PC est désactivée, la machine est en filaire.

## Ce qui tourne à côté de Docker

- **SSH**, déplacé sur un port non standard, accessible uniquement depuis le réseau local
- **fail2ban**, actif, qui bannit les tentatives d'authentification répétées
- **Samba**, pour le partage de fichiers avec les postes Windows du réseau local
- **unattended-upgrades**, pour les correctifs de sécurité du système
- **smartmontools**, pour l'état du disque
- **Hermes**, l'agent personnel, qui tourne en service système et gère ses tâches planifiées

## Les choix que j'assume

**Un WAF plutôt qu'un simple proxy.** J'ai d'abord utilisé Caddy : simple, TLS automatique, efficace. Je l'ai remplacé par BunkerWeb parce que je voulais comprendre et manipuler ModSecurity et un jeu de règles OWASP autrement qu'en théorie. Le prix : une empreinte mémoire bien plus lourde et une configuration plus raide.

**Tout épingler par digest.** Aucune image en `latest` : chaque conteneur référence un hachage SHA256, ce qui garantit que la mise à jour est un acte volontaire et jamais une surprise. La mise à jour consiste à tirer la nouvelle image, relever son digest, remplacer la référence, puis recréer le conteneur après avoir inspecté celui qui tourne.

**Un seul point de défaillance.** Ce n'est pas de la haute disponibilité et ça ne prétend pas l'être. Un mini PC, un SSD, un disque USB : si le SSD lâche, tout s'arrête. C'est accepté en connaissance de cause, et c'est précisément ce qui rend les sauvegardes obligatoires, voir [stockage-sauvegardes.md](stockage-sauvegardes.md).
