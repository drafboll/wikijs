---
title: Création d'une Backup et restauration
description: 
published: 1
date: 2026-09-25T15:55:13.264Z
tags: 
editor: markdown
dateCreated: 2025-07-12T20:27:18.135Z
---

# Tutoriel PRA avec Veeam Backup & Replication

## Objectif
 
Mettre en œuvre un **Plan de Reprise d’Activité (PRA)** complet sur l’infrastructure VMware à l’aide de **Veeam Backup & Replication**, en réalisant :
- La sauvegarde d’une VM critique (`ZEX-SVR-ZABBIX-03`)
- Sa suppression simulée (incident)
- Sa restauration intégrale
- La validation finale de son bon fonctionnement
![capture_d'écran_2025-07-12_221804.png](/pra/capture_d'écran_2025-07-12_221804.png)
---

## Prérequis

- VM `ZEX-SVR-ZABBIX-03` installée et opérationnelle
- Veeam Backup & Replication installé (`SVR-VEEAM-01`)
- Repository de sauvegarde configuré
- Droits d’accès vCenter/ESXi
- Accès réseau entre Veeam et les hôtes VMware
![capture_d'écran_2025-07-12_221745.png](/pra/capture_d'écran_2025-07-12_221745.png)
---

## Étapes de sauvegarde avec Veeam

### 1. Création du job de sauvegarde

- Menu **Backup > Backup Job > Virtual machine > VMware vSphere**
- Nommer le job `backup pra`

![capture_d'écran_2025-07-12_221816.png](/pra/capture_d'écran_2025-07-12_221816.png)
![capture_d'écran_2025-07-12_221830.png](/pra/capture_d'écran_2025-07-12_221830.png)

---

### 2. Sélection de la VM cible

- Ajouter la VM `ZEX-SVR-ZABBIX-03` dans la liste des VMs à sauvegarder

![capture_d'écran_2025-07-12_221842.png](/pra/capture_d'écran_2025-07-12_221842.png)

---

### 3. Configuration du stockage

- Choisir le `Default Backup Repository`
- Politique de rétention : `7 jours`

![capture_d'écran_2025-07-12_221850.png](/pra/capture_d'écran_2025-07-12_221850.png)

---

### 4. Lancer la sauvegarde

- Lancer manuellement le job
- Suivre l’état du job dans Veeam : débit, taille, erreurs

![Suivi du job](./veeam_job_start.png)

---

### 5. Vérification de la sauvegarde réussie

- La sauvegarde se termine avec succès
- Taille sauvegardée : 8.3 Go, aucune erreur

![capture_d'écran_2025-07-12_221936.png](/pra/capture_d'écran_2025-07-12_221936.png)
![capture_d'écran_2025-07-12_222027.png](/pra/capture_d'écran_2025-07-12_222027.png)

---

## Simulation de sinistre : suppression de la VM

- Suppression volontaire de la VM depuis vCenter
- Cela simule une perte de VM sur incident

![capture_d'écran_2025-07-12_222045.png](/pra/capture_d'écran_2025-07-12_222045.png)

---

##  Restauration complète de la VM

### 1. Lancer une restauration complète

- Menu `Backups > Disk > clic droit sur la VM > Restore entire VM...`

![capture_d'écran_2025-07-12_222111.png](/pra/capture_d'écran_2025-07-12_222111.png)

---

### 2. Sélection du point de restauration

- Sélectionner la VM et le point le plus récent

![capture_d'écran_2025-07-12_222146.png](/pra/capture_d'écran_2025-07-12_222146.png)

---

### 3. Mode de restauration

- Choisir **Restore to the original location**

![capture_d'écran_2025-07-12_222202.png](/pra/capture_d'écran_2025-07-12_222202.png)

---

### 4. Suivi de la progression de la restauration

- Veeam commence la restauration
- Chargement du disque dur (8.3 Go) via proxy

![capture_d'écran_2025-07-12_222254.png](/pra/capture_d'écran_2025-07-12_222254.png)
![capture_d'écran_2025-07-12_222358.png](/pra/capture_d'écran_2025-07-12_222358.png)

---

### 5. Tâches vCenter liées à la restauration

- Vérification des tâches vCenter : reconfiguration, enregistrement, rechargement de la VM

![capture_d'écran_2025-07-12_222442.png](/pra/capture_d'écran_2025-07-12_222442.png)

---

## Validation du PRA

- Vérification dans vSphere : la VM `ZEX-SVR-ZABBIX-03` est à nouveau présente et allumée

![capture_d'écran_2025-07-12_222533.png](/pra/capture_d'écran_2025-07-12_222533.png)

- Connexion SSH ou console Web possible
- Vérification de l’état de l’OS et des services (Zabbix, Apache, etc.)

```bash
hostname
# → ZEX-SVR-ZABBIX-03
