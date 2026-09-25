---
title: Configurer la synchronisation Git (GitHub) avec Wiki.js sous Docker
description: 
published: 1
date: 2026-09-25T17:51:47.531Z
tags: 
editor: markdown
dateCreated: 2026-09-25T17:51:47.531Z
---

# Configurer la synchronisation Git (GitHub) avec Wiki.js sous Docker

## I. Présentation

Ce guide détaille la mise en place d'une synchronisation automatique entre une instance **Wiki.js** (hébergée sous Docker) et un dépôt **GitHub** distant. L'objectif est d'utiliser le module de "Stockage Git" de Wiki.js pour sauvegarder l'intégralité du contenu textuel du wiki au format **Markdown (.md)**.

Cette standardisation permet non seulement d'avoir une sauvegarde externe versionnée, mais également d'exploiter facilement ces données pour des projets tiers (ex: ingestion de la documentation par un moteur RAG comme **AnythingLLM**).

![capture_d'écran_2026-09-25_194227.png](/git/capture_d'écran_2026-09-25_194227.png)

---

## II. Comprendre le stockage Git sous Wiki.js

### A. Ce qui est synchronisé vs Ce qui ne l'est pas

* **Synchronisé** : L'arborescence des dossiers, le texte des pages au format brut (Markdown / `.md`), les métadonnées (titre, tags, date de création).
* **Non synchronisé** : Les fichiers médias, images (`.png`, `.jpg`), PDF et pièces jointes stockés dans le dossier d'upload local.

### B. Sens de synchronisation

* **Unidirectionnel (Push)** : Wiki.js envoie ses modifications vers GitHub.
* **Bidirectionnel (Push/Pull)** : Wiki.js peut aussi lire les modifications faites directement sur GitHub pour mettre à jour la base de données (si activé).

> 🎯 Le dépôt Git agit comme un miroir textuel de votre base de données (PostgreSQL ou SQLite).

---

## III. Environnement de mise en place

* **Wiki.js** déployé via conteneur **Docker** (accès administrateur à l'interface web).
* Accès au **terminal du serveur hôte** (pour gérer les permissions du conteneur).
* Un compte **GitHub** actif.
* (Optionnel) Un outil de RAG comme AnythingLLM prêt à ingérer le dépôt.

---

## IV. Préparation du dépôt GitHub

1. Se connecter à GitHub et cliquer sur **New repository**.
2. Nommer le dépôt, par exemple : `wikijs`.
![capture_d'écran_2026-09-25_194331.png](/git/capture_d'écran_2026-09-25_194331.png)
3. Définir la visibilité sur **Private** (recommandé pour une documentation interne) ou **Public**.
4. Cochez impérativement la case **Add a README file** pour initialiser la branche principale (généralement `main`).
5. Cliquer sur **Create repository**.

---

## V. Génération et ajout des clés SSH

Wiki.js nécessite une authentification par clé SSH (sans mot de passe) pour communiquer de manière autonome avec GitHub.

### 1. Génération de la clé sur le serveur
Depuis le terminal de votre serveur, tapez :
```bash
ssh-keygen -t ed25519 -C "pro@alexandre-faye.fr"
```
![capture_d'écran_2026-09-25_194447.png](/git/capture_d'écran_2026-09-25_194447.png)
> ⚠️ **Important :** Ne renseignez **aucune passphrase** (laissez vide et appuyez sur Entrée) pour permettre l'automatisation en arrière-plan.

### 2. Ajout de la clé publique sur GitHub
1. Affichez la clé publique : `cat ~/.ssh/id_ed25519.pub`
2. Sur GitHub, allez dans **Settings > Deploy keys > Add deploy key**.
3. Donnez un titre (ex: `Serveur Wiki.js`), collez la clé publique.
4. Cochez impérativement **Allow write access** (Autoriser l'accès en écriture).
5. Cliquez sur **Add key**.

![capture_d'écran_2026-09-25_194543.png](/git/capture_d'écran_2026-09-25_194543.png)

### 3. Récupération de la clé privée
Affichez la clé privée et copiez l'intégralité de son contenu (incluant les balises `-----BEGIN...` et `-----END...`) :
```bash
cat ~/.ssh/id_ed25519
```
![capture_d'écran_2026-09-25_194656.png](/git/capture_d'écran_2026-09-25_194656.png)

---

## VI. Configuration de l'interface Wiki.js

1. Connectez-vous à Wiki.js en tant qu'administrateur.
2. Allez dans **Administration > Stockage > Git**.
![capture_d'écran_2026-09-25_194752.png](/git/capture_d'écran_2026-09-25_194752.png)

### Paramètres à renseigner :
* **Git Repository URL** : URL SSH de votre dépôt (ex: `git@github.com:utilisateur/wikijs.git`).
* **Branch** : `main` (ou `master` selon la création).
* **Authentication Method** : `SSH Authentication Only`.
* **SSH Private Key Mode** : `Contents`.
* **SSH Private Key Contents** : Collez la clé privée récupérée à l'étape précédente.
* **Verify SSL Certificate** : Activé.
* **Author Name / Email** : Renseignez vos informations.
* **Sync Interval** : Définir la fréquence (ex: toutes les 1 heure).

Cliquez sur **Appliquer**. *Ne lancez pas encore la synchronisation, des correctifs de permissions Docker sont nécessaires.*

---

## VII. Résolution des permissions Docker (Crucial)

Dans un environnement Docker, l'utilisateur restreint de Wiki.js (ex: `node` ou `abc`) se heurte souvent aux permissions de l'utilisateur `root` du serveur hôte, causant des erreurs `EACCES: permission denied` ou `dubious ownership`.

1. Entrez dans le terminal du conteneur en tant que root :
   ```bash
   docker exec -it -u root <nom_du_conteneur_wikijs> /bin/bash
   ```
2. Créez l'arborescence du dépôt si elle n'existe pas :
   ```bash
   mkdir -p /app/wiki/data/repo
   ```
![capture_d'écran_2026-09-25_194949.png](/git/capture_d'écran_2026-09-25_194949.png)
3. Accordez les droits d'écriture complets au dossier de données :
   ```bash
   chmod -R 777 /app/wiki/data
   ```
4. Protégez spécifiquement votre clé privée SSH (le protocole SSH rejettera une clé avec des droits 777) :
   ```bash
   chmod 600 /app/wiki/data/secure/git-ssh.pem
   ```
5. Ajoutez une exception de sécurité globale Git pour le dossier du wiki :
   ```bash
   git config --system --add safe.directory '/app/wiki/data/repo'
   ```
6. Quittez le conteneur (`exit`).

---

## VIII. Lancement et vérification de la synchronisation

1. Retournez dans l'administration Wiki.js (**Stockage > Git**).
2. Descendez dans la section **Actions**.
3. Exécutez les actions dans cet ordre :
   * Cliquez sur **Purge Local Repository** (nettoie le cache local).
   * Cliquez sur **Force Sync** (initialise le clone et pousse les données).
   
![capture_d'écran_2026-09-25_195018.png](/git/capture_d'écran_2026-09-25_195018.png)

> ⏳ **Note sur SQLite :** Si votre wiki contient beaucoup de pages et utilise SQLite, le bouton "Add Untracked Changes" peut générer une erreur de Timeout (la base de données s'étouffe). Dans ce cas, modifiez et sauvegardez vos pages par petits groupes via l'éditeur classique ; la synchronisation s'effectuera en douceur à chaque sauvegarde.

### Vérification post-déploiement :
* Les dossiers et sous-dossiers sont visibles sur GitHub.
* Les fichiers Markdown `.md` contiennent bien la structure de vos pages.
* Le statut du module Git dans Wiki.js est "Au vert" (Dernière synchronisation réussie).

![capture_d'écran_2026-09-25_195125.png](/git/capture_d'écran_2026-09-25_195125.png)

---

## IX. Conclusion & Cas d'usage RAG

La synchronisation Git de Wiki.js permet :
* Une **sauvegarde décentralisée** automatique.
* Un **historique des versions** géré nativement par GitHub.
* L'exploitation des données textuelles pour l'**Intelligence Artificielle**.

> 🔁 **Astuce pour AnythingLLM :** Le connecteur GitHub d'AnythingLLM peut parfois ignorer les sous-dossiers. Pour ingérer vos 68 pages sans erreur, téléchargez votre dépôt GitHub au format ZIP (bouton `<> Code > Download ZIP`), décompressez-le, et importez les dossiers manuellement dans votre espace de documents AnythingLLM. Vous conserverez ainsi 100% de la donnée textuelle de votre Wiki.