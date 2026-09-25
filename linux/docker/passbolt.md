---
title: Procédure de Déploiement Passbolt CE
description: 
published: 1
date: 2026-09-25T15:43:53.763Z
tags: 
editor: markdown
dateCreated: 2026-02-14T18:30:35.944Z
---

# Procédure de Déploiement Passbolt CE

Ce document récapitule la configuration complète de ton instance Passbolt fonctionnelle, incluant Docker Compose, la gestion des secrets et le relais mail Microsoft 365.

---
 
## 1. Structure du Répertoire
L'installation est centralisée dans : `/docker/passbolt/`

### A. Fichier de variables : `.env`
Ce fichier contient toute la configuration sensible. Les mots de passe contenant des caractères spéciaux sont entourés de guillemets.

Ce fichier centralise les variables. Docker les injecte automatiquement dans les conteneurs.

Note Sécurité : Les mots de passe contenant des caractères spéciaux (#, !, ]) doivent être entourés de guillemets doubles " " pour éviter que Docker ne les interprète mal.

```bash
# --- BASE DE DONNÉES ---
DATASOURCES_DEFAULT_DATABASE=passbolt
DATASOURCES_DEFAULT_USERNAME=passboltkeypass
DATASOURCES_DEFAULT_PASSWORD="=d=(7}|y3ka!+?8][EzH"

# --- CONFIGURATION APP ---
APP_FULL_BASE_URL=[http://192.168.40.30](http://192.168.40.30)

# --- RELAIS MAIL (MICROSOFT 365) ---
EMAIL_DEFAULT_FROM=mail@mail.
EMAIL_TRANSPORT_DEFAULT_HOST=smtp.office365.com
EMAIL_TRANSPORT_DEFAULT_PORT=587
EMAIL_TRANSPORT_DEFAULT_USERNAME=mail@mail.com
EMAIL_TRANSPORT_DEFAULT_PASSWORD="mdp alexandre"
EMAIL_TRANSPORT_DEFAULT_TLS=true
```

### A. Fichier docker-compose : `docker-compose.yml`
Il définit deux services :

- db (MariaDB) : Stocke les données. Le volume database_volume garantit que les mots de passe ne sont pas perdus au redémarrage.

- passbolt (App) : Le moteur PHP/Nginx. Il dépend de la base de données.

```bash
services:
  db:
    image: mariadb:10.11
    container_name: passbolt-db
    restart: unless-stopped
    environment:
      MYSQL_DATABASE: ${DATASOURCES_DEFAULT_DATABASE}
      MYSQL_USER: ${DATASOURCES_DEFAULT_USERNAME}
      MYSQL_PASSWORD: ${DATASOURCES_DEFAULT_PASSWORD}
      MYSQL_RANDOM_ROOT_PASSWORD: "true"
    volumes:
      - database_volume:/var/lib/mysql
    networks:
      - passbolt-network

  passbolt:
    image: passbolt/passbolt:latest-ce
    container_name: passbolt-app
    restart: unless-stopped
    depends_on:
      - db
    environment:
      APP_FULL_BASE_URL: ${APP_FULL_BASE_URL}
      DATASOURCES_DEFAULT_HOST: "db"
      DATASOURCES_DEFAULT_USERNAME: ${DATASOURCES_DEFAULT_USERNAME}
      DATASOURCES_DEFAULT_PASSWORD: ${DATASOURCES_DEFAULT_PASSWORD}
      DATASOURCES_DEFAULT_DATABASE: ${DATASOURCES_DEFAULT_DATABASE}
      PASSBOLT_SSL_FORCE: "false"
      EMAIL_DEFAULT_FROM: ${EMAIL_DEFAULT_FROM}
      EMAIL_TRANSPORT_DEFAULT_HOST: ${EMAIL_TRANSPORT_DEFAULT_HOST}
      EMAIL_TRANSPORT_DEFAULT_PORT: ${EMAIL_TRANSPORT_DEFAULT_PORT}
      EMAIL_TRANSPORT_DEFAULT_USERNAME: ${EMAIL_TRANSPORT_DEFAULT_USERNAME}
      EMAIL_TRANSPORT_DEFAULT_PASSWORD: ${EMAIL_TRANSPORT_DEFAULT_PASSWORD}
      EMAIL_TRANSPORT_DEFAULT_TLS: "true"
    volumes:
      - gpg_volume:/etc/passbolt/gpg
      - jwt_volume:/etc/passbolt/jwt
    ports:
      - "80:80"
    networks:
      - passbolt-network

networks:
  passbolt-network:
    driver: bridge

volumes:
  database_volume:
  gpg_volume:
  jwt_volume:
```
  
## Démarrage des conteneurs
On démarre les conteneurs en arrière-plan (-d).
```bash
sudo docker compose up -d
```
![capture_d'écran_2026-02-14_193129.png](/passbolt/capture_d'écran_2026-02-14_193129.png)
## Génération de l'invitation

Génère le premier utilisateur et affiche un lien unique dans le terminal.

```bash
sudo docker exec passbolt-app su -m -s /bin/bash -c "./bin/cake passbolt register_user -u afaye42@myges.fr -f Alexandre -l Faye -r admin" www-data
```
![capture_d'écran_2026-02-14_193117.png](/passbolt/capture_d'écran_2026-02-14_193117.png)


# Phase 2 : Initialisation du compte et Sécurité Client

Une fois que le serveur est opérationnel, la configuration se déplace vers le navigateur pour créer ton identité numérique sécurisée (clés GPG).

## 1. Finalisation de l'accès (Invitation)
![capture_d'écran_2026-02-14_192053.png](/passbolt/capture_d'écran_2026-02-14_192053.png)
Si tu tentes d'accéder directement à la page de connexion, le service affichera un message de sécurité : "L'accès à ce service requiert une invitation".

![capture_d'écran_2026-02-14_192133.png](/passbolt/capture_d'écran_2026-02-14_192133.png)
Pourquoi ? Passbolt est "Private by Design". Aucun compte ne peut être créé sans une invitation générée par le serveur.

Action : Utilise impérativement le lien généré dans ton terminal lors de la commande register_user.

## 2. Configuration de l'extension Navigateur
Passbolt ne stocke jamais ton mot de passe maître sur le serveur. Tout le chiffrement se fait localement via une extension.

Clique sur le lien d'invitation.

Installe l'extension Passbolt pour Chrome/Brave/Firefox.

Une fois installée, l'extension détectera automatiquement ton serveur 192.168.40.30.

![capture_d'écran_2026-02-14_192506.png](/passbolt/capture_d'écran_2026-02-14_192506.png)

## 3. Création de la Phrase de Passe (Master Password)
C'est l'étape la plus critique de ta sécurité.

Rôle : Cette phrase verrouille et déverrouille ta clé privée locale.
![capture_d'écran_2026-02-14_192517.png](/passbolt/capture_d'écran_2026-02-14_192517.png)
Conseil : Choisis une phrase longue (plus de 12 caractères) mais mémorisable. Si tu l'oublies, personne (ni même l'admin) ne pourra récupérer tes mots de passe.

## 4. Accès au Coffre-fort
Une fois ces étapes terminées, tu arrives sur ton tableau de bord.

Tu peux maintenant créer tes premiers mots de passe en cliquant sur le bouton "Créer".
![capture_d'écran_2026-02-14_192628.png](/passbolt/capture_d'écran_2026-02-14_192628.png)
Ton interface est désormais prête à accueillir tes secrets de manière centralisée et sécurisée.


# Phase 3 : Sécurisation HTTPS (SSL/TLS) via ADCS
Pour garantir la confidentialité des secrets et supprimer les alertes de sécurité, l'instance a été migrée de HTTP vers HTTPS en utilisant l'Autorité de Certification de l'entreprise.

## 1. Préparation des Certificats
Les certificats sont stockés sur le serveur Linux dans /docker/passbolt/certs/.

A. Génération de la demande (CSR)
Une clé privée et une demande de signature ont été générées via OpenSSL :
Pour éviter les alertes de sécurité, le certificat doit impérativement contenir l'extension Subject Alternative Name (SAN).

```
openssl req -new -newkey rsa:2048 -nodes -keyout passbolt.key -out passbolt.csr \
  -subj "/CN=passbolt.indiogroup.lan" \
  -addext "subjectAltName = DNS:passbolt.indiogroup.lan"
```
![capture_d'écran_2026-02-14_221103.png](/passbolt/capture_d'écran_2026-02-14_221103.png)

B. Signature par l'ADCS Windows
La CSR a été signée par le serveur INDIO-GROUP-ROOTCA en utilisant le modèle Indio-ServeurWebLinux.
![capture_d'écran_2026-02-14_195038.png](/passbolt/capture_d'écran_2026-02-14_195038.png)
Certificat Serveur : passbolt.crt
la ligne de commande en powershell : 
```
certreq -submit -attrib "CertificateTemplate:Indio-ServeurWebLinux" C:\Users\administrateur.INDIOGROUP\Downloads\passbolt.csr
```
Certificat Racine (CA) : ca.crt (Indispensable pour la chaîne de confiance).

## 2. Configuration DNS
Pour que le certificat soit valide, l'accès ne doit plus se faire par IP mais par le FQDN (Nom de domaine).

Serveur DNS : Windows Domain Controller.

Enregistrement : Type A | Nom : passbolt | Cible : 192.168.40.30.
![capture_d'écran_2026-02-14_221310.png](/passbolt/capture_d'écran_2026-02-14_221310.png)
Vérification : ping passbolt.indiogroup.lan doit répondre sur l'IP du Docker.

## 3. Mise à jour de la configuration Docker
Le fichier docker-compose.yml a été modifié pour activer le port 443 et monter les certificats.

Modification du .env

```
APP_FULL_BASE_URL=https://passbolt.indiogroup.lan
```
Modification du docker-compose.yml

```
environment:
      APP_FULL_BASE_URL: ${APP_FULL_BASE_URL}
      DATASOURCES_DEFAULT_HOST: "db"
      DATASOURCES_DEFAULT_USERNAME: ${DATASOURCES_DEFAULT_USERNAME}
      DATASOURCES_DEFAULT_PASSWORD: ${DATASOURCES_DEFAULT_PASSWORD}
      DATASOURCES_DEFAULT_DATABASE: ${DATASOURCES_DEFAULT_DATABASE}
      PASSBOLT_SSL_FORCE: "true"
      PASSBOLT_SSL_CERT_FILE: "/etc/ssl/certs/certificate.crt"
      PASSBOLT_SSL_KEY_FILE: "/etc/ssl/certs/certificate.key"
      # Configuration mail via .env
      EMAIL_DEFAULT_FROM: ${EMAIL_DEFAULT_FROM}
      EMAIL_TRANSPORT_DEFAULT_HOST: ${EMAIL_TRANSPORT_DEFAULT_HOST}
      EMAIL_TRANSPORT_DEFAULT_PORT: ${EMAIL_TRANSPORT_DEFAULT_PORT}
      EMAIL_TRANSPORT_DEFAULT_USERNAME: ${EMAIL_TRANSPORT_DEFAULT_USERNAME}
      EMAIL_TRANSPORT_DEFAULT_PASSWORD: ${EMAIL_TRANSPORT_DEFAULT_PASSWORD}
      EMAIL_TRANSPORT_DEFAULT_TLS: "true"
    volumes:
      - gpg_volume:/etc/passbolt/gpg
      - jwt_volume:/etc/passbolt/jwt
      - ./certs/passbolt.crt:/etc/ssl/certs/certificate.crt:ro
      - ./certs/passbolt.key:/etc/ssl/certs/certificate.key:ro
    ports:
      - "80:80"
      - "443:443"
```

## 4. Validation de la Confiance Client
Même si le serveur est bien configuré, le navigateur affichera une alerte tant que le certificat racine n'est pas installé sur le poste client.

Action sur le poste Windows :
Installer ca.crt dans le magasin Autorités de certification racines de confiance (Ordinateur local).

Redémarrer le navigateur.

Accéder à https://passbolt.indiogroup.lan
![capture_d'écran_2026-02-14_220936.png](/passbolt/capture_d'écran_2026-02-14_220936.png)