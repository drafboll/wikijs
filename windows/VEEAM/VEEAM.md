---
title: Configuration VEEAM
description: 
published: 1
date: 2026-09-25T15:55:04.133Z
tags: 
editor: markdown
dateCreated: 2025-06-27T12:49:07.965Z
---

VEEAM est installé 

### Déploiement du serveur Veeam

- **Nom de la VM** : `SVR-VEEAM-01`
- **Adresse IP** : `192.168.50.10`
- **Hyperviseur utilisé** : `ESXi-1`
- **Stockage de la VM** :
     
    → Datastore local de **2 To** configuré en **RAID 0**, utilisé pour :
    
    - Le système d’exploitation du serveur Veeam
    - Les sauvegardes locales

---

### Configuration logicielle

- **Application installée** : Veeam Backup & Replication
- Interface graphique accessible
- Le serveur est reconnu dans “Managed Servers”

---

### Infrastructure de sauvegarde

- **vCenter intégré** dans Veeam (`192.168.10.112`)
- Toutes les **VMs de l’environnement VMware** sont visibles
- Inventaire synchronisé avec succès

---

### Tâches de sauvegarde en place

### Job planifié (quotidien)

- **Nom du job** : `VCENTER`
- **Type** : Sauvegarde **quotidienne automatique**
- **Contenu** : Sauvegarde groupée des VMs essentielles
- **Statut** : ✅ Fonctionnel et stable

### Jobs ponctuels (sauvegarde unique)

- **Graylog (1 fois)**
- **Kubernetes (1 fois)**
- **Vcenter (1 fois)**
- Objectif : effectuer une **sauvegarde manuelle initiale** de certaines VMs

---

### Intégration du cloud – Amazon S3

**Fait avec succès**

- Création d’un bucket AWS S3 : **`veeam-demo-2025-alphatech`**
- Région : **USA Est (N. Virginia) – `us-east-1`**
- Configuration dans Veeam :
    - Ajout d’un **Object Storage Repository**
    - Authentification via un utilisateur IAM : `veeam-user`
    - Clé d’accès et clé secrète configurées
- Veeam voit désormais le compartiment S3