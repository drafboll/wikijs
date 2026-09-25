---
title: Application Load Balancer avec Journalisation S3 et Réplication
description: 
published: 1
date: 2026-09-25T15:50:29.076Z
tags: 
editor: markdown
dateCreated: 2026-02-20T15:21:12.076Z
---

# Projet IaC : Application Load Balancer avec Journalisation S3 et Réplication

## Introduction
Ce projet déploie une architecture web hautement disponible sur AWS en utilisant Terraform. Il automatise la création de deux serveurs web Apache derrière un Application Load Balancer (ALB). 
L'objectif principal de ce laboratoire est de configurer la journalisation des accès (Access Logs) de l'ALB vers un bucket S3, et de mettre en place une règle de réplication inter-régions (CRR) pour sauvegarder ces journaux dans une seconde région AWS.
 
> 💡 Ressource du Projet : Pour vous permettre de reproduire cette infrastructure et d'auditer notre code, l'intégralité des fichiers Terraform de ce projet  est téléchargeable directement en cliquant ici : **[cloud-sec.zip](/eks-terraform/cloud-sec.zip)**
{.is-success}



---

## Partie 1 : Architecture et Rôle des Fichiers Terraform

Ce projet s'appuie sur le VPC par défaut d'AWS pour simplifier le déploiement. Voici le détail des composants :

* **`variables.tf` & `providers.tf`** : Définissent la région principale (`eu-west-1`, Irlande) et le numéro de groupe (`5-5esgi`). Le fichier `providers.tf` déclare également un alias (provider secondaire) pour la région de destination de la réplication (`eu-west-3`, Paris).
* **`vpc.tf`** : Récupère dynamiquement les informations du VPC par défaut et de ses sous-réseaux.
* **`instances.tf`** : 
  * Recherche la dernière image système (AMI) Amazon Linux 2023.
  * Crée un groupe de sécurité (Security Group) n'autorisant que le port HTTP (80) entrant.
  * Déploie deux instances EC2 (`t3.micro`) équipées d'un rôle IAM SSM pour une gestion sécurisée sans clé SSH.
  * Un script `user_data` installe Apache (`httpd`) au démarrage et configure une page d'accueil personnalisée pour identifier chaque serveur (Serveur A et Serveur B).
* **`alb.tf`** : 
  * Crée un Target Group pointant vers le port 80 des instances avec un "Health Check" sur `/index.html`.
  * Déploie l'Application Load Balancer public et son Listener. 
  * Active l'envoi automatique des logs d'accès de l'ALB vers le bucket S3 principal.
* **`s3.tf`** (Le cœur du projet) : 
  * Crée le bucket source en Irlande et le bucket de réplication à Paris, avec le *Versioning* activé (requis pour la réplication).
  * Déploie un rôle et une politique IAM autorisant S3 à lire les objets du bucket source et à les répliquer vers la destination.
  * Configure la règle de réplication avec la classe de stockage économique `ONEZONE_IA` pour les objets répliqués.
  * Ajoute une `bucket_policy` critique autorisant le compte de service AWS gérant les ALB (ex: `156460612806` pour `eu-west-1`) à écrire les logs dans le bucket.
* **`outputs.tf`** : Exporte l'URL DNS de l'ALB ainsi que les adresses IP publiques des deux serveurs.

---

## Partie 2 : Guide de Déploiement et de Test

### 1. Initialisation
Placez-vous dans le répertoire du projet contenant les fichiers Terraform et initialisez l'environnement de travail :
```bash
terraform init
```

### 2. Planification
Vérifiez l'ensemble des ressources qui vont être créées (VPC, EC2, ALB, S3, IAM Roles) :
```bash
terraform plan
```

### 3. Déploiement
Appliquez la configuration :
```bash
terraform apply -auto-approve
```
*Une fois terminé, Terraform affichera l'URL DNS de votre Load Balancer (ex: `alb_dns = Group5-5esgiLB-xxxx.eu-west-1.elb.amazonaws.com`).*

### 4. Validation du Load Balancer
1. Copiez l'URL DNS fournie par les *outputs* de Terraform.
2. Collez-la dans votre navigateur web.
3. Rafraîchissez la page plusieurs fois. Vous devriez voir le texte alterner entre **"Response coming from server A"** et **"Response coming from server B"**, prouvant que l'ALB répartit bien le trafic.

### 5. Validation des Logs S3 et de la Réplication
1. Connectez-vous à la console AWS et rendez-vous dans le service **S3**.
2. Ouvrez votre bucket source (`group5-5esgi-webserver-s3log`).
3. Vous devriez y trouver un dossier **AWSLogs/** contenant vos journaux d'accès (cela peut prendre jusqu'à 5 minutes pour apparaître).
4. Rendez-vous ensuite dans votre bucket de réplication (`group5-5esgi-webserver-s3log-replica`). 
5. Vérifiez que les logs ont bien été copiés automatiquement par AWS (la réplication peut prendre 3 à 5 minutes).

### 6. Nettoyage de l'infrastructure
Pour éviter toute facturation inutile (notamment pour les instances EC2, l'ALB et le stockage S3), détruisez les ressources une fois vos tests validés :
```bash
terraform destroy -auto-approve
```
*(L'option `force_destroy = true` configurée dans `s3.tf` permettra à Terraform de supprimer les buckets même s'ils contiennent des logs).*