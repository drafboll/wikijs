---
title: Erugo avec Docker
description: 
published: 1
date: 2026-09-25T15:42:50.959Z
tags: 
editor: markdown
dateCreated: 2026-05-25T11:10:00.645Z
---

# Tutoriel : Déploiement et Mise à jour d'Erugo avec Docker, Apache2 et SSL

Ce guide détaille l'installation complète d'Erugo sur ton serveur. Il est conçu pour que tu puisses déployer l'application, l'exposer sur le web via `erugo.alexandre-faye.fr` et la sécuriser.
 
## À quoi sert Erugo ?
https://erugo.app/
Erugo est une application open-source de type "Link-in-bio", qui sert d'alternative auto-hébergée à des services très connus comme **Linktree**. Son but est simple : te permettre de créer une page web unique et personnalisée (une "landing page") qui regroupe tous tes liens importants (tes réseaux sociaux, ton site web, ton portfolio, tes projets, etc.). Tu peux ensuite partager cette URL unique sur tes profils (Instagram, TikTok, Twitter, etc.) pour rediriger facilement ton audience vers tout ton contenu.

---

## Prérequis
* Un serveur Linux avec Docker et Docker Compose installés.
* Le serveur web Apache2 installé.
* Le nom de domaine `erugo.alexandre-faye.fr` pointant vers l'adresse IP de ton serveur.

---

## ÉTAPE 1 : Préparation de l'environnement Docker

Dans cette étape, nous allons créer le dossier qui va accueillir ton application, ainsi que le fichier de configuration de Docker (`docker-compose.yml`). Ce fichier indique à Docker quelle image télécharger (wardy784/erugo), sur quel port l'exposer (9998) et où sauvegarder tes données pour ne rien perdre.

    # 1. Création du répertoire de travail pour l'application
    sudo mkdir -p /docker/erugo
    cd /docker/erugo/

    # 2. Création du fichier de configuration Docker Compose
    sudo bash -c 'cat << "EOF" > docker-compose.yml
    services:
      erugo:
        image: wardy784/erugo:latest
        container_name: erugo
        restart: unless-stopped
        volumes:
          - ./erugo:/var/www/html/storage
        ports:
          - "127.0.0.1:9998:80"  # On lie le port 80 du conteneur au 9998 de l'hôte
        networks:
          - erugo-net

    networks:
      erugo-net:
    EOF'

---

## ÉTAPE 2 : Déploiement du conteneur Docker

Maintenant que la configuration est prête, on demande à Docker de télécharger les fichiers nécessaires et de démarrer l'application Erugo en tâche de fond.

    # Lancement de l'application en arrière-plan
    sudo docker-compose up -d

    # Vérification que le conteneur tourne bien sans erreur
    sudo docker ps

---

## ÉTAPE 3 : Configuration du Reverse Proxy Apache

Erugo tourne maintenant sur le port 9998 de ton serveur local. Pour qu'il soit accessible depuis l'extérieur, nous allons configurer Apache pour qu'il intercepte les visiteurs tapant `erugo.alexandre-faye.fr` sur le port standard (80) et les redirige de manière invisible vers ton conteneur.

    # 1. Activation des modules Apache nécessaires pour faire le "pont"
    sudo a2enmod proxy proxy_http rewrite

    # 2. Création du fichier de configuration pour ton domaine (VirtualHost)
    sudo bash -c 'cat << "EOF" > /etc/apache2/sites-available/erugo.conf
    <VirtualHost *:80>
        ServerName erugo.alexandre-faye.fr

        ProxyPreserveHost On
        ProxyPass / http://127.0.0.1:9998/
        ProxyPassReverse / http://127.0.0.1:9998/

        RewriteEngine on
        RewriteCond %{SERVER_NAME} =erugo.alexandre-faye.fr
        RewriteRule ^ https://%{SERVER_NAME}%{REQUEST_URI} [END,NE,R=permanent]
    </VirtualHost>
    EOF'

    # 3. Activation du site dans Apache et redémarrage du service
    sudo a2ensite erugo.conf
    sudo systemctl restart apache2

---

## ÉTAPE 4 : Sécurisation avec un certificat SSL (HTTPS)

Cette étape est cruciale : elle permet d'ajouter le fameux "cadenas" dans le navigateur en chiffrant les échanges. Certbot va lire la configuration Apache que tu viens de faire et l'adapter automatiquement pour le HTTPS (port 443).

    # 1. Installation de Certbot (si tu ne l'as pas déjà fait pour un autre site)
    sudo apt update
    sudo apt install certbot python3-certbot-apache -y

    # 2. Génération du certificat (suis les instructions à l'écran)
    sudo certbot --apache -d erugo.alexandre-faye.fr

---

## ÉTAPE 5 : Test et Validation

![capture_d'écran_2026-05-25_131119.png](/docker/capture_d'écran_2026-05-25_131119.png)
1. Ouvre ton navigateur et tape : http://erugo.alexandre-faye.fr
2. Tu vas être automatiquement redirigé vers la version sécurisée (HTTPS).
3. La page d'accueil d'Erugo s'affiche : tu peux maintenant créer ton compte et commencer à paramétrer ta page de liens !

---

## COMMENT METTRE À JOUR ERUGO

Puisque ton fichier de configuration utilise l'étiquette `latest` (image: wardy784/erugo:latest), la mise à jour est très simple. Docker va vérifier s'il existe une version plus récente, la télécharger, et recréer le conteneur en gardant tes données intactes.

    # 1. Place-toi dans le dossier de l'application
    cd /docker/erugo/

    # 2. Télécharge la dernière version de l'image Erugo
    sudo docker-compose pull

    # 3. Relance le conteneur avec la nouvelle image (il supprimera l'ancien et démarrera le nouveau)
    sudo docker-compose up -d

    # 4. Optionnel : Nettoie les anciennes images Docker devenues inutiles pour libérer de l'espace
    sudo docker image prune -f