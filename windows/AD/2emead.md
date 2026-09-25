---
title: Active Directory (ADDS) : ajouter un contrôleur de domaine à un domaine existant
description: 
published: 1
date: 2026-09-25T15:51:41.793Z
tags: 
editor: markdown
dateCreated: 2025-06-18T08:17:26.930Z
---

# Active Directory (ADDS) : Ajouter un contrôleur de domaine à un domaine existant

 
## I. Présentation

Ajouter un second (ou troisième...) contrôleur de domaine (DC) dans un environnement Active Directory permet :

* La redondance
* La répartition des requêtes
* La résilience de l'annuaire

> La procédure fonctionne pour un DC physique ou virtuel, avec ou sans GUI.

## II. Prérequis

### Niveau fonctionnel

Vérifiez les niveaux fonctionnels avec :

```powershell
Get-ADDomain | fl Name,DomainMode
Get-ADForest | fl Name,ForestMode
```

### Vérifier la santé de l’AD

Avant l’ajout, exécuter ces commandes :

```powershell
dcdiag /s:NomDuDC /v /f:c:\dcdiag.txt
dcdiag /v /s:NomDuDC /test:dns /DnsBasic /f:c:\dcdiag-dns.txt
repadmin /showrepl *
repadmin /replsum
```

Vérifiez aussi les journaux :

* Directory Service
* DNS Server
* Réplication DFS

## III. Préparer le nouveau contrôleur de domaine

### Réseau

Configurer une IP fixe avec comme DNS primaire l'IP du DC existant.
![ip.png](/2emead/ip.png)
### Nom de la machine

```powershell
Rename-Computer -NewName SVR-DC-02
Restart-Computer
```

### Autres préparations

* Installer les mises à jour
* Régler le fuseau horaire et la date/heure

> 📝 Le serveur peut rester en Workgroup jusqu’à sa promotion.

## IV. Installer le rôle ADDS

Via **Gestionnaire de serveur** :

1. Gérer > Ajouter des rôles et fonctionnalités
2. Sélectionner « Services AD DS »
3. Ajouter les fonctionnalités nécessaires
4. Lancer l'installation
![adds.png](/2emead/adds.png)

## V. Promouvoir le serveur en tant que contrôleur de domaine ADDS

1. Une fois l’installation du rôle terminée, cliquez sur « **Promouvoir ce serveur en contrôleur de domaine** »
2. Choisir **Ajouter un contrôleur de domaine à un domaine existant**
3. Authentifiez-vous avec un compte Administrateur du domaine
![ajout.png](/2emead/ajout.png)

### Choix des options :

* ✅ Serveur DNS
* ✅ Catalogue global (GC)
* ❌ Contrôleur en lecture seule (RODC)
* Mot de passe de restauration (DSRM)
![dns.png](/2emead/dns.png)

Ignorez les avertissements DNS et conservez les chemins par défaut.

Une fois validé, le serveur redémarre automatiquement.

## VI. Vérifier l’opération

### Console AD

Vérifier que l’OU `Domain Controllers` contient le nouveau DC.

### PowerShell

```powershell
Get-ADDomainController -Identity SRV-ADDS-02
```

### Réplication AD

```powershell
repadmin /showrepl *
repadmin /replsummary
```

### Test de connexion

Essayer une connexion de session sur le nouveau DC.

### Configuration DNS croisée

Pour chaque DC, configurer :

* DNS préféré : IP de l'autre DC
* DNS auxiliaire : 127.0.0.1

## VII. Conclusion

Le second contrôleur de domaine est désormais opérationnel, ce qui renforce la résilience et la performance de votre environnement ADDS.
