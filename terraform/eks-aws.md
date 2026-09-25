---
title: Déploiement d'un Cluster AWS EKS avec Terraform
description: 
published: 1
date: 2026-09-25T15:50:37.854Z
tags: 
editor: markdown
dateCreated: 2026-02-20T14:55:48.414Z
---

# Projet Infrastructure as Code : Déploiement d'un Cluster AWS EKS avec Terraform

## Introduction
Ce document détaille l'architecture et la procédure de déploiement d'une infrastructure cloud complète sur Amazon Web Services (AWS) à l'aide de Terraform. L'objectif de ce projet est de provisionner de manière automatisée un socle réseau sécurisé, un cluster Kubernetes managé (EKS), et d'y déployer une application web (Nginx) exposée via un Load Balancer.

> 💡 **Ressource du Projet :** Pour vous permettre de reproduire cette infrastructure et d'auditer notre code, l'intégralité des fichiers Terraform de ce projet (le répertoire racine et le *core module*) est téléchargeable directement en cliquant **[déploiement_cluster_eks-terraform.zip](/eks-terraform/déploiement_cluster_eks-terraform.zip)**.
{.is-success}
 
---

## Partie 1 : Architecture du Projet et Rôle des Fichiers

Le projet est divisé en deux grandes parties : un module réseau personnalisé (`core-compute`) et le déploiement racine qui orchestre le cluster EKS et les applications. L'état de cette infrastructure est sauvegardé de manière distante et sécurisée dans un bucket S3 nommé `tfstate-projet` situé dans la région `us-east-1`.

### A. Le Module de Base : `core-compute` (Réseau et Sécurité)
Ce module prépare le terrain de jeu. Il crée un réseau privé virtuel (VPC) isolé et sécurisé pour accueillir nos serveurs. 

* **`variables.tf`** : Définit les variables personnalisables comme le nom du projet (`nom_projet`) et le préfixe réseau (`prefix_reseau`).
* **`vpc.tf`** : Crée le Virtual Private Cloud (VPC), l'enveloppe réseau globale sur AWS avec un bloc d'adresses IP basé sur `10.2.0.0/16`.
* **`vpc_subnets_public.tf`** : Définit trois sous-réseaux publics répartis sur plusieurs zones de disponibilité. 
  * Il crée une Internet Gateway (IGW) pour permettre l'accès direct à Internet.
  * Les sous-réseaux contiennent le tag `"kubernetes.io/role/elb" = "1"`. C'est une balise indispensable pour qu'AWS EKS sache où placer les répartiteurs de charge publics.
* **`vpc_subnet_private.tf`** : C'est ici que réside la sécurité. Il crée des sous-réseaux privés pour héberger les nœuds Kubernetes. 
  * Ces machines n'ont pas d'IP publique. Pour qu'elles puissent communiquer vers l'extérieur, ce fichier déploie une **NAT Gateway** et une IP Élastique (EIP) dans le réseau public.
* **`sg.tf`** : Gère le pare-feu virtuel (Security Group) en autorisant le trafic entrant sur les ports SSH (22), HTTP (80) et HTTPS (443).
* **`ssh.tf`** : Génère dynamiquement une clé RSA de 4096 bits pour l'administration et sauvegarde la clé privée localement sous le nom `key.pem` avec des droits restreints (`0400`).
* **`instances.tf` & `data.tf`** : `data.tf` recherche dynamiquement la dernière image "Amazon Linux 2023". `instances.tf` déploie une machine de test (`t3.micro`) avec un serveur web Apache (`httpd`) installé automatiquement au démarrage.
* **`output.tf`** : Exporte les identifiants (VPC, sous-réseaux, Security Group) pour que le module racine puisse les utiliser.

### B. Le Déploiement Racine (Cluster EKS & Application)
Cette partie utilise le réseau créé précédemment pour y instancier la puissance de calcul Kubernetes. 

* **`main.tf`** : Fait appel au module `core-compute` en lui passant les variables du projet.
* **`cluster_eks.tf`** : Le cœur de l'orchestration. 
  * Il provisionne le "Control Plane" EKS en version `1.34`.
  * Il crée un groupe de nœuds de calcul (Auto Scaling Group) configuré pour avoir entre 1 et 3 instances `t3.micro`, placées strictement dans les sous-réseaux privés.
  * Il ouvre le port TCP `30080` pour autoriser le trafic entrant vers nos applications.
* **`role_eks.tf`** : Gère les permissions (IAM). Le rôle `EKSClusterRole` permet au Control Plane de gérer des ressources, tandis que le rôle `NodeGroupRole` autorise les machines EC2 à s'enregistrer au cluster et à télécharger des images de conteneurs.
* **`addons.tf`** : Installe les plugins vitaux pour EKS : `coredns` (résolution de noms interne), `vpc-cni` (réseau des Pods), et `kube-proxy`.
* **`lb.tf`** : Crée un Application Load Balancer (ALB) public. Il écoute sur le port `80` et redirige le trafic vers un Target Group ciblant le port `30080` de nos instances.
* **`auto_attachement.tf`** : Lie dynamiquement les machines créées par l'Auto Scaling Group de Kubernetes au Target Group du Load Balancer.
* **`provider.tf`** : Configure la connexion à AWS et génère dynamiquement un token d'authentification pour se connecter de manière sécurisée à l'API du nouveau cluster Kubernetes.
* **`nginx.tf`** : La couche applicative. Il utilise le provider Kubernetes pour déployer un serveur Nginx (2 réplicas) et l'exposer via un service de type `NodePort` sur le port `30080`.

---

## Partie 2 : Guide de Déploiement Pas à Pas

Suivez ces instructions pour déployer l'infrastructure complète.

### 1. Prérequis
* **Terraform**, **AWS CLI** et **kubectl** installés sur votre poste de travail.
* AWS CLI configuré (`aws configure`) avec des accès valides.
* Un bucket Amazon S3 nommé `tfstate-projet` créé au préalable sur votre compte AWS.

### 2. Initialisation
Placez-vous dans le dossier racine du projet et initialisez l'environnement de travail :
```bash
terraform init
```

### 3. Planification (Audit)
Vérifiez les actions que Terraform s'apprête à exécuter :
```bash
terraform plan
```

### 4. Déploiement
Lancez la création de l'infrastructure :
```bash
terraform apply -auto-approve
```

### 5. Configuration de l'accès au Cluster
Une fois le déploiement terminé, configurez `kubectl` pour qu'il puisse communiquer avec votre nouveau cluster EKS :
```bash
aws eks update-kubeconfig --region <votre_region> --name <nom_du_projet>-cluster
```
Vérifiez que vos nœuds de calcul sont opérationnels :
```bash
kubectl get nodes
```

### 6. Validation Applicative
1. Récupérez le nom de domaine (DNS) de votre Load Balancer généré par AWS.
2. Ouvrez cette adresse dans votre navigateur web.
3. Vous devriez voir s'afficher la page d'accueil par défaut de **Nginx**.

### 7. Destruction (Nettoyage)
Pour éviter des frais de facturation AWS inutiles une fois vos tests terminés, détruisez toutes les ressources créées :
```bash
terraform destroy -auto-approve
```