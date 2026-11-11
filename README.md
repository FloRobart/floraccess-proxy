# FlorAccess — Proxy

Le reverse proxy Traefik du serveur : le seul point d'entrée sur les ports 80 et 443. `floraccess-wrapper` le déploie avec le reste de l'écosystème.

Ce dépôt déclare tout ce qui est exposé : les routes, et ce qui leur est commun (HTTPS, certificats Let's Encrypt, options TLS, en-têtes de sécurité et limite de requêtes). Traefik n'a aucun accès à l'API Docker, et les autres dépôts n'ont aucun label Traefik.

# Table des matières

- [FlorAccess — Proxy](#floraccess--proxy)
- [Table des matières](#table-des-matières)
- [Structure](#structure)
- [Mettre en service](#mettre-en-service)
- [Un seul proxy par serveur](#un-seul-proxy-par-serveur)
- [Exposer un service](#exposer-un-service)
- [Ancienne adresse](#ancienne-adresse)
- [Limite de requêtes](#limite-de-requêtes)
- [Mettre à jour Traefik](#mettre-à-jour-traefik)
- [Sécurité](#sécurité)

# Structure

```
floraccess-proxy/
├── docker-compose.yml          ← Traefik
├── .env.example                ← réglages de cette machine
└── dynamic/
    ├── routes.yml              ← tout ce qui est exposé : domaines et services
    ├── rate-limit.yml          ← limite de requêtes par IP
    ├── security-headers.yml    ← en-têtes appliqués à toutes les routes
    └── tls.yml                 ← options TLS de toutes les routes
```

| Domaine | Service |
|---|---|
| `api.floraccess.florobart.fr` | API (`floraccess-api`) |
| `floraccess.florobart.fr` | Panel d'administration (`floraccess-app`) |
| `floraccess-server.florobart.fr` | API, ancienne adresse (voir [Ancienne adresse](#ancienne-adresse)) |

# Mettre en service

Il n'y a rien à lancer depuis ce dépôt : tout passe par `floraccess-wrapper`, dont le README décrit la mise en service.

Le réglage de cette machine se trouve dans `env/proxy/.env` du wrapper. `LETSENCRYPT_EMAIL` doit y être rempli.

Compose crée le réseau `traefik-net` avec IPv6. Sans IPv6 sur ce réseau, Docker relaierait les visiteurs IPv6 en IPv4 et leur adresse serait perdue : ils partageraient tous la même limite de requêtes.

# Un seul proxy par serveur

Ce Traefik publie les ports 80 et 443 : **un serveur ne peut en faire tourner qu'un seul**. `econoris-proxy` publie les mêmes ports. Si Éconoris et FlorAccess sont sur le même serveur, le second Traefik ne démarre pas (`port is already allocated`).

Dans ce cas, un seul des deux proxys doit tourner, avec les routes des deux écosystèmes dans son `dynamic/routes.yml`, et son Traefik doit rejoindre les réseaux exposés des deux écosystèmes (`floraccess-api-net` et `floraccess-app-net` en plus de ceux d'Éconoris).

# Exposer un service

1. Dans le `docker-compose.yml` du dépôt, le service rejoint son propre réseau interne :

    ```yaml
    services:
      floraccess-app:
        networks:
          - floraccess-app-net

    networks:
      floraccess-app-net:
        name: floraccess-app-net
        internal: true
    ```

2. Dans `docker-compose.override.yml` du wrapper, Traefik rejoint ce réseau :

    ```yaml
    services:
      traefik:
        networks:
          - floraccess-app-net
    ```

3. Dans `dynamic/routes.yml` de ce dépôt, déclarer son routeur et son service. Traefik joint le service par son nom :

    ```yaml
    http:
      routers:
        floraccess-app:
          rule: "Host(`floraccess.florobart.fr`)"
          entryPoints:
            - websecure
          service: floraccess-app

      services:
        floraccess-app:
          loadBalancer:
            servers:
              - url: "http://floraccess-app:8080"
            healthCheck:
              path: /
              interval: 10s
              timeout: 5s
    ```

Il n'y a rien d'autre à déclarer : le certificat `le`, les en-têtes de sécurité et la limite de requêtes s'appliquent par défaut.

⚠️ **Ne pas écrire de clé `tls` dans un routeur.** Traefik n'applique le certificat par défaut qu'aux routes sans aucune configuration TLS : la route servirait un certificat invalide. Si la route doit préciser son TLS, utiliser `tls.certResolver: le`.

**Le service ne publie aucun port.** C'est la responsabilité de chaque dépôt : un port publié contourne Traefik, donc le HTTPS et la limite de requêtes, et Docker l'ouvre même si le pare-feu du serveur le bloque.

Si un service est arrêté, sa route reste déclarée : le visiteur reçoit une erreur `502` ou `503`.

# Ancienne adresse

`floraccess-server.florobart.fr`, l'ancienne adresse de l'API, est encore servie, sans redirection : une redirection perdrait le corps des requêtes `POST`. Elle reste utile tant que :

- des applications appellent encore l'API à cette adresse (anciennes versions de l'extension Éconoris, applications pas encore mises à jour) ;
- des emails déjà envoyés contiennent des liens vers elle (vérification d'adresse, blocage des connexions).

L'enregistrement DNS de `floraccess-server.florobart.fr` doit donc pointer vers ce serveur tant que la route est utile. Pour la supprimer, retirer le routeur `floraccess-legacy-api` de `dynamic/routes.yml`.

# Limite de requêtes

Chaque adresse IP a droit à **30 requêtes par seconde en continu, avec des pics jusqu'à 150**. Au-delà, Traefik répond `429`. Le compteur est séparé pour chaque route. En IPv6, toutes les adresses d'un même `/64` comptent pour une seule.

⚠️ L'API d'une application qui utilise FlorAccess (comme `econoris-api`) vérifie le JWT de chacune de ses requêtes auprès de FlorAccess, toujours depuis l'adresse de son serveur : tous ses utilisateurs partagent cette adresse. La limite de l'API FlorAccess elle-même (`REQUEST_LIMIT_PER_SECOND`, 10 requêtes par seconde en moyenne par défaut) est plus stricte que celle-ci : c'est elle qu'il faut augmenter en premier si ces applications grandissent.

Le réglage est dans `dynamic/rate-limit.yml`, où le calcul est expliqué. Une modification part au prochain `deploy.sh` du wrapper, une fois sur `main`.

Un routeur de `dynamic/routes.yml` peut ajouter une limite plus stricte, par exemple sur la connexion. Une limite plus large que celle-ci n'aurait aucun effet.

# Mettre à jour Traefik

L'image est figée sur une version et un digest, donc **les correctifs de sécurité ne s'installent pas seuls**. Dependabot ([.github/dependabot.yml](.github/dependabot.yml)) vérifie chaque mois les nouvelles versions et ouvre une pull request vers `dev` qui change la version et le digest dans les deux services de `docker-compose.yml`. Traefik publie souvent : pour une faille urgente, suivre [ses releases](https://github.com/traefik/traefik/releases) sans attendre Dependabot.

Pour mettre à jour à la main, changer la version et le digest des deux lignes `image: traefik:…` de `docker-compose.yml`, toujours ensemble. Une fois la mise à jour publiée sur `main`, lancer le `deploy.sh` du wrapper.

# Sécurité

- **Traefik n'a aucun accès à l'API Docker.** Le socket Docker n'est monté dans aucun conteneur. Si Traefik était compromis, il ne pourrait ni lire les variables, les journaux et les fichiers des autres conteneurs, ni agir sur eux.
- **Un conteneur ne peut pas se déclarer une route.** Seul `dynamic/routes.yml` expose un service : une image piégée ne peut ni s'exposer, ni détourner un domaine.
- **Traefik est le seul lien entre les services exposés.** Chacun a son réseau, sans accès à Internet si possible, et aucun ne peut en joindre un autre. Traefik, lui, les atteint tous : un service ne doit donc pas faire confiance à un appel sous prétexte qu'il vient de son réseau, et une base de données n'a jamais sa place sur un réseau de Traefik.
- **Le HSTS couvre les sous-domaines** : un navigateur qui a visité un domaine refusera le HTTP sur tous ses sous-domaines pendant un an. Tout nouveau sous-domaine doit donc être en HTTPS dès le départ.
