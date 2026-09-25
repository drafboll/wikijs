---
title: Stirling-PDF avec Docker
description: 
published: 1
date: 2026-09-25T15:44:24.460Z
tags: 
editor: markdown
dateCreated: 2026-05-25T10:59:37.928Z
---

# Déploiement de Stirling-PDF avec Docker, Apache2 et SSL

Ce guide regroupe l'intégralité des instructions sur une seule page pour vous permettre de déployer l'application Stirling-PDF de manière sécurisée.
 
---

## Prérequis
* Un serveur Linux avec Docker et Docker Compose installés.
* Le serveur web Apache2 installé.
* Le nom de domaine pdf.alexandre-faye.fr pointant vers l'adresse IP publique de votre serveur.

---

## ÉTAPE 1 : Préparation de l'environnement Docker

    # 1. Création des répertoires
    sudo mkdir -p /docker/stirling-pdf/trainingData
    cd /docker/stirling-pdf/

    # 2. Création du fichier .env (Remplacez 'votre_mot_de_passe')
    sudo bash -c 'cat << EOF > .env
    STIRLING_USER=alexandre
    STIRLING_PWD=votre_mot_de_passe
    EOF'

    # 3. Écriture du fichier docker-compose.yml
    sudo bash -c 'cat << EOF > docker-compose.yml
    version: "3.3"
    services:
      stirling-pdf:
        image: frooodle/s-pdf:latest
        container_name: stirling-pdf
        ports:
          - "8083:8080"
        volumes:
          - /docker/stirling-pdf/trainingData:/usr/share/tesseract-ocr/4.00/tessdata 
        environment:
          - DOCKER_ENABLE_SECURITY=true
          - SECURITY_ENABLE_LOGIN=true
          - SECURITY_INITIALLOGIN_USERNAME=\${STIRLING_USER}
          - SECURITY_INITIALLOGIN_PASSWORD=\${STIRLING_PWD}
          - INSTALL_BOOK_AND_ADVANCED_HTML_OPS=false
          - LANGS=fr_FR
        restart: unless-stopped
    EOF'

---

## ÉTAPE 2 : Déploiement du conteneur Docker

    # Lancement de l'application en arrière-plan
    sudo docker-compose up -d

    # Vérification du statut du conteneur
    sudo docker ps

---

## ÉTAPE 3 : Configuration du Reverse Proxy Apache

    # 1. Activation des modules Apache
    sudo a2enmod proxy proxy_http rewrite

    # 2. Écriture de la configuration du VirtualHost
    sudo bash -c 'cat << EOF > /etc/apache2/sites-available/pdf.conf
    <VirtualHost *:80>
        ServerName pdf.alexandre-faye.fr

        ProxyPreserveHost On
        ProxyPass / http://127.0.0.1:8083/
        ProxyPassReverse / http://127.0.0.1:8083/

        RewriteEngine on
        RewriteCond %{SERVER_NAME} =pdf.alexandre-faye.fr
        RewriteRule ^ https://%{SERVER_NAME}%{REQUEST_URI} [END,NE,R=permanent]
    </VirtualHost>
    EOF'

    # 3. Activation du site et redémarrage
    sudo a2ensite pdf.conf
    sudo systemctl restart apache2

---

## ÉTAPE 4 : Génération et installation du certificat SSL (HTTPS)

    # 1. Installation de Certbot
    sudo apt update
    sudo apt install certbot python3-certbot-apache -y

    # 2. Demande du certificat (suivez les instructions à l'écran)
    sudo certbot --apache -d pdf.alexandre-faye.fr

---

## ÉTAPE 5 : Validation du déploiement
![capture_d'écran_2026-05-25_130102.png](/docker/capture_d'écran_2026-05-25_130102.png)
1. Ouvrez votre navigateur et accédez à : http://pdf.alexandre-faye.fr
2. Vérifiez la redirection vers HTTPS (présence du cadenas).
3. Connectez-vous avec l'identifiant "alexandre" et le mot de passe configuré.