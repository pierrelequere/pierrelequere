---
title: "Nextcloud et Talk sur des VM avec NixOS : toute la plateforme dans un dépôt Git"
description: Une plateforme sur deux sites où chaque VM est décrite en Nix. Premiers essais, extraits de config et pièges.
tags: [NixOS, Nextcloud, Architecture]
---

On teste en ce moment une idée simple : et si chaque VM d'une plateforme Nextcloud
était **entièrement décrite par un fichier**, versionné dans Git, et reconstructible
à l'identique ? C'est exactement ce que propose **NixOS**. On ne configure pas une
machine : on déclare ce qu'elle doit être, et NixOS la construit.

Le scénario : une plateforme de fichiers **et** de visio prête pour la production,
sur **deux sites**, avec **uniquement des VM** (pas de Kubernetes, pas de service managé),
Nextcloud et Talk avec son backend haute performance (HPB), et **le moins de VM possible**.
On mutualise ce qui peut l'être sans risque, et on garde seuls les rôles critiques :
la base PostgreSQL et Redis.

![Schéma de l'architecture Nextcloud et Talk sur VM avec NixOS]({{ '/assets/img/blog/nextcloud-talk-vm-nixos.svg' | relative_url }})

## Pourquoi NixOS pour ce genre de plateforme

- **Pas de dérive de configuration.** Une VM est exactement ce que dit son fichier.
  Une modification faite à la main disparaît au prochain déploiement.
- **Retour arrière immédiat.** Chaque déploiement crée une nouvelle *génération*.
  Une mise à jour qui casse quelque chose ? `nixos-rebuild switch --rollback` et on repart.
- **Des versions figées.** Le fichier `flake.lock` fixe la version exacte de tous les
  paquets. Deux VM construites à six mois d'écart sont identiques.
- **Beaucoup de briques déjà prêtes.** Nextcloud, PostgreSQL, Redis, NGINX, HAProxy,
  coturn et NATS ont chacun leur module NixOS.

## L'organisation du dépôt

Un seul dépôt, un *flake* qui déclare toutes les machines, et des modules partagés :

```text
plateforme/
├── flake.nix          # liste des VM
├── flake.lock         # versions figées
├── modules/
│   ├── commun.nix     # SSH, temps, pare-feu, supervision
│   ├── haproxy.nix
│   ├── nginx.nix
│   ├── nextcloud.nix
│   ├── postgresql.nix
│   ├── redis.nix
│   └── hpb.nix        # Signaling, NATS, Janus, coturn
└── hosts/             # ce qui change d'une VM à l'autre (IP, disques…)
```

Le `flake.nix` reste très lisible : une ligne par machine.

```nix
{
  inputs.nixpkgs.url = "github:NixOS/nixpkgs/nixos-25.11"; # ou la dernière stable

  outputs = { nixpkgs, ... }:
    let
      machine = modules: nixpkgs.lib.nixosSystem {
        system = "x86_64-linux";
        modules = [ ./modules/commun.nix ] ++ modules;
      };
    in {
      nixosConfigurations = {
        haproxy-a = machine [ ./hosts/haproxy-a.nix ./modules/haproxy.nix ];
        nginx-a   = machine [ ./hosts/nginx-a.nix   ./modules/nginx.nix ];
        nc-web-01 = machine [ ./hosts/nc-web-01.nix ./modules/nextcloud.nix ];
        nc-web-02 = machine [ ./hosts/nc-web-02.nix ./modules/nextcloud.nix
                              { systemd.timers.nextcloud-cron.enable = false; } ];
        pg-a      = machine [ ./hosts/pg-a.nix      ./modules/postgresql.nix ];
        redis-01  = machine [ ./hosts/redis-01.nix  ./modules/redis.nix ];
        hpb-a     = machine [ ./hosts/hpb-a.nix     ./modules/hpb.nix ];
        # … et les VM du site B sur le même modèle
      };
    };
}
```

Regarde `nc-web-02` : la règle « **un seul cron Nextcloud actif** » tient en une ligne.
Le module Nextcloud crée un timer systemd pour le cron ; on le coupe sur le second nœud.

## Le pare-feu : tout fermé par défaut

Le module commun pose les bases, et chaque rôle n'ouvre que ce dont il a besoin.
Sur NixOS, le pare-feu est actif par défaut :

```nix
# modules/commun.nix
{
  # tout ce qui n'est pas listé est refusé
  networking.firewall.enable = true;
  services.openssh = {
    enable = true;
    settings.PasswordAuthentication = false;
  };
  services.timesyncd.enable = true;
}
```

## Nextcloud : une base distante, un Redis distant

Le module NixOS de Nextcloud installe PHP-FPM et un NGINX local. On lui dit simplement
où sont la base, Redis et le stockage partagé :

```nix
# modules/nextcloud.nix
{ pkgs, ... }:
{
  services.nextcloud = {
    enable = true;
    package = pkgs.nextcloud32;            # à adapter à la version de nixpkgs
    hostName = "cloud.exemple.fr";
    https = true;
    datadir = "/mnt/nextcloud";            # le stockage partagé

    config = {
      dbtype = "pgsql";
      # un nom logique, jamais le nom d'un nœud
      dbhost = "postgresql.service.exemple.lan";
      dbname = "nextcloud";
      dbuser = "nextcloud";
      dbpassFile = "/run/secrets/nextcloud-db";
      adminpassFile = "/run/secrets/nextcloud-admin";
    };

    settings = {
      trusted_proxies = [ "10.0.1.10" "10.0.1.11" ];   # les NGINX frontaux
      overwriteprotocol = "https";
      "memcache.locking" = "\\OC\\Memcache\\Redis";
      "memcache.distributed" = "\\OC\\Memcache\\Redis";
      redis = { host = "10.0.3.20"; port = 6379; };
    };
    # mot de passe Redis, gardé hors du store Nix
    secretFile = "/run/secrets/nextcloud-secrets";
  };

  fileSystems."/mnt/nextcloud" = {
    device = "stockage.exemple.lan:/nextcloud";
    fsType = "nfs";
  };

  networking.firewall.allowedTCPPorts = [ 80 ];   # joignable par les NGINX frontaux
}
```

## Le frontal : HAProxy, puis NGINX

Chaque site garde son HAProxy et son NGINX, avec des flux croisés entre les deux sites.

```nix
# modules/nginx.nix
{
  services.nginx = {
    enable = true;
    upstreams.nextcloud.servers = {
      "10.0.2.11:80" = { };   # nc-web-01
      "10.0.2.12:80" = { };   # nc-web-02
    };
    virtualHosts."cloud.exemple.fr" = {
      forceSSL = true;
      sslCertificate = "/run/secrets/tls/cloud.crt";
      sslCertificateKey = "/run/secrets/tls/cloud.key";
      locations."/".proxyPass = "http://nextcloud";
      locations."/standalone-signaling/" = {
        proxyPass = "http://10.0.4.10:8080/";   # Signaling sur la VM HPB
        proxyWebsockets = true;
      };
    };
  };
  networking.firewall.allowedTCPPorts = [ 443 ];
}
```

HAProxy se déclare de la même façon avec `services.haproxy`, en donnant sa configuration
en texte. Rien de spécifique à Nix ici.

## PostgreSQL et Redis

```nix
# modules/postgresql.nix
{ pkgs, ... }:
{
  services.postgresql = {
    enable = true;
    package = pkgs.postgresql_16;
    ensureDatabases = [ "nextcloud" ];
    ensureUsers = [ { name = "nextcloud"; ensureDBOwnership = true; } ];
    enableTCPIP = true;   # filtré par pg_hba et le pare-feu
    authentication = ''
      host nextcloud nextcloud 10.0.2.0/24 scram-sha-256
    '';
  };
  networking.firewall.allowedTCPPorts = [ 5432 ];
}
```

La bascule entre primaire et secondaire reste le sujet délicat, quel que soit l'outil.
nixpkgs propose un module `services.patroni` ; c'est la piste qu'on teste, avec un nom
DNS logique devant. L'objectif ne change pas : **Nextcloud ne connaît jamais le nom d'un nœud**.

```nix
# modules/redis.nix
{
  services.redis.servers.nextcloud = {
    enable = true;
    bind = "10.0.3.20";
    port = 6379;
    requirePassFile = "/run/secrets/redis";
  };
  networking.firewall.allowedTCPPorts = [ 6379 ];
}
```

## Le HPB de Talk

Sur la VM HPB, coturn et NATS ont leur module :

```nix
# modules/hpb.nix (extrait)
{
  services.coturn = {
    enable = true;
    realm = "turn.exemple.fr";
    use-auth-secret = true;
    static-auth-secret-file = "/run/secrets/turn";
    listening-port = 3478;
    tls-listening-port = 5349;
    min-port = 20000;
    max-port = 40000;
  };

  services.nats = {
    enable = true;
    # 4222 n'est pas ouvert dans le pare-feu :
    # NATS reste local à la VM
    port = 4222;
  };

  networking.firewall = {
    allowedTCPPorts = [ 3478 5349 8080 ];
    allowedUDPPorts = [ 3478 ];
    allowedUDPPortRanges = [ { from = 20000; to = 40000; } ];
  };
}
```

Janus et le serveur de signalisation demandent un peu plus de travail : selon la version
de nixpkgs, il faut écrire leur service systemd soi-même, voire empaqueter la signalisation
(c'est un projet Go). C'est la partie la moins « clé en main » de l'essai.

## Déployer

Une VM se déploie depuis le poste d'administration, en une commande :

```bash
nixos-rebuild switch --flake .#nc-web-01 --target-host root@nc-web-01
```

Pour toute la flotte, des outils comme **Colmena** ou **deploy-rs** déploient plusieurs
machines d'un coup à partir du même flake.

## Les pièges rencontrés

- **Les secrets ne vont jamais dans le code Nix.** Tout ce qui est dans le *store* Nix
  est lisible par tous les utilisateurs de la machine. D'où les `…File` et `secretFile`
  partout, alimentés par un outil comme **sops-nix** ou **agenix**.
- **Les mises à jour de Nextcloud se font une version majeure à la fois.** On ne saute
  pas de version : on change `package`, on déploie, on vérifie, puis on recommence.
- **La courbe d'apprentissage est réelle.** Le langage Nix déroute au début. Commencer
  par une seule VM (Redis, par exemple) aide beaucoup.
- **Tout n'est pas empaqueté.** La chaîne Talk (Janus, signalisation) demande plus
  d'effort que le reste.

## Premier bilan

Ce qui plaît le plus : **le dépôt Git devient la documentation de la plateforme**. Pour
savoir ce qui tourne sur une VM, on lit un fichier. Pour reconstruire un site entier,
on rejoue le flake. Et pour annuler une erreur, on revient à la génération précédente.

On continue les tests, surtout sur la bascule PostgreSQL et le HPB. Je ferai un retour
quand ce sera éprouvé. Si tu fais déjà tourner Nextcloud ou Talk sur NixOS, ton retour
m'intéresse beaucoup : les commentaires sont juste en dessous.
