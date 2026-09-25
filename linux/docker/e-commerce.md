---
title: Infrastructure E-Commerce Homelab
description: 
published: 1
date: 2026-09-25T15:42:59.792Z
tags: 
editor: markdown
dateCreated: 2026-03-01T13:29:04.064Z
---

# Guide Complet : Infrastructure E-Commerce Homelab "Indio Infrastructure Pro"

Bienvenue sur le projet **Indio Infrastructure Pro**, une plateforme microservices robuste dédiée à la vente de matériel pour passionnés d'infrastructure (Intel NUC, GPU Tesla T4, Switchs Mikrotik). Ce document détaille l'architecture microservices, le cycle de vie des conteneurs, et l'automatisation totale via GitLab CI/CD.
 
* **Dépôt GitLab** : [https://gitlab.com/indiogroup/terraform-eks](https://gitlab.com/indiogroup/projet_final_docker)

![capture_d'écran_2026-03-01_163508.png](/e-commerce/capture_d'écran_2026-03-01_163508.png)

---

## 1. Architecture Docker Compose (L'Orchestrateur)

Si vous souhaitez voir le résultat immédiatement, exécutez simplement les commandes suivantes dans votre terminal :

Le fichier `docker-compose.prod.yml` définit un écosystème complet et sécurisé où chaque service est isolé et optimisé.

```bash
include:
  - project: 'drafboll/projet_final_docker'
    file: '/templates/.services_template.yml'
    ref: 'feature/auth-service'

stages:
  - build
  - script
  - deploy

build-auth:
  extends: .services_template
  variables:
    SERVICE_NAME: auth-service
    SERVICE_PATH: ./services/auth-service
  rules:
    - if: '$CI_COMMIT_BRANCH == "feature/auth-service"'
    - if: '$CI_COMMIT_BRANCH == "main"'
    - if: '$CI_COMMIT_BRANCH == "dev"'


build-order:
  extends: .services_template
  variables:
    SERVICE_NAME: order-service
    SERVICE_PATH: ./services/order-service
  rules:
    - if: '$CI_COMMIT_BRANCH == "feature/order-service"'
    - if: '$CI_COMMIT_BRANCH == "main"'
    - if: '$CI_COMMIT_BRANCH == "dev"'

build-product:
  extends: .services_template
  variables:
    SERVICE_NAME: product-service
    SERVICE_PATH: ./services/product-service
  rules:
    - if: '$CI_COMMIT_BRANCH == "feature/product-service"'
    - if: '$CI_COMMIT_BRANCH == "main"'
    - if: '$CI_COMMIT_BRANCH == "dev"'

build-frontend:
  extends: .services_template
  variables:
    SERVICE_NAME: frontend
    SERVICE_PATH: ./frontend
  rules:
    - if: '$CI_COMMIT_BRANCH == "feature/frontend"'
    - if: '$CI_COMMIT_BRANCH == "main"'
    - if: '$CI_COMMIT_BRANCH == "dev"'

deploy-prod-on-dev:
  stage: deploy
  tags:
    - shell-runner  # Ton runner en mode système
  script:
    - cd /docker/e-commerce
    # On s'assure d'être sur la branche dev localement
    - git checkout dev
    - git pull origin dev
    # Lancement de la version PROD
    - docker compose -f docker-compose.prod.yml down --remove-orphans
    - docker compose -f docker-compose.prod.yml up -d --build
    - docker ps
  rules:
    - if: '$CI_COMMIT_BRANCH == "dev"'
    - if: '$CI_COMMIT_BRANCH == "main"'
```

> Résultat attendu : L'interface "Indio Infrastructure Pro" sera accessible sur votre navigateur à l'adresse http://localhost:8080. Après environ 15 secondes, le catalogue de serveurs et composants Homelab sera automatiquement injecté et visible sans aucune action supplémentaire de votre part.
{.is-info}

![capture_d'écran_2026-03-01_181525.png](/e-commerce/capture_d'écran_2026-03-01_181525.png)

> Pour aller plus loin : Si vous souhaitez comprendre comment nous avons sécurisé cette infrastructure, comment fonctionne le pipeline CI/CD qui automatise ces déploiements, ou comment les images sont construites et versionnées, l'analyse détaillée de l'architecture commence ci-dessous.
{.is-success}


### Détail des Services de Production :
* **Frontend (v1.2)** : Interface Vue.js stylisée avec le thème sombre "Infrastructure Pro". Elle est exposée sur le port 8080.
* **Auth-Service (v1.0)** : Gère la base de données des utilisateurs et la sécurité des sessions sur le port 3001.
* **Product-Service (v1.0)** : Le cœur du catalogue. Il expose une API REST interne sur le port 3000.
* **Order-Service (v1.0)** : Gère le tunnel d'achat et la validation des commandes sur le port 3002.
* **MongoDB (8.0)** : Base de données NoSQL avec volume persistant (mongodb_data) pour la conservation des données.
* **Catalog-Loader (v1.0)** : Service éphémère qui injecte automatiquement les produits au démarrage.

### Le Réseau Isolé :
Nous utilisons un réseau de type bridge nommé `e-commerce-network`.
* **Sécurité** : Seuls les ports nécessaires sont ouverts vers l'extérieur.
* **Résolution DNS** : Les services communiquent via leurs noms Docker (ex: http://product-service:3000) plutôt que par des adresses IP.



---

## 2. Stratégie Docker Hub (Versionnement et Décentralisation)

L'utilisation de Docker Hub comme registre centralisé transforme notre serveur local en une véritable infrastructure Cloud sécurisée.
![capture_d'écran_2026-03-01_180603.png](/e-commerce/capture_d'écran_2026-03-01_180603.png)

### Pourquoi externaliser les images sur Docker Hub ?
* **Indépendance et Survie (Disaster Recovery)** : En cas de panne matérielle du serveur **SVR-DOCKER-01**, les images ne sont pas perdues. Elles sont sauvegardées dans le Cloud, permettant une restauration immédiate de la stack sur n'importe quelle autre machine.
* **Optimisation du Pipeline CI/CD** : Le serveur de production n'a plus besoin de compiler le code (Build) à chaque déploiement. Il se contente de télécharger l'image prête à l'emploi, ce qui économise les ressources CPU/RAM et accélère la mise en ligne.
* **Standardisation (Build Once, Run Anywhere)** : L'image testée par GitLab est strictement la même que celle déployée en production, garantissant l'absence d'erreurs liées à l'environnement de développement.
* **Professionnalisme et Scalabilité** : Le projet devient portable et partageable. N'importe quel collaborateur peut lancer l'infrastructure complète avec un simple `docker compose pull` sans avoir accès aux fichiers sources.

### Pourquoi bannir le tag ":latest" ?
* **Reproductibilité** : Le tag `latest` est instable par nature. En utilisant le tag `v1.2`, nous "figeons" le code validé, assurant que le serveur aura exactement le même comportement aujourd'hui et dans six mois.
* **Rollback Instantané** : En cas de bug sur une nouvelle version, il suffit de repasser le tag à `v1.1` dans le fichier YAML pour restaurer l'ancienne version fonctionnelle en quelques secondes.

### Procédure de Publication :
    # Taggage d'une image stable
    docker tag drafboll/e-commerce-frontend:latest drafboll/e-commerce-frontend:v1.2

    # Envoi vers le registre distant
    docker push drafboll/e-commerce-frontend:v1.2


---

## 3. L'Image Custom "Catalog-Loader" (Data-Loader)

Cette image résout le problème de l'initialisation manuelle de la base de données.

### Résolution du bug Windows (CRLF vs LF) :
Les scripts créés sous Windows contiennent des caractères invisibles \r provoquant l'erreur "not found" sous Linux.
* **Solution** : Intégration de l'utilitaire dos2unix directement dans le Dockerfile.init pour "nettoyer" le script durant le build.

### Configuration du Loader :
    FROM alpine:latest
    RUN apk add --no-cache curl dos2unix
    COPY init-products.sh /init-products.sh
    RUN dos2unix /init-products.sh && chmod +x /init-products.sh
    ENTRYPOINT ["/bin/sh", "-c", "sleep 15 && /bin/sh /init-products.sh"]


---

## 4. Pipeline CI/CD GitLab : L'Automatisation Totale

![capture_d'écran_2026-03-01_181609.png](/e-commerce/capture_d'écran_2026-03-01_181609.png)

![capture_d'écran_2026-03-01_181630.png](/e-commerce/capture_d'écran_2026-03-01_181630.png)

Le pipeline GitLab CI/CD est le moteur qui permet de passer du code à la production sans intervention humaine manuelle sur le serveur.
![capture_d'écran_2026-03-01_180652.png](/e-commerce/capture_d'écran_2026-03-01_180652.png)
### 4.1. Analyse du fichier `.gitlab-ci.yml`
Notre configuration utilise des **templates** pour rester "DRY" (Don't Repeat Yourself) et optimiser la maintenance.

```bash
include:
  - project: 'drafboll/projet_final_docker'
    file: '/templates/.services_template.yml'
    ref: 'feature/auth-service'

stages:
  - build
  - script
  - deploy

build-auth:
  extends: .services_template
  variables:
    SERVICE_NAME: auth-service
    SERVICE_PATH: ./services/auth-service
  rules:
    - if: '$CI_COMMIT_BRANCH == "feature/auth-service"'
    - if: '$CI_COMMIT_BRANCH == "main"'
    - if: '$CI_COMMIT_BRANCH == "dev"'

build-order:
  extends: .services_template
  variables:
    SERVICE_NAME: order-service
    SERVICE_PATH: ./services/order-service
  rules:
    - if: '$CI_COMMIT_BRANCH == "feature/order-service"'
    - if: '$CI_COMMIT_BRANCH == "main"'
    - if: '$CI_COMMIT_BRANCH == "dev"'

build-product:
  extends: .services_template
  variables:
    SERVICE_NAME: product-service
    SERVICE_PATH: ./services/product-service
  rules:
    - if: '$CI_COMMIT_BRANCH == "feature/product-service"'
    - if: '$CI_COMMIT_BRANCH == "main"'
    - if: '$CI_COMMIT_BRANCH == "dev"'

build-frontend:
  extends: .services_template
  variables:
    SERVICE_NAME: frontend
    SERVICE_PATH: ./frontend
  rules:
    - if: '$CI_COMMIT_BRANCH == "feature/frontend"'
    - if: '$CI_COMMIT_BRANCH == "main"'
    - if: '$CI_COMMIT_BRANCH == "dev"'

deploy-prod-on-dev:
  stage: deploy
  tags:
    - shell-runner
  script:
    - cd /docker/e-commerce
    - git checkout dev
    - git pull origin dev
    - docker compose -f docker-compose.prod.yml down --remove-orphans
    - docker compose -f docker-compose.prod.yml up -d --build
    - docker ps
  rules:
    - if: '$CI_COMMIT_BRANCH == "dev"'
    - if: '$CI_COMMIT_BRANCH == "main"'
```

#### A. Inclusion et Fondations (`include` & `stages`)

* **`include`** : Cette section permet de réutiliser de la logique externe. Nous importons un **template** nommé `.services_template.yml` qui contient les instructions génériques de build Docker. Cela permet de centraliser la maintenance et d'éviter la duplication de code pour chaque microservice.
* **`stages`** : Définit l'ordre chronologique d'exécution des tâches :
    * **`build`** : Étape de construction technique des images Docker.
    * **`deploy`** : Mise en production effective sur le serveur **SVR-DOCKER-01**.

---

#### B. Les Jobs de Build (Microservices)

Chaque bloc de service (`build-auth`, `build-order`, `build-product`, `build-frontend`) suit une architecture industrielle identique :

* **`extends: .services_template`** : Le job hérite de toutes les instructions de construction (Docker-in-Docker) définies dans le template maître.
* **`variables`** : On injecte les paramètres spécifiques au service :
    * **`SERVICE_NAME`** : Définit le nom de l'image finale sur le registre (ex: `auth-service`).
    * **`SERVICE_PATH`** : Indique au runner le chemin du dossier contenant le code source et le Dockerfile.
* **`rules`** : Définit les conditions de déclenchement. Le build est activé lors d'un push sur la branche de fonctionnalité correspondante (ex: `feature/auth-service`), ou lors d'une fusion sur les branches de regroupement `dev` ou `main`.
![capture_d'écran_2026-03-01_180805.png](/e-commerce/capture_d'écran_2026-03-01_180805.png)


---

#### C. Le Job de Déploiement (`deploy-prod-on-dev`)

C'est ici que l'automatisation pilote physiquement le serveur Rocky Linux via le **Shell Runner**.

* **`tags: shell-runner`** : Indique que ce job s'exécute directement sur le système hôte, lui donnant un accès natif au moteur Docker local.
* **`script`** : Suite de commandes critiques exécutées sur le serveur :
    1. **`cd /docker/e-commerce`** : Positionnement dans le répertoire de travail du projet.
    2. **`git checkout dev`** & **`git pull origin dev`** : Synchronisation du code local avec le dépôt distant pour récupérer les dernières validations.
    3. **`docker compose ... down`** : Arrêt propre des anciens conteneurs pour libérer les ressources.
    4. **`docker compose ... up -d --build`** : Relancement de la stack en mode détaché. L'option `--build` force la reconstruction locale des images si des changements sont détectés.
    5. **`docker ps`** : Commande de contrôle final pour valider l'état "Up" de tous les conteneurs dans les logs GitLab.
![capture_d'écran_2026-03-01_180851.png](/e-commerce/capture_d'écran_2026-03-01_180851.png)

---

#### D. La branche `main` : La Version Stable

Dans notre stratégie de branchement, la branche **`main`** fait office de gardienne de la stabilité :

* **Validation Finale** : Après validation sur la branche `dev`, une "Merge Request" est effectuée vers `main`.
* **Build Automatique** : Chaque modification sur `main` déclenche la reconstruction de toutes les images microservices pour s'assurer que les images de "Production Finale" sont prêtes et à jour.
* **Isolation** : Le déploiement automatique est volontairement restreint à la branche `dev`. Cela permet de tester les nouvelles fonctionnalités dans un environnement isolé avant toute mise en ligne définitive sur l'environnement client.

![capture_d'écran_2026-03-01_180939.png](/e-commerce/capture_d'écran_2026-03-01_180939.png)

---

### 4.2. Construction des Images (Dockerfiles)
Chaque microservice possède son propre `Dockerfile` optimisé pour la production.

* **Services Backend (Auth, Order, Product)** : Ils utilisent le **Multi-stage Build**.
    * **Étape 1 (builder)** : Utilise une image `node:18` complète pour compiler les dépendances et installer les paquets via `npm install`.
    * **Étape 2 (production)** : Utilise une image `node:18-slim` beaucoup plus légère, ne contenant que le strict nécessaire (dossier `/app`) pour l'exécution, réduisant ainsi la surface d'attaque.
* **Service Frontend** : Utilise `node:18` pour servir l'application via Vite en mode développement/test sur le port 8080.
* **Catalog Loader (Data-Loader)** : Basé sur `alpine:latest` pour une légèreté extrême, incluant `curl` pour les requêtes API et `dos2unix` pour garantir la compatibilité des scripts entre Windows et Linux.



---

### 4.2. Construction des Images (Dockerfiles)

Chaque microservice de l'infrastructure **Indio Infrastructure Pro** possède son propre Dockerfile, conçu pour maximiser la sécurité et minimiser le poids des images finales en production.

---

#### A. Services Backend (Auth, Order, Product)
Ces services utilisent une stratégie de **Multi-stage Build**. Cette méthode permet de séparer l'environnement de compilation (lourd) de l'environnement d'exécution (léger).
![capture_d'écran_2026-03-01_181055.png](/e-commerce/capture_d'écran_2026-03-01_181055.png)
**Dockerfile type (exemple Auth-Service) :**
```bash
# Étape 1 : Build de l'application Node.js
FROM node:18 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .

# Étape 2 : Image finale, plus légère
FROM node:18-slim
WORKDIR /app
COPY --from=builder /app /app
EXPOSE 3001
CMD ["npm", "start"]
```
* **Explication** : L'étape 1 (`builder`) utilise une image `node:18` complète pour installer les dépendances via `npm install`. L'étape 2 bascule sur `node:18-slim`, une version dépouillée qui ne contient que le dossier `/app` final. Cela réduit la taille de l'image de plusieurs centaines de mégaoctets et limite la surface d'attaque en supprimant les outils de développement (compilateurs, caches npm) inutiles en production.



---

#### B. Service Frontend
Le Frontend est conçu pour servir l'interface utilisateur de manière réactive et optimisée pour le développement.

**Dockerfile (Frontend) :**
```bash

# Utilisation de l'image Node officielle (version 18 comme dans tes logs)
FROM node:18

# Définition du dossier de travail dans le conteneur
WORKDIR /app

# Copie des fichiers de dépendances en premier pour optimiser le cache Docker
COPY package*.json ./

# Installation des dépendances (inclut Vite et Vue)
RUN npm install

# Copie de TOUT le contenu du dossier frontend (code source, config Vite, etc.)
# Note : Grâce au contexte de build 'frontend/', cela copie directement index.html à la racine /app
COPY . .

# Information sur le port utilisé par le conteneur
EXPOSE 8080

# Commande de démarrage CRITIQUE :
# --host 0.0.0.0 : Autorise l'accès depuis l'extérieur du conteneur (évite Connection Refused)
# --port 8080 : Aligne Vite sur le port attendu par ton Docker Compose et ton Reverse Proxy
CMD ["npx", "vite", "--host", "0.0.0.0", "--port", "8080"]

```


* **Explication** : Ce Dockerfile utilise une image `node:18` pour exécuter l'application via le serveur de développement Vite. La structure des commandes `COPY` est stratégique : en copiant d'abord le `package.json`, Docker met en cache l'étape `npm install`. Ainsi, si vous modifiez uniquement le code source du frontend, la reconstruction de l'image est quasi instantanée car Docker ne réinstalle pas les dépendances.

---

#### C. Catalog Loader (Data-Loader)
Ce service est un utilitaire d'automatisation critique pour l'injection du catalogue sans intervention manuelle.

**Dockerfile.init (Data-Loader) :**
```bash
FROM alpine:latest

# Installation de curl et dos2unix (toujours nécessaire pour le réseau et le format)
RUN apk add --no-cache curl dos2unix

WORKDIR /app

# Copie et préparation du script
COPY scripts/init-products.sh .
RUN dos2unix init-products.sh && chmod +x init-products.sh

# Utilisation de /bin/sh explicitement
ENTRYPOINT ["/bin/sh", "init-products.sh"]
```

* **Explication** : Basé sur `alpine:latest`, ce conteneur est extrêmement léger (environ 13.9 MB). L'ajout de `dos2unix` est la clé de la stabilité : cet outil convertit les fins de lignes Windows (CRLF) en format Linux (LF) directement durant le build. Cela garantit que le script `/init-products.sh` sera toujours exécutable sur le serveur, évitant les erreurs de type "not found" liées à l'encodage du fichier. Enfin, le `sleep 15` assure que la base de données MongoDB et le Product-Service ont fini de démarrer avant que le script ne tente d'injecter les produits.
---

### 4.4. Installation des Runners (Docker & Shell)
Pour que GitLab puisse agir sur le serveur **SVR-DOCKER-01**, il faut installer et enregistrer des "Runners".
Pour la facon docker runner : 

![capture_d'écran_2026-03-01_181159.png](/e-commerce/capture_d'écran_2026-03-01_181159.png)

#### Installation du binaire sur Rocky Linux :
```bash
sudo curl -L --output /usr/local/bin/gitlab-runner [https://gitlab-runner-downloads.s3.amazonaws.com/latest/binaries/gitlab-runner-linux-amd64](https://gitlab-runner-downloads.s3.amazonaws.com/latest/binaries/gitlab-runner-linux-amd64)
sudo chmod +x /usr/local/bin/gitlab-runner
sudo gitlab-runner install --user=docker --working-directory=/home/docker
sudo gitlab-runner start
```
pour la facon docker shell 
![capture_d'écran_2026-03-01_181402.png](/e-commerce/capture_d'écran_2026-03-01_181402.png)

---

###  Guide de Mise à Jour du Catalogue et de l'Image Docker

Cette section détaille la procédure opérationnelle pour modifier le catalogue Homelab, reconstruire l'image utilitaire `indio-data-loader` et la publier sur le registre Docker Hub pour garantir la cohérence de l'infrastructure.

---

#### 1. Le Script d'Initialisation Complet (`scripts/init-products.sh`)
Ce script est le cœur de l'automatisation des données. Il utilise `curl` pour peupler l'API `product-service` via des requêtes POST.

```bash
#!/bin/sh
# Attente pour s'assurer que le service est prêt (Docker Network)
echo " Attente du démarrage des microservices..."
sleep 15

# Configuration (product-service est le nom du conteneur dans le réseau Docker)
API_URL="http://product-service:3000/api"
TOKEN="efrei_super_pass"

create_product() {
    local name=$1
    local price=$2
    local description=$3
    local stock=$4
    
    curl -X POST "${API_URL}/products" \
        -H "Content-Type: application/json" \
        -H "Authorization: Bearer ${TOKEN}" \
        -d "{
            \"name\": \"${name}\",
            \"price\": ${price},
            \"description\": \"${description}\",
            \"stock\": ${stock}
        }"
    echo
}

echo " Injection du catalogue étendu Indio Infrastructure Pro..."

# --- SERVEURS & CALCUL ---
create_product "Intel NUC 13 Pro" 580 "Serveur compact i7-1360P, 32GB RAM, parfait pour Proxmox." 15
create_product "NVIDIA Tesla T4" 950 "GPU 16GB pour transcodage Plex et inférence IA locale (LLM)." 5
create_product "Dell PowerEdge R730" 1200 "Serveur rackable 2U, bi-processeur Xeon, 128GB RAM DDR4." 3
create_product "Raspberry Pi 5 8GB" 85 "Micro-ordinateur pour nodes Kubernetes légers et IoT." 50

# --- RÉSEAU ---
create_product "Switch Mikrotik 10Gbps" 145 "Switch administrable 4 ports SFP+ pour réseau ultra-rapide." 12
create_product "Routeur Ubiquiti UDM Pro" 380 "Console réseau tout-en-un avec firewall et contrôleur UniFi." 7
create_product "Point d'accès WiFi 6 Pro" 160 "AP UniFi WiFi 6 haute performance pour environnements denses." 20
create_product "Câble DAC SFP+ 1m" 25 "Câble d'attache directe 10Gbps pour liaison switch-serveur." 100

# --- STOCKAGE ---
create_product "SSD NVMe 2TB Samsung" 160 "Samsung 990 Pro pour stockage VM et databases haute performance." 25
create_product "HDD Seagate IronWolf 12TB" 280 "Disque dur spécial NAS optimisé pour fonctionnement 24/7." 15
create_product "NAS Synology DS923+" 620 "Solution de stockage 4 baies extensible pour Docker et backups." 10

# --- INFRA & ACCESSOIRES ---
create_product "Onduleur APC 1500VA" 450 "Protection contre les coupures de courant et surtensions." 8
create_product "Baie de brassage 9U" 120 "Coffret mural pour organiser proprement votre Homelab." 5
create_product "PDU Administrable" 95 "Multiprise rackable avec suivi de consommation électrique." 10
create_product "Sonde Température Zigbee" 20 "Capteur pour monitoring thermique de la salle serveur." 40

echo " Catalogue complet (15 articles) injecté avec succès !"

```


---

#### 2. Procédure de Publication Manuelle (Docker Hub)
Si vous devez mettre à jour l'image en dehors du pipeline CI/CD, suivez ces étapes sur le serveur **SVR-DOCKER-01**.

```bash
    # 1. Accéder au dossier racine du projet
    cd /docker/e-commerce

    # 2. Reconstruire l'image avec un tag de version précis
    # L'utilisation de tags (ex: v1.1) permet de garder une trace historique
    docker build -t drafboll/indio-data-loader:v1.1 -f Dockerfile.init .

    # 3. Authentification sur Docker Hub
    docker login

    # 4. Envoi de l'image vers le registre distant
    docker push drafboll/indio-data-loader:v1.1
```


---

#### 3. Mise à jour automatisée via GitLab CI/CD (Recommandé)
Le pipeline est configuré pour détecter les changements de catalogue et redéployer la stack sans intervention manuelle.

1.  **Enregistrement des modifications** :
```bash
    git add scripts/init-products.sh
    git commit -m "feat: ajout de nouveaux produits au catalogue"
```
2.  **Déclenchement du pipeline** :
```bash
    git push origin dev
```

**Mécanisme interne** :
Le pipeline détecte le commit sur la branche `dev`, sollicite le **Docker Runner** pour reconstruire l'image, puis utilise le **Shell Runner** pour mettre à jour les conteneurs sur le serveur. Le nouveau catalogue est injecté automatiquement par le conteneur `indio-catalog-loader` au bout de 15 secondes.

## 6. Reverse Proxy & Sécurité SSL (Caddy Server)

Pour passer d'un accès technique en `http://192.168.40.30:8080` à une URL professionnelle sécurisée comme `https://shop.indiogroup.fr`, nous utilisons **Caddy** sur le serveur **SVR-CADDY-01** (Rocky Linux).

### 6.1. Configuration du Caddyfile
Caddy gère nativement le SSL Let's Encrypt et redirige le trafic vers le backend Docker.

```caddy
shop.indiogroup.fr {
    # Reverse Proxy vers le serveur Docker distant
    reverse_proxy 192.168.40.30:8080
}
```
Le bloc dans ton Caddyfile transforme ton serveur Rocky Linux en une porte d'entrée sécurisée.

- **shop.indiogroup.fr** { ... } : Caddy surveille toutes les requêtes arrivant sur Internet. S'il voit ce nom de domaine, il active automatiquement le protocole HTTPS en allant chercher un certificat SSL gratuit chez Let's Encrypt.

- **reverse_proxy 192.168.40.30:8080** : C'est l'instruction la plus importante. Elle dit à Caddy : "Ne cherche pas les fichiers du site sur ton propre disque dur. Prends la requête du visiteur et renvoie-la telle quelle au serveur Docker (192.168.40.30) sur le port 8080".

### 6.2. Configuration du Frontend (Vite)
Le fichier frontend/vite.config.js est configuré pour autoriser le domaine et éviter l'erreur "Blocked request".

```JavaScript
// frontend/vite.config.js
export default defineConfig({
  server: {
    host: '0.0.0.0',
    port: 8080,
    allowedHosts: ['shop.indiogroup.fr'], // Correction de l'erreur Blocked Request
    cors: true
  }
})
```

C'est ici que ton application (le Frontend) décide si elle accepte de répondre ou non.

- **host**: '0.0.0.0' : Par défaut, un serveur de développement n'écoute que lui-même (localhost). En mettant 0.0.0.0, tu dis à Vite : "Écoute toutes les cartes réseau du conteneur". C'est ce qui permet à Caddy de lui parler de l'extérieur.

- **allowedHosts**: ['shop.indiogroup.fr'] : C'est ta sécurité anti-piratage (DNS Rebinding). Par défaut, Vite bloque les requêtes qui arrivent avec un nom de domaine qu'il ne connaît pas (l'erreur "Blocked Request" que tu avais). Ici, tu lui donnes une "liste blanche" : "Si la requête vient de Caddy avec le nom shop.indiogroup.fr, alors c'est bon, tu peux répondre".

- **cors**: true : Cela autorise ton navigateur à faire des requêtes vers tes autres microservices (Auth, Product, etc.) même s'ils sont sur des ports ou des sous-domaines différents.

![capture_d'écran_2026-03-03_113113.png](/e-commerce/capture_d'écran_2026-03-03_113113.png)


## 7. Gestion des Données : Architecture Multi-Bases

Contrairement à une application monolithique, notre infrastructure **Indio Infrastructure Pro** utilise une séparation stricte des données pour garantir l'isolation des services. Bien que tous les services partagent l'instance **MongoDB**, ils opèrent sur des bases de données logiques distinctes.

### 7.1. Structure des Bases de Données
Voici comment vos données sont réparties après une commande :

* **Base `ecommerce`** : Contient le catalogue produit (Collection `products`) et les paniers temporaires (Collection `carts`).
* **Base `auth`** : Stocke exclusivement les profils utilisateurs et les identifiants de connexion (Collection `users`).
* **Base `orders`** : Archive l'historique des transactions et le détail des commandes passées (Collection `orders`).



### 7.2. Commandes de Vérification (Administration)
Pour auditer vos données directement depuis le serveur **SVR-DOCKER-01**, utilisez le shell MongoDB :

```bash
# Entrer dans le conteneur
docker exec -it e-commerce-mongodb-1 mongosh

# Vérifier les produits (Base ecommerce)
use ecommerce
db.products.find().pretty()

# Vérifier les comptes clients (Base auth)
use auth
db.users.find().pretty()

# Vérifier les ventes réalisées (Base orders)
use orders
db.orders.find().pretty()
```
![capture_d'écran_2026-03-03_114831.png](/e-commerce/capture_d'écran_2026-03-03_114831.png)

### 7.3. Intégrité et Flux de Données

- Lorsqu'un client passe une commande sur https://shop.indiogroup.fr :

- L'Order-Service crée un enregistrement dans la base orders.

- Le Product-Service est sollicité pour décrémenter le stock dans la base ecommerce.

- L'utilisateur peut consulter son historique car l'interface web agrège les données provenant de la base orders et de la base auth.


---

## 8. Rapport de Sécurité et Conformité (Analyse RSSI)

En tant qu'administrateur de l'infrastructure **Indio Infrastructure Pro**, la sécurité a été intégrée dès la conception (Security by Design). Ce rapport détaille les mécanismes mis en place pour protéger l'intégrité du serveur **SVR-DOCKER-01** et la confidentialité des données clients.

---

### 8.1. Gestion des Privilèges et Contrôle d'Accès
L'un des vecteurs d'attaque les plus courants est l'escalade de privilèges. Nous avons appliqué le **principe du moindre privilège**.

* **Runner Non-Root** : Le Shell Runner GitLab ne possède pas les droits `sudo`. Il appartient exclusivement au groupe `docker`, ce qui lui permet de piloter les conteneurs sans pouvoir modifier les fichiers sensibles du système d'exploitation Rocky Linux.
* **Isolation de l'Exécution** : Le Docker Runner (Builder) utilise le mode `privileged` uniquement pour la construction des images, garantissant qu'aucun artefact de compilation ne reste sur l'hôte après le job.



---

### 8.2. Isolation Réseau et Segmentation (Microservices)
L'architecture microservices permet de compartimenter les services pour limiter la propagation d'une éventuelle intrusion.

* **Réseau Privé Virtuel (`e-commerce-network`)** : Tous les services (Auth, Product, Order, MongoDB) communiquent sur un réseau interne Docker de type `bridge`.
* **MongoDB Invisible** : La base de données n'est **jamais exposée** sur le réseau public. Elle n'a pas de mapping de port sur l'hôte, ce qui la rend totalement invisible pour un attaquant externe tentant de scanner les ports du serveur.
* **Reverse Proxying Implicite** : Seul le Frontend est exposé sur le port `8080`, servant de point d'entrée unique et contrôlé pour les utilisateurs.

---

### 8.3. Immuabilité et Intégrité des Images
Le passage du tag `latest` vers des **tags versionnés** (ex: `v1.2`) est une mesure de sécurité critique.

* **Protection contre l'Empoisonnement** : Les tags versionnés empêchent l'écrasement accidentel ou malveillant d'une image de production. Une image poussée avec le tag `v1.2` devient une référence immuable.
* **Traçabilité Totale** : Chaque image sur Docker Hub peut être reliée à un commit spécifique sur GitLab. En cas de comportement suspect, nous pouvons auditer le code source exact ayant généré l'image en production.



---

### 8.4. Sécurisation du Cycle de Build (Multi-stage)
L'utilisation de Dockerfiles "Multi-stage" réduit drastiquement la surface d'attaque des conteneurs.

* **Réduction de l'Empreinte** : En utilisant des images finales `node:18-slim` ou `alpine`, nous supprimons les compilateurs, les gestionnaires de paquets (npm, apk) et les shells inutiles.
* **Moins de Vulnérabilités (CVE)** : Moins de binaires dans l'image signifie moins de failles de sécurité potentielles exploitables par un attaquant.

---

### 8.5. Automatisation de l'Initialisation Sécurisée
Le service `indio-data-loader` sécurise la phase critique de mise en service.

* **Scripting Propre** : L'utilisation de `dos2unix` garantit qu'aucun caractère malveillant ou malformé n'est injecté lors de l'exécution des scripts shell.
* **Injection Interne** : L'injection des produits se fait via le réseau interne Docker, évitant ainsi d'exposer des routes API sensibles sur l'internet public durant la phase de configuration.

---
**Conclusion Sécurité** : L'infrastructure **Indio Infrastructure Pro** respecte les standards de durcissement (hardening) modernes, offrant une plateforme résiliente face aux menaces actuelles tout en restant agile grâce au pipeline CI/CD.