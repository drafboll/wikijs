---
title: Configuration ADCONNECT
description: 
published: 1
date: 2026-09-25T15:54:00.042Z
tags: 
editor: markdown
dateCreated: 2025-06-27T12:38:04.208Z
---

# Configuration d’Azure AD Connect

## Objectif

Configurer Azure AD Connect afin de synchroniser les comptes utilisateurs, groupes et mots de passe depuis un **Active Directory local (On-Prem)** vers **Azure Active Directory (Azure AD)**, utilisé par Microsoft 365.

---
 
## Préparation de l’environnement

Avant l’installation d’AD Connect, plusieurs prérequis doivent être validés :

- ✅ Un domaine Active Directory fonctionnel
- ✅ Un serveur Windows membre du domaine, à jour, pour héberger Azure AD Connect
- ✅ Un compte administrateur local (Domain Admin) pour se connecter à l’AD
- ✅ Un compte Azure AD disposant du rôle **Administrateur global**
- ✅ Connexion Internet active
- ✅ Vérification du domaine personnalisé dans Azure AD (ex. `alpha-pro.fr`)

![image_(5).png](/adconnect/image_(5).png)

---

## Téléchargement d’Azure AD Connect

Le programme d’installation est disponible sur le site officiel de Microsoft :

🔗 [Télécharger Azure AD Connect](https://www.microsoft.com/en-us/download/details.aspx?id=47594)

---

## Lancement de l’installation

1. Exécuter le fichier `AzureADConnect.msi` sur le serveur prévu à cet effet.
2. Accepter les termes de licence.
3. Choisir le **mode d'installation** :

### ➤ Mode Express (recommandé)

- Synchronise **tous les utilisateurs et groupes** de l’AD local
- Synchronise les **mots de passe** vers Azure AD
- Configure la **réplication automatique** toutes les 30 minutes
- Configure un **SSO partiel** si les conditions sont remplies

*(Ajouter ici une capture de l’étape « mode Express »)*

> Ce mode est idéal pour une configuration rapide avec un seul domaine AD.
> 

---

## Authentification Azure AD

Saisir les identifiants d’un **compte Azure AD disposant du rôle Administrateur global**.

![image_(6).png](/adconnect/image_(6).png)
---

## Connexion à l’Active Directory local

Saisir les identifiants d’un **compte administrateur de domaine local** pour autoriser AD Connect à accéder à l'annuaire.

![image_(7).png](/adconnect/image_(7).png)

---

## Finalisation de la configuration

1. Vérifier le récapitulatif de configuration.
2. Laisser cochée l’option **Démarrer la synchronisation**.
3. Cliquer sur **Installer**.

![image_(8).png](/adconnect/image_(8).png)

---

## Vérification post-installation

Après l’installation :

- Les utilisateurs apparaissent dans Azure AD via le **portail M365** ([https://admin.microsoft.com](https://admin.microsoft.com/))
- Les synchronisations s'effectuent automatiquement
- L'outil **"Synchronization Service Manager"** permet de consulter les logs
- 

---

## Outils complémentaires

- **Azure AD Connect Health** (sur le portail Azure) : supervision de l'état de la synchronisation
- **PowerShell** : commande de synchronisation manuelle :

```powershell
Start-ADSyncSyncCycle -PolicyType Delta

```

---

## Résumé des points clés

| Étape | Description |
| --- | --- |
| Préparation | Vérification des comptes et domaines |
| Installation d’AD Connect | Choix du mode Express recommandé |
| Authentification Azure / AD | Connexion des deux environnements |
| Synchronisation automatique | Active immédiatement après installation |
| Suivi | Outils intégrés pour vérifier et dépanner |