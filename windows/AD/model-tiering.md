---
title: Installation Model-Tiering
description: 
published: 1
date: 2026-09-25T15:53:30.199Z
tags: 
editor: markdown
dateCreated: 2025-06-18T06:48:46.923Z
---

# Active Directory : mettre en place un système de tiers (tiering)

## Le tiering c’est quoi ?

Le modèle 3 tiers (tiering) consiste à cloisonner l’Active Directory en trois niveaux de sensibilité :
 
* **Tier 0** : Contrôleurs de domaine, PKI — le plus critique.
* **Tier 1** : Serveurs applicatifs, de fichiers, bases de données.
* **Tier 2** : Postes utilisateurs, imprimantes, smartphones...

Chaque niveau doit être isolé des autres. Un compte T1 ne doit pas administrer un élément T0, et inversement. Des bastions (PAW) sont recommandés.
![schéma.png](/model-tiering/schéma.png)

Il est aussi essentiel d'utiliser :

* Le MFA
* Le mode Restricted Admin pour RDP
* Le groupe Protected Users

L’[ANSSI propose un guide complet](https://www.ssi.gouv.fr/guide/recommandations-pour-ladministration-securisee-des-si-reposant-sur-active-directory/) à ce sujet.

---

## Qu’est-ce que l’on va voir dans ce tutoriel

Ce tutoriel se concentre sur deux GPO simples à implémenter :

1. L’automatisation des droits d’administration pour les serveurs T1
2. Le blocage des connexions non autorisées aux serveurs T1

---

## Organisation de l’Active Directory

1. Créer une OU `Comptes_d'Administration` contenant `T0`, `T1` et `T2` pour les comptes admin
2. Créer une OU `Serveurs` avec sous-OU `T0`, `T1` et `T2`
3. Ne pas déplacer les contrôleurs de domaine : ils restent dans l’OU `Domain Controllers`
4. Créer une OU `Ressources` → `Tiering_Model` → `T0`, `T1` et `T2` pour organiser les groupes d’administration
![addc.png](/model-tiering/addc.png)
Convention de nommage des groupes T1 : `GG_T1_ADM_Serveur`

Pour faire un test vous pouvez crer le groupe T1, un compte administrateur T1 et le placer dans ce groupe et deplacer un serveur dans l'OU T1 serveur.

---

## GPO 1 : Automatiser les administrateurs T1

**Objectif :** Ajouter automatiquement le groupe `GG_T1_ADM_Serveur` aux administrateurs locaux des serveurs T1.

### Étapes :

1. Créer une nouvelle GPO (ex. `T1-Admins`) et l’éditer
2. Aller dans : `Configuration ordinateur` → `Préférences` → `Paramètres du Panneau de configuration` → `Utilisateurs et groupes locaux`
3. Ajouter un **Groupe local** :

   * Action : `Mettre à jour`
   * Groupe : `Administrateurs (intégré)`
   * Membre : `GG_T1_ADM_Serveur`
![gpo_adm.png](/model-tiering/gpo_adm.png)

4. Lier la GPO à l’OU `Serveurs/T1`

Une fois appliquée, vérifier que le groupe local a bien été mis à jour et que les comptes peuvent ouvrir une session.

---

## GPO 2 : Autoriser seulement les administrateurs T1 à se connecter au tier 1

**Objectif :** Empêcher les comptes T0 (Admins du domaine, etc.) de se connecter aux serveurs T1.

### Étapes :

1. Créer une nouvelle GPO (ex. `T1_ADM_ONLY`) et l’éditer

2. Aller dans : `Configuration ordinateur` → `Paramètres Windows` → `Paramètres de sécurité` → `Stratégies locales` → `Attribution des droits utilisateur`
![adm_only.png](/model-tiering/adm_only.png)
3. Configurer les paramètres suivants :

   * `Interdire l’accès à cet ordinateur à partir du réseau`
   * `Interdire l’ouverture de session locale`
   * `Interdire l’ouverture de session en tant que service`
   * `Interdire l’ouverture de session en tant que tâche`
   * `Interdire l’ouverture de session par les services Bureau à distance`

   **Groupes à interdire :**

   * `Admins du domaine`
   * `Administrateurs de l’entreprise`
   * `Administrateurs du schéma`

4. Lier la GPO à l’OU `Serveurs/T1`
![lier_gpo.png](/model-tiering/lier_gpo.png)
5. Forcer une mise à jour avec `gpupdate`

6. Tester : les comptes T0 ne doivent plus pouvoir accéder aux serveurs T1 (ni RDP, ni partage C\$, ni observateur d’événements)

## Suite

Faire de meme avec les autre T0 et T2 en focntion des serveurs d'infrastructure pour le T0 et pour les devices / postes clients pour le T2

---

## Conclusion

Ce tutoriel montre les premières étapes vers un modèle tiering sécurisé dans Active Directory. Pour aller plus loin :

* Implémentez un **silo d’authentification** pour le Tier 0
* Déployez [**HardenAD**](https://github.com/cym13/HardenAD) pour automatiser la création des groupes et GPO via PowerShell
* Utilisez un LAB pour tester avant production

> ⚠️ Ce modèle nécessite une rigueur et une documentation précise pour éviter les erreurs de cloisonnement

