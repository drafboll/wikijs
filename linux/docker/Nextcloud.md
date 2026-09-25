---
title: Nextcloud avec Docker
description: 
published: 1
date: 2026-09-25T15:43:33.016Z
tags: 
editor: markdown
dateCreated: 2026-05-25T11:04:50.165Z
---

# Déploiement et Mise à jour de Nextcloud avec Docker

Ce guide regroupe l'intégralité des instructions sur une seule page pour déployer ton instance Nextcloud, la rendre accessible via `cloud.alexandre-faye.fr`, la sécuriser avec HTTPS, et la mettre à jour facilement.

--- 

## Prérequis
* Un serveur Linux avec Docker et Docker Compose installés.
* Le serveur web Apache2 installé.
* Le nom de domaine cloud.alexandre-faye.fr pointant vers l'adresse IP de ton serveur.

---

## ÉTAPE 1 : Préparation de l'environnement Docker

    # 1. Création du répertoire principal
    sudo mkdir -p /docker/nextcloud
    cd /docker/nextcloud/

    # 2. Écriture du fichier docker-compose.yml
    # (Tes mots de passe "mdp" sont inclus, pense à les changer si ce serveur est public !)
    sudo bash -c 'cat << "EOF" > docker-compose.yml
    services:
      db:
        image: mariadb:11.5.2
        restart: always
        command: --transaction-isolation=READ-COMMITTED --binlog-format=ROW
        environment:
          - MYSQL_ROOT_PASSWORD=mdp
          - MYSQL_PASSWORD=mdp 
          - MYSQL_DATABASE=nextcloud
          - MYSQL_USER=nextcloud
        volumes:
          - db_data:/var/lib/mysql

      redis:
        image: redis:alpine
        restart: always

      app:
        image: nextcloud:32.0.6
        restart: always
        ports:
          - 8082:80
        volumes:
          - nextcloud_data:/var/www/html
          - ./app/config:/var/www/html/config
          - ./app/custom_apps:/var/www/html/custom_apps
          - ./app/data:/var/www/html/data
          - ./app/themes:/var/www/html/themes
          - /etc/localtime:/etc/localtime:ro
        environment:
          - MYSQL_PASSWORD=mdp
          - MYSQL_DATABASE=nextcloud
          - MYSQL_USER=nextcloud
          - MYSQL_HOST=db
        depends_on:
          - db

    volumes:
      db_data:
      nextcloud_data:
    EOF'

---

## ÉTAPE 2 : Déploiement du conteneur Docker

    # Lancement de la base de données, redis et Nextcloud
    sudo docker-compose up -d

    # Vérification du statut des conteneurs
    sudo docker ps

---

## ÉTAPE 3 : Configuration du Reverse Proxy Apache

    # 1. Activation des modules Apache (si pas déjà fait)
    sudo a2enmod proxy proxy_http rewrite

    # 2. Écriture de la configuration du VirtualHost
    sudo bash -c 'cat << "EOF" > /etc/apache2/sites-available/cloud.conf
    <VirtualHost *:80>
        ServerName cloud.alexandre-faye.fr

        ProxyPreserveHost On
        ProxyPass / http://127.0.0.1:8082/
        ProxyPassReverse / http://127.0.0.1:8082/

        RewriteEngine on
        RewriteCond %{SERVER_NAME} =cloud.alexandre-faye.fr
        RewriteRule ^ https://%{SERVER_NAME}%{REQUEST_URI} [END,NE,R=permanent]
    </VirtualHost>
    EOF'

    # 3. Activation du site et redémarrage
    sudo a2ensite cloud.conf
    sudo systemctl restart apache2

---

## ÉTAPE 4 : Génération et installation du certificat SSL (HTTPS)

    # 1. Installation de Certbot (si pas déjà fait)
    sudo apt update
    sudo apt install certbot python3-certbot-apache -y

    # 2. Demande du certificat pour le cloud
    sudo certbot --apache -d cloud.alexandre-faye.fr

---

## ÉTAPE 5 : Validation du déploiement
![capture_d'écran_2026-05-25_130417.png](/docker/capture_d'écran_2026-05-25_130417.png)
1. Ouvre ton navigateur et accédez à : http://cloud.alexandre-faye.fr
2. Vérifie que tu es bien redirigé vers la version HTTPS.
3. Termine l'installation de Nextcloud depuis l'interface web en créant ton compte administrateur.

---

## COMMENT METTRE À JOUR NEXTCLOUD

Puisque tu utilises des versions fixes dans ton docker-compose (ex: nextcloud:32.0.6 et mariadb:11.5.2), la mise à jour se fait en changeant le numéro de version.

    # 1. Va dans le dossier de Nextcloud
    cd /docker/nextcloud/

    # 2. Modifie le fichier docker-compose.yml pour changer la version
    # (par exemple, remplace "image: nextcloud:32.0.6" par la nouvelle version)
    sudo nano docker-compose.yml

    # 3. Télécharge la nouvelle image
    sudo docker-compose pull

    # 4. Redémarre les conteneurs avec la nouvelle image
    sudo docker-compose up -d

    # 5. Lance la mise à jour de la base de données Nextcloud (très important !)
    sudo docker exec -u www-data -it nom_du_conteneur_app php occ upgrade

Note : Pour l'étape 5, remplace "nom_du_conteneur_app" par le vrai nom du conteneur Nextcloud (tu peux le trouver avec la commande `sudo docker ps`).