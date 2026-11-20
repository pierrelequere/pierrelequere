---
title: "Nextcloud et Talk sur des VM, sans Kubernetes : une architecture type"
description: Deux sites, peu de VM, du temps réel et tout en Ansible. Les choix d'architecture et les pièges à éviter.
tags: [Architecture, Nextcloud, Ansible]
---

Pas de Kubernetes, pas de service managé, une capacité d'hébergement comptée…
et pourtant il faut une plateforme de fichiers **et** de visio prête pour la production,
répartie sur deux sites. C'est un scénario classique en cloud privé ou on-premise.
Voici une architecture type pour y répondre, et surtout le *pourquoi* de chaque choix.

![Schéma de l'architecture Nextcloud et Talk sur VM]({{ '/assets/img/blog/nextcloud-talk-vm.svg' | relative_url }})

## Le scénario

- Nextcloud pour les fichiers, **Nextcloud Talk** pour l'audio et la vidéo, avec le
  *High Performance Backend* (HPB) dès qu'on dépasse quelques participants.
- **Uniquement des machines virtuelles** : chaque composant tourne en service système.
- **Deux sites**, pour survivre à la perte de l'un des deux.
- **PostgreSQL** comme base de données.
- Tout est **déployé par Ansible** : aucune VM configurée à la main.
- Et une contrainte forte : **le moins de VM possible**.

Cette dernière contrainte guide tout le reste. L'idée n'est pas de tout empiler sur
une seule VM, mais de **mutualiser seulement ce qui peut l'être sans risque**.

## Ce qu'on mutualise… et ce qu'on refuse de mutualiser

| Rôle | Choix | Pourquoi |
|---|---|---|
| HAProxy / NGINX | **Séparés**, une VM de chaque par site | Répartition de charge et reverse proxy n'ont pas le même cycle de vie ni les mêmes réglages |
| Cron Nextcloud | **Sur un nœud Web** | Une VM de moins, et le cron ne doit tourner qu'à un seul endroit de toute façon |
| Signaling, NATS, Janus, coturn | **Regroupés** sur une VM HPB par site | Ils forment une seule chaîne temps réel, autant les garder ensemble |
| Redis | **Isolé** | Une saturation mémoire ne doit pas faire tomber les nœuds Web |
| PostgreSQL | **Jamais mutualisé** | C'est le cœur de la plateforme |

## Le frontal : HAProxy, puis NGINX

Le chemin d'une requête est **Utilisateur → HAProxy → NGINX → Nextcloud**.
Chaque site a son HAProxy et son NGINX, et les flux sont **croisés** : un HAProxy
du site A peut envoyer vers le NGINX du site B. Si un site tombe, l'autre prend le relais.

Avec deux couches de proxy devant Nextcloud, pense à déclarer ces proxys dans
`config.php`, sinon Nextcloud voit toutes les connexions venir du proxy :

```php
'trusted_proxies'   => ['10.0.1.10', '10.0.1.11'],
'overwriteprotocol' => 'https',
'overwrite.cli.url' => 'https://cloud.exemple.fr',
```

## Le cron : une seule instance, toujours

Les tâches de fond de Nextcloud (`cron.php`) doivent tourner sur **un seul nœud**.
Les deux nœuds Web ont le même code, mais le cron n'est activé que sur l'un d'eux :

```bash
# /etc/cron.d/nextcloud, uniquement sur le nœud désigné
*/5 * * * * www-data php -f /var/www/nextcloud/cron.php
```

Dans Ansible, c'est une simple variable d'inventaire (`nextcloud_cron_enabled: true`)
posée sur un seul hôte. Pour basculer le cron, on déplace la variable et on rejoue le playbook.

## PostgreSQL : un nom, jamais une machine

Un primaire sur le site A, un secondaire répliqué sur le site B. La règle d'or :
**Nextcloud ne connaît jamais le nom d'un nœud PostgreSQL.** Il passe par un nom
logique, par exemple `postgresql.service.exemple.lan`, qui pointe toujours vers le primaire.

Derrière ce nom, on peut mettre une VIP, un proxy SQL ou un gestionnaire de bascule
comme Patroni. Le choix se fait dans le design détaillé. Ce qui compte, c'est qu'une
bascule ne demande **aucune modification côté Nextcloud**.

> Le vrai danger d'une base répartie sur deux sites, c'est le *split-brain* : deux nœuds
> qui se croient primaires en même temps. Tester la bascule fait partie du projet,
> ce n'est pas un bonus.

## Redis : petit, mais critique

Redis sert au cache et surtout au **verrouillage des fichiers** quand plusieurs nœuds
Web écrivent en même temps. Sans lui, deux nœuds Nextcloud finissent par se marcher dessus.
Il est seul sur sa VM, joignable uniquement depuis le réseau privé, et protégé par mot de passe.

## Talk et le HPB : le temps réel à part

Le HPB, ce sont quatre briques :

- **Signaling** : la signalisation WebSocket des appels ;
- **NATS** : le bus de messages interne ;
- **Janus** : le SFU qui redistribue l'audio et la vidéo ;
- **coturn** : le relais STUN/TURN pour les réseaux qui bloquent le pair-à-pair.

Une VM HPB par site embarque les quatre. Point important : **les flux média ne passent
pas par HAProxy**. Les clients joignent directement le HPB en TURN (3478 en UDP/TCP,
5349 en TLS, 443 en secours) et sur une plage UDP dédiée à WebRTC. Cette plage doit
être ouverte et dimensionnée **dès la conception réseau**, pas la veille de la mise en production.

## Les flux : tout est interdit par défaut

Chaque couche a son sous-réseau : frontal, applicatif, HPB, données, administration.
Seuls les flux suivants sont ouverts :

| De | Vers | Port |
|---|---|---|
| Utilisateurs | HAProxy | 443 |
| Utilisateurs | HPB | 3478, 5349, 443 + plage UDP média |
| HAProxy | NGINX | 443 |
| NGINX | Nextcloud Web, Signaling | 443 |
| Nextcloud Web | PostgreSQL (nom logique) | 5432 |
| Nextcloud Web | Redis | 6379 |
| PostgreSQL primaire | PostgreSQL secondaire | 5432 |
| Signaling | NATS | 4222 |
| Bastion et nœud Ansible | Toutes les VM | 22 |

PostgreSQL, Redis et l'administration ne sont **jamais** joignables depuis les postes utilisateurs.

## Ansible : un rôle par composant

Un rôle par brique (haproxy, nginx, nextcloud, postgresql, redis, nats, signaling,
janus, coturn), un inventaire par environnement et par site, et les secrets dans un
coffre (Ansible Vault ou équivalent). Le bénéfice va au-delà du déploiement :
**le code devient la documentation**, et reconstruire une VM devient banal.

## Les points à retenir

- **Mutualiser, oui, mais pas les rôles critiques.** La base et Redis restent seuls.
- **Surveiller de près les VM qui portent plusieurs rôles**, surtout le HPB :
  CPU, mémoire et réseau. C'est là que la saturation arrivera en premier.
- **Prévoir la sortie.** Le jour où la charge augmente, on sépare d'abord Janus
  et coturn, puis NATS et Signaling, puis le cron. Le design le permet sans tout casser.
- **Écrire l'ordre de reprise.** Réseau, DNS, PostgreSQL, Redis, stockage, frontaux,
  Nextcloud, HPB, puis une validation fonctionnelle complète.

Et toi, comment as-tu monté Talk HPB ? D'autres choix de mutualisation, un autre
mécanisme de bascule PostgreSQL ? Je suis preneur de ton retour en commentaire.
