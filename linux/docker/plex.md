---
title: Plex avec Docker
description: 
published: 1
date: 2026-09-25T15:43:44.217Z
tags: 
editor: markdown
dateCreated: 2026-05-25T11:13:58.736Z
---

# Déploiement et Mise à jour de Plex avec Docker, Apache2 et SSL

Ce guide détaillé te permet d'installer ton propre serveur Plex, de le rendre accessible proprement via `plex.alexandre-faye.fr` et de sécuriser la connexion.
 
## À quoi sert Plex ?
https://watch.plex.tv/fr
Plex est un véritable "Netflix personnel". C'est une application de serveur multimédia qui scanne tes dossiers de films, séries, animes ou musiques (ici situés dans `/media/alexandre/plex`), télécharge automatiquement les affiches, les résumés et les notes, puis te permet de les lire en streaming depuis n'importe quel appareil (TV connectée, smartphone, navigateur web, tablette), que tu sois chez toi ou à l'extérieur.

---

## Prérequis
* Un serveur Linux avec Docker et Docker Compose.
* Le serveur web Apache2.
* Le nom de domaine `plex.alexandre-faye.fr` pointant vers l'adresse IP de ton serveur.
* Tes médias (fichiers vidéos) déjà présents dans le dossier `/media/alexandre/plex`.

---

## ÉTAPE 1 : Préparation de l'environnement Docker

Ta configuration utilise `network_mode: host`. C'est la méthode recommandée pour Plex car elle lui permet de détecter facilement les appareils sur ton réseau local (comme les Smart TV ou le Chromecast) sans être bloqué par le réseau virtuel de Docker. 

*Note importante concernant `PLEX_CLAIM` : Le jeton de réclamation que tu as mis (claim-...) sert à lier automatiquement ton serveur à ton compte Plex. Ces jetons expirent en 4 minutes. Si ton déploiement échoue à se lier à ton compte, tu devras générer un nouveau jeton sur https://plex.tv/claim et le remplacer dans le fichier.*

    # 1. Création du dossier pour les fichiers de configuration de Plex
    sudo mkdir -p /docker/plex/config
    cd /docker/plex/

    # 2. Création du fichier de configuration Docker Compose
    sudo bash -c 'cat << "EOF" > docker-compose.yml
    ---
    version: "2.1"
    services:
      plex:
        image: lscr.io/linuxserver/plex:latest
        container_name: plex
        network_mode: host
        environment:
          - PUID=1000
          - PGID=1000
          - VERSION=docker
          - PLEX_CLAIM=claim-ZtNVo2czJQ9szG5vdyQv
        volumes:
          - /docker/plex/config:/config
          - /media/alexandre/plex:/anime
          - /media/alexandre/plex:/movies
          - /media/alexandre/plex:/series
        restart: unless-stopped
    EOF'

---

## ÉTAPE 2 : Déploiement du conteneur Docker

    # Lancement de Plex en arrière-plan
    sudo docker-compose up -d

    # Vérification que le conteneur tourne bien
    sudo docker ps

---

## ÉTAPE 3 : Configuration du Reverse Proxy Apache

Par défaut, l'interface web de Plex est accessible sur le port `32400` et nécessite qu'on ajoute `/web` à la fin de l'URL. Ta configuration Apache est très intelligente : elle s'occupe de rediriger le trafic vers le port 32400 ET ajoute automatiquement le `/web` si l'utilisateur l'oublie, grâce aux règles "RewriteCond".

    # 1. Activation des modules Apache (proxy et réécriture)
    sudo a2enmod proxy proxy_http rewrite

    # 2. Création du fichier VirtualHost pour Plex
    sudo bash -c 'cat << "EOF" > /etc/apache2/sites-available/plex.conf
    <VirtualHost *:80>
      ServerName plex.alexandre-faye.fr

      <Proxy>
        Order deny,allow
        Allow from all
      </Proxy>

      ProxyRequests Off
      ProxyPreserveHost On
      ProxyPass / http://127.0.0.1:32400/
      ProxyPassReverse / http://127.0.0.1:32400/

    # Ces règles ajoutent automatiquement /web à l'URL et forcent le HTTPS
    RewriteEngine on
    RewriteCond %{REQUEST_URI} !^/web
    RewriteCond %{HTTP:X-Plex-Device} ^$
    RewriteRule ^/$ /web/$1 [R,L]
    RewriteCond %{SERVER_NAME} =plex.alexandre-faye.fr
    RewriteRule ^ https://%{SERVER_NAME}%{REQUEST_URI} [END,NE,R=permanent]
    </VirtualHost>
    EOF'

    # 3. Activation du site et redémarrage d'Apache
    sudo a2ensite plex.conf
    sudo systemctl restart apache2

---

## ÉTAPE 4 : Sécurisation avec un certificat SSL (HTTPS)

Plex gère en partie sa propre sécurité, mais passer par ton reverse proxy Apache avec Let's Encrypt garantit une connexion 100% sécurisée quand tu y accèdes depuis l'extérieur via ton nom de domaine personnalisé.

    # 1. Installation de Certbot (si nécessaire)
    sudo apt update
    sudo apt install certbot python3-certbot-apache -y

    # 2. Génération du certificat HTTPS
    sudo certbot --apache -d plex.alexandre-faye.fr

---

## ÉTAPE 5 : Test et Validation
![capture_d'écran_2026-05-25_131338.png](/docker/capture_d'écran_2026-05-25_131338.png)
1. Ouvre ton navigateur et tape : http://plex.alexandre-faye.fr
2. Magie de ta configuration Apache : tu seras redirigé vers `https://plex.alexandre-faye.fr/web` !
3. Connecte-toi avec ton compte Plex.
4. Va dans les réglages (la clé à molette) > **Réseau** (Network) et vérifie la configuration. Sous "URL personnalisées pour accéder au serveur", tu pourras ajouter `https://plex.alexandre-faye.fr:443` plus tard pour optimiser l'application mobile.

---

## COMMENT METTRE À JOUR PLEX

L'image `linuxserver/plex` est excellente car elle est conçue pour se mettre à jour très facilement. 

    # 1. Place-toi dans le dossier de Plex
    cd /docker/plex/

    # 2. Demande à Docker de télécharger la dernière version de l'image
    sudo docker-compose pull

    # 3. Relance le conteneur (Docker recréera le conteneur avec la nouvelle image)
    sudo docker-compose up -d

    # Note : Même si tu supprimes/recrées le conteneur, toutes tes affiches, 
    # réglages et historiques de lecture sont conservés dans /docker/plex/config.