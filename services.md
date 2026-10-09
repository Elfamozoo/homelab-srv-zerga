# Services

Huit conteneurs tournent en permanence. Ils sont décrits ici avec leur empreinte mémoire mesurée, parce que sur une machine de 8 Go c'est le chiffre qui décide de ce qu'on peut ajouter.

| Service | Rôle | RAM au repos |
|---|---|---|
| BunkerWeb | WAF et reverse proxy | ~715 Mo |
| Jellyfin | Serveur média | ~350 Mo |
| ChangeDetection | Veille de pages web | ~99 Mo |
| Homepage | Tableau de bord | ~84 Mo |
| AdGuard Home | DNS et filtrage | ~72 Mo |
| Maintenant | Supervision | ~32 Mo |
| Spiderfoot | OSINT | ~7 Mo |
| IT-Tools | Outils réseau et développement | ~1 Mo |

Plus un service système, Hermes, qui gère les tâches planifiées et les sauvegardes.

Au total, environ 2,8 Go de RAM utilisés sur 7,5 Go, avec une charge moyenne sous 0,4. La machine respire.

## À quoi sert chaque service

**BunkerWeb** est la seule porte d'entrée. Il termine le TLS, filtre les requêtes avec ModSecurity et le jeu de règles OWASP, limite le débit et journalise. C'est aussi le plus gourmand, de loin : l'image tout-en-un embarque nginx, l'ordonnanceur de configuration et une interface d'administration.

**Jellyfin** est le serveur média de la maison. Les fichiers sont sur le disque USB, l'accès distant passe par le WAF, l'accès local est direct. L'accélération matérielle est activée via le GPU intégré du N100, ce qui évite de saturer les quatre coeurs au moment de transcoder.

**AdGuard Home** filtre le DNS pour tout le réseau. C'est le service le plus critique au quotidien : s'il tombe, plus personne ne résout de nom. Le serveur lui-même garde un résolveur de secours pour ne pas dépendre d'un conteneur qu'il héberge.

**Homepage** est le tableau de bord : chaque service avec son état et son lien. La règle que je me suis fixée est de le mettre à jour à chaque ajout ou retrait de service, sinon il devient un mensonge.

**Maintenant** supervise le reste. Il découvre les conteneurs tout seul, surveille les ports, les certificats et les métriques, et envoie ses alertes sur Discord. Détail dans [supervision.md](supervision.md).

**ChangeDetection** surveille des pages web et me prévient quand leur contenu change. Utile pour suivre une page d'annonce, une disponibilité ou une documentation.

**IT-Tools** regroupe une centaine de petits utilitaires réseau et développement. C'est le couteau suisse, il ne consomme rien.

**Spiderfoot** sert à la reconnaissance OSINT sur des domaines et des adresses. Il est strictement local : cet outil ne s'expose pas, et le pare-feu le dit.

## Les images

Aucune image n'est référencée en `latest`. Chaque conteneur pointe vers un digest SHA256, ce qui fige la version exacte qui tourne :

```yaml
services:
  changedetection:
    image: ghcr.io/dgtlmoon/changedetection.io@sha256:5438423d…7882
```

La mise à jour est donc un geste délibéré : tirer la nouvelle image, relever son digest, remplacer la référence dans le fichier Compose, puis recréer le conteneur. Et avant de recréer, on inspecte l'existant, parce que le fichier Compose n'est pas toujours la vérité sur ce qui tourne réellement. Cette règle vient d'un incident, raconté dans [incidents-et-lecons.md](incidents-et-lecons.md).

## Organisation des fichiers

```
/srv/docker/
├── docker-compose.yml          # services principaux
├── adguard/                    # configuration et filtres DNS
├── jellyfin/                   # compose + configuration du média
├── bunkerweb/data/             # données du WAF
├── homepage/                   # configuration du tableau de bord
├── changedetection/            # base de la veille
├── homarr/                     # ancien tableau de bord, conservé
└── caddy/                      # ancien reverse proxy, conservé
```

Chaque service a son dossier et son fichier Compose quand il a des spécificités (réseau hôte, périphérique, volumes particuliers). Un Compose principal regroupe les services simples.

**Environ 4,6 Go** de données dans `/srv/docker`, dont 3,4 Go pour le seul serveur média.

## Ce qui traîne

Un inventaire honnête voit aussi le ménage en retard :

- 4 conteneurs arrêtés : l'ancien WAF, une interface de test d'un modèle de langage, et deux conteneurs de démonstration oubliés
- 30 images pour 30 Go au total, dont **20,7 Go récupérables** (68 pour cent) : anciennes versions conservées pour un retour arrière, images de test
- 4 volumes locaux, dont 1,5 Go inutilisé

Rien de cassé, mais c'est de la place prise pour rien, et ça se nettoie en une commande quand j'aurai vérifié qu'aucun retour arrière ne dépend encore de ces images.
