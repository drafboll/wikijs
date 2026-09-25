---
title: Création automatique de répertoires personnels
description: 
published: 1
date: 2026-09-25T15:52:56.164Z
tags: 
editor: markdown
dateCreated: 2025-07-04T12:39:09.747Z
---

# Création automatique de répertoires personnels (lecteur U:) via Homes$ et Active Directory


## Configuration du dossier partagé Homes$

### Création du dossier

Sur le serveur `SVR-FIC-01`, créer le dossier :
 
```
D:\Homes$
```

### Partage du dossier

1. Clic droit > Propriétés > Partage > Partage avancé
2. Nom du partage : `Homes$` (le `$` le rend invisible dans le réseau)
3. Autorisations :
   - Groupe : **Utilisateurs authentifiés**
   - Permissions : **Modifier** + **Lecture** (ne pas cocher Contrôle total)
![capture_d'écran_2025-07-04_143959.png](/lecteur_u/capture_d'écran_2025-07-04_143959.png)
---

## Script PowerShell de création et de configuration

Voici le script complet qui :
- Crée le dossier personnel pour chaque utilisateur
- Attribue les bons droits NTFS (Modify à l'utilisateur, FullControl aux Admins)
- Met à jour les attributs HomeDrive (`U:`) et HomeDirectory (`\\serveur\homes$\utilisateur`) dans l’AD

```powershell
Import-Module ActiveDirectory

# Dossier racine des homes
$basePath = "D:\Homes$"
$serverName = "SVR-FIC-01"
$OU = "OU=Utilisateurs,DC=alpha,DC=lan"

# Récupérer tous les utilisateurs de l’OU
$users = Get-ADUser -Filter * -SearchBase $OU -Properties HomeDrive, HomeDirectory

foreach ($user in $users) {
    $login = $user.SamAccountName
    $userFolder = Join-Path $basePath $login
    $uncPath = "\\$serverName\Homes$\$login"

    # Créer le dossier s’il n’existe pas
    if (-not (Test-Path $userFolder)) {
        New-Item -ItemType Directory -Path $userFolder | Out-Null
        Write-Host " Dossier créé pour $login"
    }

    # Appliquer les droits NTFS
    $acl = Get-Acl $userFolder
    $acl.SetAccessRuleProtection($true, $false)  # Désactiver l’héritage
    $acl.Access | ForEach-Object { $acl.RemoveAccessRule($_) }

    # Droits de l’utilisateur : Modify
    $ruleUser = New-Object System.Security.AccessControl.FileSystemAccessRule(
        "$login", "Modify", "ContainerInherit,ObjectInherit", "None", "Allow"
    )
    $acl.AddAccessRule($ruleUser)

    # Droits des Domain Admins : FullControl
    $domainAdmins = New-Object System.Security.Principal.NTAccount("alpha\Domain Admins")
    $ruleAdmins = New-Object System.Security.AccessControl.FileSystemAccessRule(
        $domainAdmins, "FullControl", "ContainerInherit,ObjectInherit", "None", "Allow"
    )
    $acl.AddAccessRule($ruleAdmins)

    Set-Acl -Path $userFolder -AclObject $acl
    Write-Host " Droits NTFS appliqués à $login + Domain Admins"

    # Définir le lecteur réseau dans l’AD
    Set-ADUser $login -HomeDrive "U:" -HomeDirectory $uncPath
    Write-Host " Mapping U: défini pour $login ($uncPath)"
}
```

---

## Mise en place d’une tâche planifiée

Pour exécuter automatiquement ce script à chaque démarrage du serveur :

### Étapes :

1. Ouvrir le **Planificateur de tâches**
2. Créer une tâche nommée : **Script de Profil Utilisateur**
![capture_d'écran_2025-07-04_144100.png](/lecteur_u/capture_d'écran_2025-07-04_144100.png)
3. Onglet **Général** :
   - Exécuter avec les autorisations maximales
   - Compte utilisé : Administrateur
   - Configurer pour : Windows Server 2016/2019/2022

4. Onglet **Déclencheurs** :
   - Déclenchement : **Au démarrage**

5. Onglet **Actions** :
   - Action : Démarrer un programme
   - Programme/script : `powershell.exe`
   - Arguments :
     ```
     -ExecutionPolicy Bypass -File "C:\scripts\profils_utilisateurs.ps1"
     ```

Placez le script dans `C:\scripts\profils_utilisateurs.ps1` et assurez-vous que le chemin existe.

---

## Vérification sur un utilisateur AD

Dans les **propriétés du compte** AD d’un utilisateur (ex. `alexandre.faye`) :

- Onglet **Profil**
- Cochez : `Connecter`
- Lettre : `U:`
- Chemin : `\\SVR-FIC-01\Homes$\alexandre.faye`
![capture_d'écran_2025-07-04_144205.png](/lecteur_u/capture_d'écran_2025-07-04_144205.png)
---

## Résultat attendu

- Chaque utilisateur dispose de son propre répertoire personnel : `\\SVR-FIC-01\Homes$\prenom.nom`
- Le lecteur `U:` est monté automatiquement à la connexion
- L’utilisateur a accès **exclusivement à son propre dossier**
- Les administrateurs du domaine ont un contrôle total

![capture_d'écran_2025-07-04_160409.png](/lecteur_u/capture_d'écran_2025-07-04_160409.png)