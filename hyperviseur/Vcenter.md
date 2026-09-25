---
title: VCenter / VSan / PCA
description: 
published: 1
date: 2026-09-25T15:42:38.092Z
tags: 
editor: markdown
dateCreated: 2025-06-27T12:43:13.300Z
---

# Déploiement ESXi, vSAN & Stratégie de Stockage – Cluster Alpha
 
## Infrastructure déployée

- **Serveurs physiques (ESXi)** :
    - `ESX01` – IP : `192.168.10.110`
    - `ESX02` – IP : `192.168.10.111`
- **WITNESS (vSAN)** : `192.168.10.113`
- **vCenter** : `192.168.10.12`
- **Accès iDRAC** :
    - `ESX01` : `192.168.10.100`
    - `ESX02` : `192.168.10.101`

---

# Déploiement ESXi, configuration vSAN et stratégie de stockage

## Infrastructure physique

Chaque hôte ESXi (ESX01 & ESX02) dispose de la configuration matérielle suivante :

- **4 disques SSD SAS de 900 Go**
    - 2 disques sont utilisés pour **le cache vSAN**
    - 2 disques sont utilisés pour **le stockage principal vSAN**
- **2 disques HDD SAS de 600 Go**
    - utilisés pour **le stockage principal vSAN**
- **2 disques HDD SAS de 2 To**
    - utilisés pour héberger les **ISOs, Veeam, vSphere et données NAS**

![image_(9).png](/vcenter/image_(9).png)

- Tous les disques sont passés en **mode HBA** via le contrôleur **PERC H730 Mini**, en configurant manuellement le mode “HBA” dans l’utilitaire BIOS du contrôleur RAID.

---

## Installation et configuration des ESXi

- Les hyperviseurs ESX01 (192.168.10.110) et ESX02 (192.168.10.111) sont déployés dans un **cluster vCenter nommé Cluster-01**.
- Les deux hôtes sont correctement reliés à :
    - **Réseau de management** : 192.168.10.x
    - **Réseau vSAN** : 192.168.11.x
    - **Réseau vMotion** : 192.168.12.x

> Ces réseaux sont isolés via des VLANs et attribués à des groupes de ports distincts
> 

![image_(10).png](/vcenter/image_(10).png)

---

## Configuration vSAN

La configuration vSAN a été réalisée comme suit :

## 1. Activation vSAN HCI

Dans la vue du cluster :

- Sélection de l’option **vSAN HCI** (HCI cluster standard)
- Mode **“vSAN à un seul site”** sélectionné (pas d’extension multi-site)

![image_(11).png](/vcenter/image_(11).png)

## 2. Services vSAN activés

- **Efficacité d’utilisation de l’espace** : Aucun
- **Chiffrement** : Désactivé
- **Redondance réduite** : Autorisée
- **Support RDMA** : Désactivé (non nécessaire ici)

![image_(12).png](/vcenter/image_(12).png)

## 3. Réclamation des disques

Dans l’assistant de configuration vSAN :

- Les disques sont affectés manuellement :
    - Les **SSD de 900 Go** sont affectés en :
        - 2x en **cache**
        - 2x en **stockage principal**
    - Les **HDD de 600 Go** sont également utilisés en **stockage principal**

> Répartition équilibrée sur chaque hôte pour assurer tolérance aux pannes.
> 

![image_(13).png](/vcenter/image_(13).png)

## 4. Sélection de l’hôte témoin

Le **vSAN Witness Appliance** est déployé sur une **machine virtuelle (192.168.10.113)** hébergée sur le **datastore local de l’ESX01**, distinct du cluster vSAN.

![image_(14).png](/vcenter/image_(14).png)

> Cette VM ne participe pas au stockage, elle assure uniquement le quorum pour les situations de split-brain (perte d’un hôte)
> 

![image_(15).png](/vcenter/image_(15).png)

---

## Stratégie de stockage vSAN personnalisée

Création d’une stratégie de stockage dans vCenter, appliquée aux machines critiques :

- **Nom** : `VSAN Dual-Site Alpha`
- **Tolérance aux pannes** : 1 panne
- **Règle** : RAID-1 (mise en miroir)
- **Composants** :
    - Réplication des objets sur les 2 nœuds physiques
    - Témoin placé sur la VM Witness
- **Utilisation** :
    - Toutes les VMs critiques du cluster utilisent cette stratégie par défaut

![image_(16).png](/vcenter/image_(16).png)

# Bascule automatique d’une VM critique via HA et vSAN

## Objectif

Démontrer la capacité du cluster vSphere à assurer la **continuité de service** en cas de défaillance d’un hôte physique, via la **relocalisation automatique de la VM `SVR-ZABBIX-01`** sur un autre nœud grâce à vSAN et HA.

---
![capture_d'écran_2025-07-12_203611.png](/vcenter/capture_d'écran_2025-07-12_203611.png)
## 1. État initial de `SVR-ZABBIX-01`

- Hébergée sur `ESX02` (192.168.10.111)
- Connectée au réseau `SUPERVISION` et au **datastore vSAN**
- Zabbix actif sur `http://192.168.30.10`
![capture_d'écran_2025-07-12_203716.png](/vcenter/capture_d'écran_2025-07-12_203716.png)

![capture_d'écran_2025-07-12_203744.png](/vcenter/capture_d'écran_2025-07-12_203744.png)
## 2. Simulation d’une panne de `ESX02`

- Arrêt volontaire sans mise en maintenance
- Avertissement : risque de coupure des VMs hébergées

![capture_d'écran_2025-07-12_203804.png](/vcenter/capture_d'écran_2025-07-12_203804.png)

---

## 3. Détection de la panne dans vCenter

- L’hôte `192.168.10.111` passe en **"Pas de réponse"**
- Toutes les VMs de cet hôte apparaissent **déconnectées**

![capture_d'écran_2025-07-12_203904.png](/vcenter/capture_d'écran_2025-07-12_203904.png)

---

## 4. Redémarrage automatique de la VM sur `ESX01`

- Bascule HA automatique
- `SVR-ZABBIX-01` est redémarrée sur `ESX01` sans intervention manuelle

![capture_d'écran_2025-07-12_203921.png](/vcenter/capture_d'écran_2025-07-12_203921.png)

---

## 5. Confirmation sur l’interface d’`ESX01`

- La VM est bien présente et en état "Normale"
- Aucun changement réseau ou stockage détecté

![capture_d'écran_2025-07-12_203959.png](/vcenter/capture_d'écran_2025-07-12_203959.png)

---

## 6. Vérification finale du service Zabbix

- L’interface Zabbix reste accessible à `http://192.168.30.10`
- Aucune alerte critique, la surveillance reprend normalement

![capture_d'écran_2025-07-12_204041.png](/vcenter/capture_d'écran_2025-07-12_204041.png)

---

## Conclusion

La démonstration confirme que :

- Le stockage partagé **vSAN** permet à n’importe quel hôte du cluster d'accéder aux VMs.
- Le mécanisme **vSphere HA** redémarre automatiquement les VMs critiques sur un autre hôte en cas de panne.
- Le service Zabbix est maintenu **sans perte durable de disponibilité**, prouvant la **résilience de l’infrastructure**.
