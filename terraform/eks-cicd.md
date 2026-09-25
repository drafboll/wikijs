---
title: Infrastructure EKS Sécurisée & CI/CD GitLab
description: 
published: 1
date: 2026-09-25T15:51:26.095Z
tags: 
editor: markdown
dateCreated: 2026-03-02T12:17:40.406Z
---

# Infrastructure EKS Sécurisée & CI/CD GitLab

Ce projet déploie une architecture Kubernetes complète sur AWS de manière sécurisée et automatisée.

## Liens Utiles
* **Dépôt GitLab** : [https://gitlab.com/indiogroup/terraform-eks](https://gitlab.com/indiogroup/terraform-eks)
![capture_d'écran_2026-03-02_140247.png](/eks_cicd/capture_d'écran_2026-03-02_140247.png)
## Structure Détaillée du Projet et Rôle des Composants

L'architecture est découpée de manière modulaire afin de séparer les responsabilités (Réseau, Calcul, Données, Automatisation), suivant les meilleures pratiques de l'**Infrastructure as Code**.
 
---

### 1. Le Socle Réseau : `/modules/core-compute`
Ce module constitue la fondation de toute l'infrastructure. Il isole l'environnement dans un **VPC (Virtual Private Cloud)** dédié pour garantir une étanchéité totale.

* **VPC & Segmentation** : Découpage du réseau en sous-réseaux publics (pour l'accès externe) et privés (pour la sécurité des données).
* **Passerelles (IGW & NAT)** : L'Internet Gateway gère le flux entrant, tandis que la NAT Gateway permet aux ressources privées (nœuds EKS) de communiquer vers l'extérieur sans être exposées directement sur Internet.
* **Sécurité Globale (Security Groups)** : Définition des règles de pare-feu initiales pour autoriser uniquement les flux indispensables (HTTP, HTTPS, SSH).

### 2. L'Intelligence Kubernetes : `cluster_eks.tf`
Ce fichier orchestre le déploiement du service Kubernetes managé d'AWS (EKS).

* **Control Plane** : Configuration de l'API Kubernetes gérée par AWS pour garantir une haute disponibilité du cluster.
* **Node Groups** : Définition des "Worker Nodes" (instances EC2) qui exécutent les conteneurs. Ils sont configurés en **Auto Scaling** pour adapter la puissance de calcul à la charge réelle.
* **Versionnage & Stabilité** : Fixation de la version de Kubernetes (ex: 1.34) pour assurer la pérennité et la compatibilité des API applicatives.

### 3. La Persistance des Données : `rds.tf`
Gestion de la base de données relationnelle MySQL via le service managé Amazon RDS.

* **Isolation Critique** : La base est placée dans un "Subnet Group" strictement privé, la rendant invisible et inaccessible depuis l'extérieur du VPC.
* **Filtrage des Flux (Port 3306)** : Une règle de sécurité spécifique (`allow_eks_to_rds`) restreint l'accès à la base : seul le Cluster EKS possède l'autorisation de s'y connecter.
* **Injection de Secrets** : Utilisation de la variable `db_password` injectée dynamiquement par GitLab, garantissant qu'aucun mot de passe n'est écrit dans le code source.

### 4. La Couche Applicative : `wordpress.tf`
Déploiement des ressources finales via le provider Kubernetes de Terraform.

* **Deployments & Pods** : Orchestration des conteneurs applicatifs WordPress sur les nœuds du cluster.
* **Liaison Dynamique** : Injection des paramètres de connexion (DB_HOST, DB_USER, DB_PASSWORD) pour lier l'application à l'instance RDS de manière sécurisée.
* **Exposition via Load Balancer** : Création d'un répartiteur de charge AWS pour fournir une URL d'accès stable et sécurisée aux utilisateurs finaux.

### 5. Le Cœur DevOps : `.gitlab-ci.yml`
Ce fichier centralise l'automatisation de tout le cycle de vie de l'infrastructure.

* **Pipeline Multi-stages** : Enchaînement rigoureux des étapes de validation (`validate`), de planification (`plan`), et de déploiement (`apply`).
* **Gestion du State (Backend HTTP)** : Stockage distant et sécurisé du fichier `terraform.tfstate` dans GitLab, permettant un travail collaboratif sans risque de conflit.
* **Sécurité des Jobs** : Utilisation de tokens temporaires (`CI_JOB_TOKEN`) pour authentifier les actions du Runner sans exposer de clés d'accès permanentes.

## Rapport de Déploiement Industriel : Cluster EKS & RDS via GitLab CI/CD

Ce document détaille la mise en œuvre d'une infrastructure cloud sécurisée sur AWS, entièrement pilotée par le code (IaC) et automatisée par une chaîne de déploiement continu.

---

### 1. Stratégie de Gestion des Sources (Git Workflow)
L'organisation de la collaboration repose sur une séparation stricte des environnements via les branches Git.

* **Branche `dev`** : Bac à sable technique pour déclencher les premiers tests du pipeline.
* **Branche `main`** : État de production, accessible uniquement après succès des tests sur `dev`.
* **Le rôle du `.gitignore`** : Indispensable pour la sécurité, il empêche l'envoi de fichiers sensibles ou inutiles vers GitLab.
    * **Sécurité** : Il exclut les fichiers de secrets comme `*.tfvars`, `terraform.tfstate` (car géré par le backend HTTP).
    * **Propreté** : Il ignore le dossier `.terraform/` qui contient les plugins volumineux téléchargés lors du `init`.
    
Le Workflow de Développement (Git Flow)
Le processus commence sur une branche isolée (`dev`) pour garantir que la branche principale (`main`) reste stable et sécurisée. 

* **Validation en local** : Les modifications sont testées avant d'être poussées.
* **Synchronisation et Fusion** : Une fois le pipeline validé sur `dev`, nous effectuons un `merge` vers `main` pour déclencher le déploiement de production.

**Preuve du cycle de déploiement en terminal :**
```bash
# Envoi des corrections sur la branche de développement
git add .gitlab-ci.yml
git commit -m "Fix: Add backend configuration to CI/CD"
git push origin dev
```
![capture_d'écran_2026-03-02_141319.png](/eks_cicd/capture_d'écran_2026-03-02_141319.png)

Fusion vers la branche principale après validation
```bash
git checkout main
git merge dev
git push origin main
```
![capture_d'écran_2026-03-02_141542.png](/eks_cicd/capture_d'écran_2026-03-02_141542.png)

---

### 2. Configuration Avancée du Pipeline `.gitlab-ci.yml`
### Configuration du Runner Privé
Pour garantir une isolation totale et une rapidité d'exécution, nous avons déployé un **Runner Docker** nommé `docker-indio` sur une machine Debian dédiée.
* **Exécuteur** : Docker (permet l'usage d'images éphémères).
* **Tag utilisé** : `docker-indio` pour cibler nos ressources propres.
![capture_d'écran_2026-03-02_150039.png](/eks_cicd/capture_d'écran_2026-03-02_150039.png)

### pipeline automatisé
```yaml
default:
  tags:
    - docker-indio # Utilisation du runner Debian-Docker configuré

image:
  name: hashicorp/terraform:latest
  entrypoint: [""] # Indispensable pour l'exécution des scripts Shell

variables:
  TF_ADDRESS: "${CI_API_V4_URL}/projects/${CI_PROJECT_ID}/terraform/state/default"
  # Configuration automatique de l'authentification du State
  TF_HTTP_ADDRESS: ${TF_ADDRESS}
  TF_HTTP_LOCK_ADDRESS: "${TF_ADDRESS}/lock"
  TF_HTTP_LOCK_METHOD: "POST"
  TF_HTTP_UNLOCK_ADDRESS: "${TF_ADDRESS}/lock"
  TF_HTTP_UNLOCK_METHOD: "DELETE"
  TF_HTTP_USERNAME: "gitlab-ci-token"
  TF_HTTP_PASSWORD: ${CI_JOB_TOKEN}

before_script:
  - terraform init # Initialisation simplifiée grâce aux variables d'environnement

stages:
  - validate
  - plan
  - apply
  - destroy

validate:
  stage: validate
  script:
    - terraform validate

plan:
  stage: plan
  script:
    - terraform plan -out=tfplan
  artifacts:
    paths:
      - tfplan # Passage du plan certifié au job apply

apply:
  stage: apply
  script:
    - terraform apply -auto-approve tfplan
  when: manual # Validation humaine obligatoire avant modification AWS

destroy:
  stage: destroy
  script:
    - terraform destroy -auto-approve
  when: manual # Sécurité pour le déprovisionnement total
```

* **Image & Entrypoint** : Nous utilisons l'image `hashicorp/terraform:latest`. Un point critique a été de vider l'entrypoint (`entrypoint: [""]`) pour permettre au Runner GitLab d'exécuter ses propres scripts Shell, contournant ainsi les limitations de l'image officielle.
* **Le Backend HTTP (Terraform State)** : L'état de l'infrastructure (`terraform.tfstate`) n'est plus stocké localement. Il est centralisé sur GitLab via une API HTTP. Cela permet :
    * Le verrouillage du State (Locking) pour éviter que deux déploiements ne se chevauchent.
    * La persistance des données entre les différents jobs du pipeline.

**Le pipeline GitLab CI/CD est le chef d'orchestre de l'infrastructure. Chaque stage a un rôle critique pour garantir la stabilité de l'environnement AWS** :

### 1️⃣ Validate : La Ceinture de Sécurité Syntaxique
Avant toute interaction avec AWS, le job `terraform validate` vérifie la cohérence du code.
* **Rôle** : S'assurer que les variables sont correctement déclarées, que les modules sont bien appelés et que la syntaxe HCL est parfaite.
* **Intérêt** : Éviter de lancer un Runner pour rien et détecter les erreurs humaines avant qu'elles ne puissent impacter le Cloud.

### 2️⃣ Plan : La Simulation Prédictive (Artefact `tfplan`)
Le job `terraform plan -out=tfplan` compare l'état actuel de votre Cloud avec votre code cible.
* **Artefact Crucial** : Le résultat est sauvegardé dans un fichier binaire `tfplan`. C'est une **garantie de sécurité** : le stage suivant (Apply) utilisera *uniquement* ce fichier. Cela empêche qu'une modification faite dans le code entre le "Plan" et le "Apply" ne soit déployée sans avoir été vue.
* **Visibilité** : Il liste précisément les ressources qui seront ajoutées, modifiées ou supprimées (ex: "40 resources to add").

### 3️⃣ Apply : Le Déploiement Réel (Validation Humaine)
Ce stage exécute la création des ressources sur AWS.
* **Trigger Manuel (`when: manual`)** : C'est le "bouton rouge". Un ingénieur DevOps doit vérifier les logs du "Plan" précédent, s'assurer que les coûts prévus sont corrects, puis cliquer sur "Play".
* **Exécution** : Le Runner utilise le `tfplan` pour monter l'infrastructure (VPC, EKS, RDS) en environ 15 minutes.
![capture_d'écran_2026-03-02_142954.png](/eks_cicd/capture_d'écran_2026-03-02_142954.png)

### 4️⃣ Destroy : (Nettoyage, Sécurité & FinOps)
Le stage `destroy` est souvent sous-estimé, mais il est **le plus important** pour la santé financière et la propreté de votre compte AWS.
![capture_d'écran_2026-03-02_145215.png](/eks_cicd/capture_d'écran_2026-03-02_145215.png)

#### Maîtrise des Coûts (Approche FinOps)
Sur AWS, vous payez à l'usage. Un cluster EKS coûte environ **0,10 $ par heure** de frais fixes, sans compter les instances EC2, les disques EBS et surtout la **NAT Gateway**, qui est l'un des composants les plus chers par heure.
* **Automatisation du Nettoyage** : En un clic, le job `terraform destroy -auto-approve` garantit que **100% des ressources** créées sont supprimées. Cela évite les "ressources orphelines" (comme une IP élastique ou un volume disque oublié) qui continuent de facturer inutilement.
![capture_d'écran_2026-03-02_150641.png](/eks_cicd/capture_d'écran_2026-03-02_150641.png)

#### Sécurité et Éphémérité
Dans un cadre de test ou de TP, laisser une infrastructure active sans surveillance est une faille de sécurité.
* **Réduction de la surface d'attaque** : Si le projet n'est pas utilisé, il ne doit pas exister. Le `destroy` permet de supprimer toute trace de l'infrastructure, rendant toute tentative d'intrusion impossible.
* **Preuve de Reproductibilité** : Cela prouve que votre code Terraform est parfait. Si vous pouvez "Destroy" et "Apply" à nouveau sans erreur, c'est que votre Infrastructure as Code est robuste.

#### Fonctionnement Technique
* **Logique Inverse** : Terraform lit le fichier `terraform.tfstate` (stocké sur le Backend HTTP GitLab) pour connaître l'ordre inverse des dépendances. Il supprimera d'abord les Pods, puis le cluster EKS, puis les sous-réseaux, et enfin le VPC.
* **Verrouillage (Locking)** : Pendant le `destroy`, le State GitLab est verrouillé. Personne d'autre ne peut modifier l'infrastructure, évitant ainsi de corrompre l'état.

### Analyse Technique du Déploiement Réel

Le job apply (Pipeline #2358503289) a duré 15 minutes et a créé 40 ressources sans aucune erreur.
![capture_d'écran_2026-03-02_145200.png](/eks_cicd/capture_d'écran_2026-03-02_145200.png)

**Liaison EKS-RDS** : La base de données wordpress_db a été créée avec succès et liée dynamiquement au déploiement Kubernetes via l'injection de variables d'environnement dans les Pods.

**Resource Map du Load Balancer** : Le trafic entrant sur le port 80 est correctement acheminé vers le Target Group tg-eks-apps qui pointe vers nos deux instances saines sur le port 30080 (NodePort).
![capture_d'écran_2026-03-02_145124.png](/eks_cicd/capture_d'écran_2026-03-02_145124.png)

**Résultat Final** : L'interface d'installation de WordPress est accessible et fonctionnelle, confirmant la bonne communication entre le Load Balancer, les Pods et la base de données RDS.
![capture_d'écran_2026-03-02_145049.png](/eks_cicd/capture_d'écran_2026-03-02_145049.png)

---

### 3. Exposition DNS & Stratégie de Nom de Domaine

Le passage d'une URL technique Amazon vers une URL métier (`wordpress.indiogroup.fr`) a été le point d'orgue de la sécurisation.

### Gestion du Certificat SSL (AWS ACM)
Pour obtenir le "cadenas vert", nous avons utilisé **AWS Certificate Manager** :
![capture_d'écran_2026-03-02_155356.png](/eks_cicd/capture_d'écran_2026-03-02_155356.png)

* **Validation DNS** : Un jeton CNAME unique a été généré par AWS et injecté dans la **Zone DNS OVH** pour prouver la propriété du domaine.
* **Statut "Issued"** : Une fois validé, le certificat devient utilisable par le Load Balancer pour déchiffrer le trafic HTTPS (port 443).

![capture_d'écran_2026-03-02_155440.png](/eks_cicd/capture_d'écran_2026-03-02_155440.png)

### Configuration du Load Balancer (ALB)
Le fichier `lb.tf` a été configuré pour gérer la **Terminaison TLS** :
* **Redirection Automatique** : Toute requête arrivant en HTTP (port 80) sur `wordpress.indiogroup.fr` est automatiquement redirigée vers le port 443 (HTTPS).
* **Listener HTTPS** : Le port 443 utilise le certificat ACM pour sécuriser les échanges avant de transmettre le trafic au cluster EKS sur le port NodePort 30080.

![capture_d'écran_2026-03-02_155603.png](/eks_cicd/capture_d'écran_2026-03-02_155603.png)

### 4. Gestion Sécurisée des Secrets (DevSecOps)
Conformément aux exigences de sécurité INDIO (Zero Trust), aucun secret n'est présent dans le code source.

* **Variables d'environnement** : Les clés AWS (`AWS_ACCESS_KEY_ID`) et le mot de passe RDS (`TF_VAR_db_password`) sont configurés dans GitLab (Settings > CI/CD > Variables).
* **Masquage (Masking)** : Les variables sont marquées comme "Masked" pour être invisibles dans les logs du Runner.
* **Protection des branches** : Sur `dev`, les variables sont "Unprotected" pour permettre le test, alors qu'elles sont "Protected" sur `main` pour un contrôle maximal.
* **Preuve d'implémentation** :
    Dans rds.tf : la valeur n'apparaît jamais
    password = var.db_password
    Dans wordpress.tf : injection dynamique dans le conteneur
    value = var.db_password
    
![capture_d'écran_2026-03-02_141647.png](/eks_cicd/capture_d'écran_2026-03-02_141647.png)

---

### 5. Détail de l'Infrastructure Déployée
Le déploiement crée un écosystème complet et durci sur AWS.

* **Réseau (VPC)** : Segmentation en sous-réseaux publics et privés. Les instances Kubernetes n'ont pas d'adresse IP publique.
* **Calcul (EKS)** : Le cluster tourne en version 1.34. Les nœuds sont répartis sur plusieurs zones de disponibilité pour la haute disponibilité.
* **Données (RDS)** : Une base MySQL isolée. Seul le groupe de sécurité d'EKS est autorisé à contacter la base sur le port 3306 (Micro-segmentation).
* **Exposition (ALB)** : Un Load Balancer frontal reçoit le trafic internet et le redirige vers les pods WordPress à l'intérieur du cluster.

---

### Retour d'Expérience Technique (Points bloquants résolus)
* **Erreur "sh" sur le Runner** : Résolue par la modification de l'entrypoint du conteneur Terraform.
* **Erreur "Credential Source"** : Résolue en ajustant les drapeaux "Protected" des variables GitLab pour la branche de développement.
* **Erreur "Address required"** : Résolue en configurant dynamiquement le backend HTTP dans le `before_script` du pipeline.

---

## Analyse de Sécurité – Positionnement RSSI INDIO

En tant que Responsable de la Sécurité des Systèmes d'Information (RSSI), j'ai audité l'infrastructure déployée. Ce projet répond aux exigences critiques de sécurité, de traçabilité et de souveraineté des données imposées par le référentiel INDIO.

---

### 1. Isolation Réseau et Micro-segmentation (Principe de Defense-in-Depth)
L'architecture réseau (VPC) a été conçue pour minimiser la surface d'attaque.

* **Segmentation Stricte** : Les nœuds du cluster EKS et l'instance de base de données RDS sont positionnés exclusivement dans des **sous-réseaux privés**. Ils ne possèdent aucune adresse IP publique, rendant toute tentative de connexion directe depuis Internet techniquement impossible.
* **Filtrage granulaire (Security Groups)** : Nous appliquons une politique de micro-segmentation. Par exemple, la règle `allow_eks_to_rds` restreint l'accès à la base de données : seul le flux provenant du groupe de sécurité du cluster EKS sur le port 3306 est autorisé.
* **Passerelle NAT sécurisée** : Les flux sortants des nœuds privés (pour les mises à jour) passent par une NAT Gateway, masquant ainsi l'identité réelle des serveurs internes.

### 2. Gestion des Identités et Moindre Privilège (IAM & Zero Trust)
Le projet applique le principe du moindre privilège pour chaque composant technique.

* **Rôles IAM Dédiés** : Au lieu d'utiliser des droits administrateurs larges, des rôles spécifiques (`EKSClusterRole`, `NodeGroupRole`) ont été créés avec des permissions restreintes aux actions strictement nécessaires.
* **Authentification Dynamique** : Le pipeline utilise des tokens temporaires (`CI_JOB_TOKEN`) pour s'authentifier auprès du backend GitLab, évitant l'usage de clés statiques à longue durée de vie dans le Runner.
* **Accès au Cluster** : L'accès administratif au cluster Kubernetes est protégé par une authentification AWS IAM intégrée, garantissant que seuls les utilisateurs autorisés peuvent exécuter des commandes `kubectl`.

### 3. Traçabilité, Audit et Cycle de Vie CI/CD
Chaque modification de l'infrastructure est documentée, testée et logguée de manière immuable.

* **Historique des Changements** : Le dépôt Git sert de "Source of Truth". Toute modification (ex: commit `9746acc`) est historisée avec l'identité de l'auteur.
* **Audit des Pipelines** : Le succès des étapes `validate` et `plan` (Pipeline #2358336709) constitue une preuve d'audit technique avant tout déploiement.
* **Validation Manuelle (Guardrail)** : L'étape `apply` est déclenchée manuellement, forçant une revue humaine du plan d'exécution Terraform avant toute modification de l'environnement de production.

### 4. Gestion des Secrets (Zéro Secret en Clair)
L'un des succès majeurs de ce TP est l'éradication totale des secrets dans le code source.

* **Variables Masquées (Masking)** : Les secrets tels que `AWS_SECRET_ACCESS_KEY` et `TF_VAR_db_password` sont stockés dans le coffre-fort de GitLab. Ils sont marqués comme **Masked**, ce qui signifie qu'ils n'apparaissent jamais en clair dans les logs du pipeline, même en cas d'erreur.
* **Injection par Environnement** : Le mot de passe de la base de données est injecté dynamiquement. Dans le fichier `rds.tf`, nous utilisons `var.db_password`, dont la valeur réelle n'est connue que par le Runner lors de l'exécution.
* **Protection contre les fuites** : L'usage du fichier `.gitignore` a été audité pour s'assurer que les fichiers d'état locaux (`.tfstate`) ou les clés SSH privées (`.pem`) ne soient jamais téléversés sur le serveur.

### 5. Protection des Données en Transit (Chiffrement TLS/SSL)

L'exposition de l'application WordPress sur le domaine wordpress.indiogroup.fr a été sécurisée selon les standards de cryptographie moderne.

* **Certificat Officiel (AWS ACM)** : Nous avons banni l'usage de certificats auto-signés. L'utilisation d'AWS Certificate Manager garantit une chaîne de confiance valide, certifiée par une autorité de certification reconnue.

* **Validation par DNS (Preuve de Détention)** : La sécurité de l'émission du certificat repose sur une validation DNS stricte via un jeton CNAME unique injecté chez le registraire (OVH). Cela prouve que seul le propriétaire légitime du domaine peut générer ce certificat.

* **Terminaison TLS sur l'Application Load Balancer (ALB)** : Le déchiffrement des flux s'effectue au point d'entrée du Cloud AWS (l'ALB). Cela permet d'inspecter et de filtrer le trafic avant qu'il n'atteigne les nœuds du cluster EKS, tout en déchargeant ces derniers de la consommation CPU liée au chiffrement.

* **Politique HSTS et Redirection Forcée** : Une règle d'Ingress impose une redirection systématique du port 80 (HTTP) vers le port 443 (HTTPS). Aucun flux ne circule en clair sur le réseau public, protégeant ainsi les identifiants de connexion WordPress contre les attaques de type "Man-in-the-Middle" (Ecoute réseau).

---

## Décision RSSI : GO

L'infrastructure démontre une maturité opérationnelle élevée. Les contrôles de sécurité (isolation, identités, traçabilité) sont intégrés nativement dans le code et le pipeline.

**Actions prioritaires pré-déploiement final :**
1. Activer la protection des branches pour `main` afin de rendre les variables de production (`Protected`) inaccessibles depuis les branches de test.
2. Mettre en place une rotation automatique des clés d'accès AWS tous les 90 jours.
3. Configurer les alertes de supervision sur les échecs de pipeline pour une réaction immédiate en cas de tentative de modification non autorisée.

---
© 2026 - Projet EKS Terraform - Alexandre FAYE