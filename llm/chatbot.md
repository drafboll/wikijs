---
title: Déploiement d'un Chatbot Support 100% Local
description: 
published: 1
date: 2026-09-25T15:50:05.297Z
tags: 
editor: markdown
dateCreated: 2026-09-23T16:59:30.016Z
---

# Tutoriel Complet : Déploiement d'un Chatbot Support 100% Local (Ollama + AnythingLLM)

Ce document détaille la mise en place de A à Z d'un assistant conversationnel privé permettant d'interroger une documentation interne sans aucune fuite de données vers le cloud. L'ensemble de la solution (cerveau IA, base de données et interface) tourne localement sur votre machine.
 
## 1. Préparation de l'environnement Docker

L'environnement repose sur deux composants isolés via Docker :
* **Ollama** : Le moteur qui télécharge et exécute les modèles de langage (LLM) sur votre machine.
* **AnythingLLM** : L'interface utilisateur et le gestionnaire documentaire (RAG - *Retrieval-Augmented Generation*).

**Pourquoi Docker ?** Docker permet de faire tourner ces deux applications dans des "boîtes" (conteneurs) isolées, avec toutes leurs dépendances, garantissant que la solution fonctionnera sur n'importe quelle machine de la même manière.

### Création du répertoire de travail

Ouvrez votre terminal Linux. Nous allons créer un dossier dédié pour ranger proprement notre projet et nous y déplacer :
```bash
    mkdir -p ~/chatbot-local
    cd ~/chatbot-local
```
### Création du fichier de configuration Docker

Nous devons maintenant créer le fichier qui va indiquer à Docker comment télécharger et relier Ollama et AnythingLLM. Créez un fichier nommé `docker-compose.yml` (par exemple avec la commande `vi docker-compose.yml`) et collez-y très exactement ce contenu :

```bash
vi docker-compose.yml
```

```bash
    version: '3.8'
    
    services:
      ollama:
        image: ollama/ollama:latest
        container_name: ollama
        ports:
          - "11434:11434"
        volumes:
          - ./ollama-data:/root/.ollama
        restart: always
    
      anythingllm:
        image: mintplexlabs/anythingllm
        container_name: anythingllm
        ports:
          - "3001:3001"
        volumes:
          - ./anythingllm-data:/app/server/storage
        environment:
          - STORAGE_DIR=/app/server/storage
        restart: always
```

*(Note : les lignes `volumes` permettent de sauvegarder vos documents et configurations directement dans votre dossier `chatbot-local` pour ne rien perdre si le serveur redémarre).*

### Lancement des services

Maintenant que le fichier est prêt, lancez la création de l'infrastructure avec cette commande dans votre terminal :
```bash
    # Lancement des conteneurs en tâche de fond
    docker compose up -d
```

### Téléchargement du modèle de langage

Une fois les conteneurs démarrés, il faut indiquer à Ollama de télécharger le "cerveau" de notre chatbot. Nous utilisons **Mistral**, un modèle performant, très bon en français, et suffisamment léger pour tourner sans carte graphique (sur CPU). Exécutez :
```bash
    # Téléchargement et installation de Mistral dans le conteneur Ollama
    docker exec -it ollama ollama run mistral
```

## 2. Configuration Initiale d'AnythingLLM

Une fois les services démarrés et Mistral téléchargé, toute la configuration s'effectue depuis le navigateur web.

1. Accédez à l'interface d'AnythingLLM via `http://localhost:3001` (ou l'IP de votre serveur).
2. Sur la page d'accueil affichant "Bienvenue", cliquez sur **Commencer**.
3. **Configuration utilisateur :** Choisissez **Juste moi**. Cela permet d'accéder à l'espace de travail sans mettre en place un système lourd de gestion de comptes et d'invitations pour la phase de test.
4. **Sécurité :** Définissez un mot de passe administrateur pour sécuriser l'accès. *Attention : il n'y a pas de mécanisme de récupération en cas d'oubli.*

![capture_d'écran_2026-09-23_181947.png](/chatbot/capture_d'écran_2026-09-23_181947.png)

## 3. Paramétrage des Moteurs d'Intelligence Artificielle

Il faut maintenant connecter l'interface AnythingLLM au moteur Ollama qui contient notre modèle Mistral.

### Fournisseur LLM (Le cerveau conversationnel)
* **LLM Provider** : Sélectionnez `Ollama`.
* **Ollama Base URL** : Saisissez `http://ollama:11434`. 
  * *Pourquoi cette URL ?* Au lieu de mettre une adresse IP classique, nous utilisons le nom du conteneur (`ollama`). Le réseau interne de Docker fait automatiquement le lien.
* **Chat Model Selection** : Choisissez `mistral` (ou `mistral:latest`).

![capture_d'écran_2026-09-23_182858.png](/chatbot/capture_d'écran_2026-09-23_182858.png)

### Vectorisation et Base de données (Le traitement documentaire)
* **Embedding Preference** : Laissez sur `AnythingLLM Embedder`. C'est le petit moteur intégré qui va découper et transformer vos documents en vecteurs mathématiques (coordonnées) sans avoir besoin d'un outil externe.
* **Vector Database** : Laissez sur `LanceDB`. C'est la base de données locale qui stocke ces vecteurs de façon sécurisée.

![capture_d'écran_2026-09-23_182959.png](/chatbot/capture_d'écran_2026-09-23_182959.png)

## 4. Configuration de l'Espace de Travail (Workspace)

Un espace de travail regroupe des documents spécifiques et des règles de comportement pour l'IA.

1. Ouvrez les paramètres de votre espace de travail (icône engrenage).
2. Allez dans l'onglet **Paramètres du chat**.
3. **Mode de chat** : Sélectionnez **Chat**. Ce mode est optimisé pour converser en s'appuyant sur les documents.
4. **Température LLM** : Réglez la valeur sur **0.2**. 
   * *Pourquoi ?* Une température proche de 0 rend l'IA très stricte et factuelle. Cela empêche les hallucinations (lorsque l'IA invente des procédures qui n'existent pas).
5. **Invite Système (System Prompt)** : C'est le cadre imposé au bot. Remplacez le texte par défaut par cette directive stricte :

> Tu es l'assistant du support informatique interne (niveau 0). 
> RÈGLE DE LANGUE : Ta langue par défaut est le français. Tu dois répondre en français, SAUF si l'utilisateur t'écrit dans une autre langue ou te demande explicitement de changer de langue.
> RÈGLE DE SUPPORT : Base-toi uniquement sur les documents fournis. N'invente aucune manipulation ni procédure non documentée.
> Si la réponse ne figure pas dans tes documents ou si la solution proposée échoue, indique clairement : « Je ne trouve pas la solution dans la documentation. Merci d'ouvrir un ticket d'incident. »

![capture_d'écran_2026-09-23_191050.png](/chatbot/capture_d'écran_2026-09-23_191050.png)

## 5. Importation de la Documentation (RAG)

Pour que l'IA puisse répondre, elle a besoin de connaissances. Nous utilisons des fiches de procédures (format texte, Markdown ou PDF). Vous pouvez télécharger un exemple comme celui ci : 
[llm_procedure.txt](/chatbot/llm_procedure.txt)


1. Cliquez sur le bouton **Télécharger un document** dans l'interface principale.
![capture_d'écran_2026-09-23_191236.png](/chatbot/capture_d'écran_2026-09-23_191236.png)
2. Glissez-déposez votre fichier (ex: [llm_procedure.txt](/chatbot/llm_procedure.txt)) dans la fenêtre.
3. Cochez le fichier dans la liste **Mes documents**.
4. Cliquez sur **Déplacer vers l'espace de travail** (le fichier passe dans la colonne de droite).
5. Cliquez sur **Save and Embed** en bas à droite.
   * *Que se passe-t-il ici ?* Le document est découpé en paragraphes, analysé par l'Embedder, et sauvegardé dans LanceDB. Le système est maintenant capable de retrouver le bon paragraphe en fonction de la question posée.

![capture_d'écran_2026-09-23_191605.png](/chatbot/capture_d'écran_2026-09-23_191605.png)

## 6. Tests et Validation

Pour vous assurer du bon fonctionnement, lancez un nouveau fil de discussion (**New Thread**) et effectuez trois tests :

1. **Test d'exactitude :** Demandez `J'ai oublié mon mot de passe`. Le bot doit lister les étapes exactes de votre document et afficher ses sources.
2. **Test de l'escalade :** Précisez que la procédure Wi-Fi a échoué. Il doit vous indiquer la bonne catégorie pour ouvrir un ticket.
![capture_d'écran_2026-09-23_191729.png](/chatbot/capture_d'écran_2026-09-23_191729.png)
3. **Test anti-hallucination (Hors-sujet) :** Demandez comment déclarer des congés. Le bot doit refuser de répondre et proposer d'ouvrir un ticket, prouvant qu'il respecte la restriction imposée par son prompt et sa température.

## 7. Gestion des ressources (Mettre en pause le chatbot)

L'intelligence artificielle (surtout le modèle Mistral via Ollama) consomme beaucoup de ressources CPU et de mémoire RAM. Il est donc recommandé d'éteindre les conteneurs lorsque vous ne les utilisez pas.

Grâce aux "volumes" que nous avons configurés dans le fichier `docker-compose.yml`, **aucune donnée ne sera perdue** (ni vos documents, ni votre configuration) lorsque vous éteignez le système.

### Pour éteindre (mettre en pause)

Ouvrez votre terminal et placez-vous dans le dossier de votre projet :
```bash
    cd ~/chatbot-local
    docker compose stop
```
*Cette commande arrête les conteneurs sans les détruire. Votre machine retrouve toutes ses ressources.*

### Pour redémarrer (reprendre)

Lorsque vous souhaitez utiliser à nouveau le chatbot, retournez dans le dossier et relancez les conteneurs :
```bash
    cd ~/chatbot-local
    docker compose start
```
*Le démarrage est quasi-instantané et vous retrouverez votre espace de travail exactement tel que vous l'avez laissé.*

*(Alternative : Si vous avez redémarré complètement votre ordinateur physique, utilisez plutôt `docker compose up -d` pour tout réinitialiser proprement).*

