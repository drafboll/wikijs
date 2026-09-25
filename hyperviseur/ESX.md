---
title: Installation d'un ESX
description: 
published: 1
date: 2026-09-25T15:42:16.530Z
tags: 
editor: markdown
dateCreated: 2025-06-27T12:47:44.273Z
---

**Installation d'un hôte VMware ESXi 8.0.3**

Voici les étapes complètes pour installer un hôte VMware ESXi 8.0.3 sur un serveur physique. Cette procédure est basée sur l'installation réalisée sur des serveurs Dell PowerEdge R430 avec des disques configurés en mode HBA (non-RAID).
 
---

### 1. Préparation de l'installation

- Démarrer le serveur sur l'image ISO d'ESXi 8.0.3.
- Attendre le chargement de l'assistant d'installation.

### 2. Sélection du disque d'installation

- Choisir le disque sur lequel installer ESXi. Dans notre cas, un disque SAS SEAGATE ST600MM008 (558,91 Go).
    
![image_(17).png](/esx/image_(17).png)
    
- Noter que certains disques peuvent être préfixés par un `# Claimed by VMware vSAN`, ce qui signifie qu’ils sont déjà alloués pour vSAN.

### 3. Avertissements lors du scan du système

- L’installateur peut afficher deux avertissements :
    - **Legacy BIOS détecté** : préférer l’UEFI pour de meilleures performances.
    - **CPU non supporté officiellement** : il est possible de forcer l’installation, même si la configuration n’est pas officiellement prise en charge.
    
![image_(18).png](/esx/image_(18).png)
    

### 4. Choix de la partition

- Si un ESXi est déjà présent, l’assistant proposera :
    - Upgrade ESXi
    - Installer ESXi et conserver le datastore VMFS
    - **Installer ESXi et écraser le datastore VMFS** (option choisie ici).
    
![image_(19).png](/esx/image_(19).png)

### 5. Configuration du clavier

- Choisir la disposition du clavier : `French` dans notre cas.

### 6. Mot de passe root

- Définir un mot de passe root sûr et le confirmer.

### 7. Lancement de l’installation

- L’installation commence (ex. : "Partitioning disk for ESXi").
- Patienter jusqu'à 100 %.

![image_(20).png](/esx/image_(20).png)

### 8. Redémarrage et accès au serveur

- Une fois redémarré, l'écran d’accueil affiche :
    - Nom de l’hôte (ex. `ESX02`)
    - Adresse IP de gestion (ex. `192.168.10.111`)

![image_(21).png](/esx/image_(21).png)
### 9. Configuration réseau

- Appuyer sur `F2` pour accéder aux paramètres.
- Définir l’adresse IPv4 manuellement :
    - IP : `192.168.10.111`
    - Masque : `255.255.255.0`
    - Passerelle : `192.168.10.1`
- Définir aussi le VLAN si nécessaire et les DNS.

![image_(22).png](/esx/image_(22).png)

---

> Vous pouvez maintenant accéder à l’interface de gestion de votre ESXi via : https://192.168.10.111 et commencer la configuration dans vCenter.
>