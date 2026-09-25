---
title: Configuration de Windows LAPS
description: 
published: 1
date: 2026-09-25T15:51:51.914Z
tags: 
editor: markdown
dateCreated: 2025-06-27T11:19:43.255Z
---

# Guide de mise en place de Windows LAPS avec Active Directory

## I. Présentation

Dans ce tutoriel, nous allons apprendre à configurer Windows LAPS avec l'Active Directory pour sécuriser le compte administrateur local des machines grâce à la rotation automatique du mot de passe. 

Windows LAPS, pour Windows Local Administrator Password Solution, est un composant intégré à Windows, développé par Microsoft, et qui va venir se greffer à un domaine Active Directory ou Azure Active Directory pour renforcer la sécurité des comptes "Administrateur" locaux des postes de travail et des serveurs.

Windows LAPS va générer un mot de passe robuste et unique pour le compte administrateur local de chaque machine qu'il gère, tout en effectuant une rotation automatique de ces mots de passe. Ensuite, les sésames seront chiffrés et stockés dans l'Active Directory ou l'Azure Active Directory, selon la configuration mise en place.

C'est une solution facile à mettre en place et qui permet de renforcer la sécurité de son infrastructure, sans pour autant que ce soit trop contraignant pour les administrateurs système. En soit, Windows LAPS n'est pas une solution nouvelle puisque LAPS existe depuis plusieurs années.

Toutefois, Windows LAPS, c'est le nom du nouveau produit de Microsoft qui prend la suite de LAPS (legacy) et qui apporte un certain nombre de nouveautés. À commencer par le fait que Windows LAPS est intégré à Windows, contrairement à LAPS qui est un agent à déployer.


## III. Mise à jour du schéma AD

```powershell
Import-Module LAPS
Update-LapsADSchema -Verbose
```
![1.png](/laps/1.png)

Ensuite verifier la presence des attributs windows laps :
- msLAPS-PasswordExpirationTime
- msLAPS-Password
- msLAPS-EncryptedPassword
- msLAPS-EncryptedPasswordHistory
- msLAPS-EncryptedDSRMPassword
- msLAPS-EncryptedDSRMPasswordHistory

![2.png](/laps/2.png)

![3.png](/laps/3.png)

## IV. Attribution des droits

Quand une machine va devoir effectuer une rotation du mot de passe du compte administrateur géré par Windows LAPS, elle devra sauvegarder ce mot de passe dans l'Active Directory. De ce fait, la machine doit pouvoir écrire/modifier son objet correspondant dans l'Active Directory.

La commande ci-dessous donne cette autorisation sur l'unité d'organisation "PC" (au sein de laquelle il y a mes postes de travail). Même si ce n'est pas obligatoire, je vous recommande de préciser le DistinguishedName de l'OU ciblée pour éviter les erreurs (notamment si vous avez plusieurs OUs avec le même nom).

```powershell
Set-LapsADComputerSelfPermission -Identity "OU=Serveurs,DC=alpha,DC=lan"
Set-LapsADReadPasswordPermission -Identity "OU=Serveurs,DC=alpha,DC=lan" -AllowedPrincipals "ALPHA\\GG-LAPS-READ"
Set-LapsADResetPasswordPermission -Identity "OU=Serveurs,DC=alpha,DC=lan" -AllowedPrincipals "ALPHA\\GG-LAPS-READ"
```
## IV. Déploiement des fichiers ADMX/ADML (GPO)

La suite consiste à configurer Windows LAPS à partir d'une stratégie de groupe. Cette GPO va permettre de définir la politique de mots de passe à appliquer sur le compte administrateur géré, l'emplacement de sauvegarde du mot de passe (Active Directory / Azure Active Directory), mais aussi le nom du compte administrateur à gérer avec Windows LAPS.

Tout d'abord, nous devons importer les modèles d'administration (ADMX) de Windows LAPS (c'est nécessaire s'il y a déjà un magasin central sur votre domaine, car Windows n'ira pas lire le magasin local). Ce processus n'est pas automatique. Sur le contrôleur de domaine, vous devez récupérer deux fichiers :

1. copier les fichier local :

   *  `C:\Windows\PolicyDefinitions\LAPS.admx`
   *  `C:\Windows\PolicyDefinitions\fr-FR\LAPS.adml`

1. Les déposer dans le Central Store :

   * `\\svr-dc-01\SYSVOL\alpha.lan\Policies\PolicyDefinitions\LAPS.admx`
   * `\\svr-dc-01\SYSVOL\alpha.lan\Policies\PolicyDefinitions\fr-FR\LAPS.adml`


## V. Configuration de la GPO Windows LAPS

Chemin : `Configuration ordinateur > Stratégies > Modèles d'administration > Système > LAPS`
![4.png](/laps/4.png)
Paramètres recommandés :

* **Configurer le répertoire de sauvegarde de mot de passe** : Activé → Active Directory
* **Paramètres du mot de passe** : Activé

  * Complexité : majuscules, minuscules, chiffres, spéciaux
  * Longueur : 16
  * Âge : 30 jours
* **Configurer la taille de l'historique des mots de passe chiffrés** : Activé → 1
* **Activer le chiffrement du mot de passe** : Activé
* **Nom du compte administrateur à gérer** : Laisser vide si c'est "Administrateur"

## VII. Application et vérification

La stratégie de groupe est prête, il ne reste plus qu'à faire une actualisation des GPO sur un poste, ici "SVR-ADCONNECT", afin de tester.

Sur les serveurs clients :

```powershell
gpupdate /force
Invoke-LapsPolicyProcessing
```

Vérification (sur DC ou admin autorisé) :

```powershell
Get-LapsADPassword -Identity "SVR-ADCONNECT" -AsPlainText
```

## VIII. Lecture des mots de passe (GUI + PowerShell)

* ADUC : Onglet "LAPS" dans les propriétés de l'objet ordinateur
* PowerShell :

```powershell
Get-LapsADPassword -Identity "SVR-ADCONNECT" -AsPlainText
```
![5.png](/laps/5.png)
## IX. Historique des mots de passe

```powershell
Get-LapsADPassword -Identity "SVR-ADCONNECT" -AsPlainText -IncludeHistory
```
![6.png](/laps/6.png)
## X. Forcer l'expiration / la rotation du mot de passe

* ADUC : Onglet LAPS → Bouton "Expirer maintenant"
* Ou en PowerShell local :

```powershell
Reset-LapsPassword
```

## XI. Logs

Observateur d'événements sur la machine :

```
Journaux des applications et des services > Microsoft > Windows > LAPS > Operational
```

## XII. Déchiffrement autorisé : définir les déchiffreurs

Dans la GPO LAPS : **Configurer les déchiffreurs de mot de passe autorisés** → spécifier le groupe (ex: `ALPHA\\GG-LAPS-READ`)

---

> Ne pas appliquer cette GPO sur l'OU `Domain Controllers`, car LAPS ne gère pas les DC (ils n'ont pas de compte admin local standard).

> Toujours vérifier que les serveurs sont à jour et que le module LAPS est bien présent (sinon ajouter avec : `Add-WindowsCapability`).
