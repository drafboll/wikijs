---
title: Stack Multimédia (VPN, Deluge, Jackett, FlareSolverr)
description: 
published: 1
date: 2026-09-25T15:44:08.730Z
tags: 
editor: markdown
dateCreated: 2026-05-25T11:20:29.474Z
---

# Déploiement de la Stack Multimédia Sécurisée (VPN, Deluge, Jackett, FlareSolverr)

Ce guide détaille l'installation d'un écosystème de téléchargement entièrement routé à travers un tunnel VPN. Tu pourras accéder aux interfaces de gestion via `deluge.alexandre-faye.fr` et `jackett.alexandre-faye.fr` de manière sécurisée.
 
## Comprendre l'architecture de cette Stack
La force de ton fichier `docker-compose.yml` réside dans l'instruction `network_mode: service:vpn`. 
Au lieu de donner un accès internet direct à Deluge, Jackett et FlareSolverr, tu les obliges à utiliser la connexion réseau du conteneur VPN. 
**Résultat :** Si le VPN se déconnecte, les autres applications perdent instantanément leur accès internet (c'est un "Kill Switch" naturel). C'est pour cela que **tous les ports (8112 pour Deluge, 9117 pour Jackett...) sont ouverts sur le conteneur VPN** et non sur les applications elles-mêmes !

* **VPN** (Wireguard) : Le garde du corps. Il établit une connexion chiffrée avec ton fournisseur VPN (via le fichier wg0.conf) pour masquer ton adresse IP réelle.
* **Deluge** : C'est ton client BitTorrent. Il se charge de télécharger les fichiers (films, séries) et de les placer directement dans le dossier de Plex (`/media/alexandre/plex/`).
* **Jackett** : C'est un moteur de recherche universel. Il rassemble des dizaines de sites de torrents (trackers) en un seul endroit. (Généralement utilisé en duo avec des applications comme Sonarr ou Radarr pour automatiser les recherches).
* **FlareSolverr** : C'est un outil d'assistance pour Jackett. Beaucoup de sites de torrents sont protégés par Cloudflare (la fameuse page "Vérification que vous êtes humain"). FlareSolverr résout ces défis en arrière-plan pour permettre à Jackett de faire ses recherches sans être bloqué.

---

## Prérequis
* Un serveur Linux avec Docker et Docker Compose.
* Le serveur web Apache2.
* Les noms de domaine `deluge.alexandre-faye.fr` et `jackett.alexandre-faye.fr` pointant vers l'IP de ton serveur.
* Un abonnement chez un fournisseur VPN compatible Wireguard (pour générer les clés privées/publiques).

---

## ÉTAPE 1 : Préparation de l'environnement Docker

Création de l'arborescence des dossiers
```
    sudo mkdir -p /docker/multimedia/config/wg_confs
    cd /docker/multimedia/
```
Création de la configuration Wireguard (wg0.conf)
ATTENTION : Remplace 'ta privet cle' et 'ta publique clé' par les vraies valeurs 
fournies par ton fournisseur VPN.
```
    sudo bash -c 'cat << "EOF" > config/wg_confs/wg0.conf
    [Interface]
    PrivateKey = ta privet cle
    Address = 10.0.2.76/32
    MTU = 1420
    DNS = 9.9.9.9

    [Peer]
    PublicKey = ta publique clé
    AllowedIPs = 0.0.0.0/0
    Endpoint = 213.239.219.28:31661
    PersistentKeepalive = 21
    EOF'
```
    # 3. Création du fichier de configuration Docker Compose
```
    services:
      vpn:
        image: lscr.io/linuxserver/wireguard:latest
        container_name: vpn
        cap_add:
          - NET_ADMIN
        environment:
          - PUID=1000
          - PGID=1000
          - TZ=Europe/Paris
        volumes:
          - /docker/multimedia/config:/config
          - /lib/modules:/lib/modules
        ports:
          - 8080:8080
          - 8112:8112   # Port de Deluge
          - 6881:6881
          - 6881:6881/udp
          - 9117:9117   # Port de Jackett
          - 8191:8191   # Port de FlareSolverr
          - 7878:7878
        sysctls:
          - net.ipv4.conf.all.src_valid_mark=1
          - net.ipv6.conf.all.disable_ipv6=1
        restart: unless-stopped
      
      deluge:
        image: lscr.io/linuxserver/deluge:latest
        depends_on:
          vpn:
            condition: service_started
        network_mode: service:vpn
        container_name: deluge
        environment:
          - PUID=1000
          - PGID=1000
          - DELUGE_LOGLEVEL=error
        volumes:
          - /docker/multimedia/deluge/config:/config
          - /media/alexandre/plex/:/plex
        restart: unless-stopped

      jackett:
        image: lscr.io/linuxserver/jackett:latest
        depends_on:
          vpn:
            condition: service_started
        network_mode: service:vpn
        container_name: jackett
        environment:
          - PUID=1000
          - PGID=1000
        volumes:
          - /docker/multimedia/jackett/config:/config
          - /docker/plex/torrent:/downloads
        restart: unless-stopped

      flaresolverr:
        image: ghcr.io/flaresolverr/flaresolverr:latest
        depends_on:
          vpn:
            condition: service_started
        network_mode: service:vpn
        container_name: flaresolverr
        environment:
          - LOG_LEVEL=${LOG_LEVEL:-info}
          - LOG_HTML=${LOG_HTML:-false}
          - CAPTCHA_SOLVER=${CAPTCHA_SOLVER:-none}
          - TZ=Europe/Paris
        restart: unless-stopped 
```
---

## ÉTAPE 2 : Déploiement des conteneurs Docker

Lancement de la stack réseau et multimédia
```
    sudo docker-compose up -d
```
Vérification (Assure-toi que les 4 conteneurs sont "Up")
```
    sudo docker ps
```

---

## ÉTAPE 3 : Configuration du Reverse Proxy Apache

Nous allons créer deux fichiers de configuration Apache pour rediriger tes sous-domaines vers les bons ports locaux.

Activation des modules Apache (si pas déjà fait)
```
    sudo a2enmod proxy proxy_http rewrite
```
Configuration pour Jackett (Port 9117)
```
sudo bash -c 'cat << "EOF" > /etc/apache2/sites-available/jackett.conf
    <VirtualHost *:80>
        ServerName jackett.alexandre-faye.fr

        ProxyPreserveHost On
        ProxyPass / http://127.0.0.1:9117/
        ProxyPassReverse / http://127.0.0.1:9117/
        
        RewriteEngine on
        RewriteCond %{SERVER_NAME} =jackett.alexandre-faye.fr
        RewriteRule ^ https://%{SERVER_NAME}%{REQUEST_URI} [END,NE,R=permanent]
    </VirtualHost>
    EOF'
```

Configuration pour Deluge (Port 8112)
```
    sudo vi /etc/apache2/sites-available/deluge.conf
```
```
    <VirtualHost *:80>
        ServerName deluge.alexandre-faye.fr

        ProxyPreserveHost On
        ProxyPass / http://127.0.0.1:8112/
        ProxyPassReverse / http://127.0.0.1:8112/
        
        RewriteEngine on
        RewriteCond %{SERVER_NAME} =deluge.alexandre-faye.fr
        RewriteRule ^ https://%{SERVER_NAME}%{REQUEST_URI} [END,NE,R=permanent]
    </VirtualHost>
```
    # 4. Activation des sites et redémarrage d'Apache
    sudo a2ensite jackett.conf deluge.conf
    sudo systemctl restart apache2

---

## ÉTAPE 4 : Génération des certificats SSL (HTTPS)

Lancement de Certbot pour sécuriser les deux sous-domaines
```
sudo certbot --apache -d jackett.alexandre-faye.fr -d deluge.alexandre-faye.fr
```
---

## ÉTAPE 5 : Test et Validation

1. **Vérifie Deluge :** Va sur `https://deluge.alexandre-faye.fr`. Le mot de passe par défaut de l'interface web Deluge est généralement `deluge` (il te demandera de le changer à la première connexion).
2. **Vérifie Jackett :** Va sur `https://jackett.alexandre-faye.fr`. Tu pourras commencer à ajouter tes trackers.
3. **Vérifie ton adresse IP VPN (Optionnel mais recommandé) :** Pour être sûr que tes téléchargements passent bien par le VPN, tu peux exécuter cette commande qui va interroger l'IP publique vue par le conteneur Deluge :
   `sudo docker exec -it deluge curl ifconfig.me`
   *(L'adresse IP qui s'affiche doit être celle de ton VPN, pas celle de ton serveur !)*