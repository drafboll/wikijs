---
title: 04 - Installation d'Horizon pas à pas
description: Prérequis, transfert des installeurs, Connection Server, console, vCenter, golden image, Horizon Agent, pool Instant Clone, client et dépannage
published: 1
date: 2026-11-11T13:46:58.541Z
tags: 
editor: markdown
dateCreated: 2026-11-11T13:46:58.541Z
---

# 04 - Installation d'Horizon pas à pas

> **Objectif de la page :** aller d'un lab vSphere fonctionnel jusqu'à **un utilisateur connecté à son bureau Windows virtuel**. Chaque écran d'assistant est expliqué : ce que tu choisis, et surtout *pourquoi*.
{.is-info}

**Parcours du cours :** [01 Bases](/hyperviseur/VDI/01-bases-virtualisation) ➜ [02 Infrastructure](/hyperviseur/VDI/02-infrastructure-lab) ➜ [03 VDI & Horizon](/hyperviseur/VDI/03-vdi-et-horizon) ➜ **04 Installation** ➜ [05 Administration](/hyperviseur/VDI/05-horizon-administration) ➜ [06 Golden image](/hyperviseur/VDI/06-horizon-golden-image) ➜ [07 Production](/hyperviseur/VDI/07-horizon-production)

![Feuille de route](/vdi/04-feuille-de-route.svg)

> Les intitulés exacts des écrans peuvent légèrement varier d'une version d'Horizon à l'autre. Ce guide suit **Horizon 8 2412**. Si un écran ne correspond pas, la logique reste la même : lis l'écran et retrouve l'option équivalente.
{.is-warning}

---

## Étape 0 : prérequis

### 0.1 La checklist de la page 02

Ne commence pas sans avoir validé la [checklist de la page 02](/hyperviseur/VDI/02-infrastructure-lab) : DNS direct et inverse, heure, DHCP unique, promiscuité sur vSwitch0, comptes et OU créés.

### 0.2 La compatibilité

Avant chaque installation ou mise à jour, on vérifie trois choses sur les sites officiels :

| Quoi | Où | Pourquoi |
|---|---|---|
| Horizon 2412 ⇄ ta version de vCenter / ESXi | Matrice d'interopérabilité (Omnissa et Broadcom) | Une vCenter trop récente peut ne pas être encore supportée |
| Windows Server supportés pour le Connection Server | Notes de version Horizon 2412 | Typiquement Windows Server 2019 / 2022 |
| Windows supportés pour l'Agent | Notes de version, section « Supported Windows versions » | Chaque version de Windows 11 (23H2, 24H2...) est validée séparément |

Pour connaître tes versions : vSphere Client ➜ menu **?** ➜ *About vSphere* (vCenter) et sélection de l'hôte ➜ *Summary* (ESXi).

### 0.3 Les fichiers nécessaires

| Fichier | Où l'installer | Statut |
|---|---|---|
| `Omnissa-Horizon-Connection-Server-x86_64-2412-8.14.0-....exe` | cs01 | ✅ téléchargé |
| `Omnissa-Horizon-Agent-x86_64-2412-8.14.0-....exe` | Golden image | ✅ téléchargé |
| Horizon Client pour Windows | Ton PC (hôte) | ❌ à télécharger (étape 9) |
| ISO Windows Server 2022 (évaluation) | cs01 | Centre d'évaluation Microsoft |
| ISO Windows 11 Enterprise (évaluation) | Golden image | Centre d'évaluation Microsoft |
| Omnissa OS Optimization Tool | Golden image | Site Omnissa |
| Clé de licence d'évaluation Horizon | Console Horizon | Demande d'essai sur le site Omnissa |

> **Compte Omnissa bloqué sur le champ « Pays » vide :** essaie un autre navigateur ou une fenêtre privée, sans bloqueur de publicités ni traduction automatique (ils cassent souvent ces listes déroulantes). Si ça persiste, passe par le support du site.
{.is-info}

---

## Étape 1 : transférer les installeurs dans les VM

Les installeurs sont sur ton PC, mais doivent être exécutés **dans** cs01 et dans la golden image. Trois méthodes :

### Méthode A : partage réseau depuis ton PC (recommandée)

Sur ton PC Windows :
1. Crée un dossier, par exemple `D:\Lab-Partage`, et copie les installeurs dedans.
2. Clic droit ➜ *Propriétés ➜ Partage ➜ Partage avancé* ➜ ☑ Partager ce dossier, nom `Lab-Partage`, *Autorisations* : ton compte en lecture.
3. **Profil réseau de la carte VMnet8 :** s'il est « Public », Windows bloque le partage. En PowerShell administrateur :
   ```powershell
   Get-NetConnectionProfile
   Set-NetConnectionProfile -InterfaceAlias "VMware Network Adapter VMnet8" -NetworkCategory Private
   ```

Dans la VM (cs01 ou golden image) : `Win + R` ➜ `\\192.168.50.1\Lab-Partage`, puis connecte-toi avec **ton compte Windows du PC** (`NOMDUPC\toncompte`). Copie les fichiers en local (sur le bureau, par exemple) avant de les lancer.

### Méthode B : une ISO sur le datastore

1. Fabrique une ISO contenant les installeurs (avec un outil gratuit de création d'ISO, ou `oscdimg` du Windows ADK).
2. vSphere Client ➜ *Storage* ➜ `ds-esx01` ➜ *Files* ➜ *New Folder* `ISO` ➜ *Upload Files*.
3. Dans la VM : *Edit Settings ➜ CD/DVD drive ➜ Datastore ISO File* ➜ ☑ Connected.

C'est aussi comme ça que tu montes les ISO de Windows pour installer les VM.

### Méthode C : télécharger directement dans la VM

Le NAT donne accès à Internet : tu peux ouvrir le navigateur de la VM et te connecter au portail Omnissa. C'est simple, mais tu saisis tes identifiants dans une VM de test.

---

## Étape 2 : préparer cs01 (la VM « Horizon »)

### 2.1 La VM

| Paramètre | Valeur |
|---|---|
| OS | Windows Server 2022 Standard (**Expérience de bureau**) |
| vCPU / RAM | 4 / 8 Go (minimum supporté : 4 Go, 10 Go conseillés au-delà de 50 utilisateurs) |
| Disque | 80 Go thin |
| Carte réseau | VMXNET3 sur `VM Network` |

Installe Windows, puis les **VMware Tools** (vSphere Client ➜ VM ➜ *Guest OS ➜ Install VMware Tools*, puis `setup64.exe` dans le lecteur CD de la VM) et redémarre.

### 2.2 Nom, IP, domaine

Dans PowerShell administrateur sur la VM :

```powershell
# 1. Nom d'hôte
Rename-Computer -NewName "cs01" -Restart

# 2. IP fixe (vérifie le nom de la carte avec Get-NetAdapter)
New-NetIPAddress -InterfaceAlias "Ethernet0" -IPAddress 192.168.50.13 -PrefixLength 24 -DefaultGateway 192.168.50.2
Set-DnsClientServerAddress -InterfaceAlias "Ethernet0" -ServerAddresses 192.168.50.10

# 3. Vérifier que le DC répond avant de joindre le domaine
Resolve-DnsName srv-grp3.lan
Test-NetConnection 192.168.50.10 -Port 389

# 4. Jonction au domaine, directement dans la bonne OU
Add-Computer -DomainName "SRV-GRP3.lan" `
  -OUPath "OU=Serveurs,OU=LAB,DC=SRV-GRP3,DC=lan" -Credential (Get-Credential) -Restart
```

Après le redémarrage, **ouvre la session avec un compte du domaine** administrateur (par exemple `SRV-GRP3\Administrateur`). Le Connection Server s'installe avec un compte du domaine, pas avec un compte local.

### 2.3 Dernières vérifications

```powershell
nltest /dsgetdc:SRV-GRP3.lan                # trouve-t-on le contrôleur de domaine ?
w32tm /query /source                        # doit renvoyer srv-grp3.srv-grp3.lan
Resolve-DnsName vcsa01.srv-grp3.lan
Test-NetConnection vcsa01.srv-grp3.lan -Port 443
```

Passe aussi les mises à jour Windows **avant** d'installer Horizon.

> **Un Connection Server, c'est une machine dédiée.** Pas d'autre rôle dessus (pas de DC, pas de SQL, pas d'IIS, pas de vCenter). Et surtout pas sur ton PC Windows : il doit être dans le domaine, toujours allumé, et joignable par les bureaux.
{.is-danger}

---

## Étape 3 : installer le Connection Server

Lance l'installeur **en tant qu'administrateur** (clic droit ➜ *Exécuter en tant qu'administrateur*).

| # | Écran | Choix | Explication |
|---|---|---|---|
| 1 | Welcome | *Next* | |
| 2 | License Agreement | Accepter | |
| 3 | Destination Folder | Par défaut | Ne change pas le chemin sans raison : les docs et les outils supposent le chemin par défaut. |
| 4 | **Installation Options** | **Horizon Standard Server** · ☑ **Install HTML Access** · IPv4 | *Standard* = premier serveur, crée une nouvelle base. *Replica* = serveur supplémentaire. *Enrollment* = True SSO. HTML Access permet la connexion par navigateur. |
| 5 | **Data Recovery** | Un mot de passe fort + un indice | Il protège les **sauvegardes de la configuration** (base AD LDS). Sans lui, impossible de restaurer une sauvegarde. Note-le dans ton gestionnaire de mots de passe. |
| 6 | **Firewall Configuration** | ☑ *Configure Windows Firewall automatically* | Ouvre les ports 443, 8443, 4172, 4100-4101, etc. sur cs01. |
| 7 | **Initial Horizon Administrators** | *Authorize a specific domain user or domain group* : `SRV-GRP3\GRP-HZ-ADMINS` | Qui pourra se connecter à la console Horizon. Un groupe est plus propre qu'un compte : on ajoute ou retire des personnes sans toucher à Horizon. |
| 8 | User Experience Improvement Program | Au choix | Envoi de statistiques anonymes à Omnissa. |
| 9 | (selon la version) Type de déploiement / cloud | *General* | Ne concerne que des environnements gouvernementaux ou connectés au cloud Omnissa. |
| 10 | Ready to Install | *Install* | **10 à 20 minutes** en nested. |
| 11 | Finish | ☐ Show readme | |

### Ce qui vient d'être installé

```powershell
Get-Service | Where-Object { $_.DisplayName -match "Horizon|VDM" } |
    Select-Object Status, Name, DisplayName | Format-Table -AutoSize
```

Tu dois voir plusieurs services en *Running* :

| Service (nom affiché, approximatif) | Rôle |
|---|---|
| Horizon Connection Server | Le service principal (broker) |
| Horizon Web Component | Console d'admin et portail web (Tomcat) |
| Horizon Blast Secure Gateway | Passerelle Blast (port 8443) |
| Horizon PCoIP Secure Gateway | Passerelle PCoIP (port 4172) |
| Horizon Security Gateway Component | Tunnel HTTPS |
| Horizon Message Bus Component | Communication JMS avec les agents |
| Horizon Script Host | Scripts de démarrage de sessions |
| Horizon Framework Component | Socle commun |
| AD LDS Instance VMwareVDMDS | La base de configuration Horizon |

### Le certificat

Le Connection Server génère un **certificat auto-signé**, avec le nom convivial (*Friendly name*) **`vdm`**. C'est ce nom qui indique au service quel certificat utiliser.

- **En lab :** on garde l'auto-signé. Le client affichera un avertissement, et la console signalera le certificat comme non fiable : c'est normal.
- **En production :** on le remplace par un certificat émis par une autorité (interne ou publique) contenant le FQDN du serveur (et le nom du load balancer), on lui donne le Friendly name `vdm`, on renomme l'ancien, puis on redémarre le service. Procédure détaillée en page 07.

---

## Étape 4 : configurer Horizon dans la console

Depuis ton PC : **`https://cs01.srv-grp3.lan/admin`** ➜ accepte l'avertissement de certificat ➜ connecte-toi avec un compte membre de `GRP-HZ-ADMINS` (domaine : `SRV-GRP3`).

### 4.1 La licence

*Settings ➜ Product Licensing and Usage ➜ Edit License* ➜ colle ta clé d'évaluation.

> Sans licence valide, tu peux configurer Horizon, mais **les utilisateurs ne peuvent pas se connecter**. Les modèles de licence d'Omnissa (perpétuel, abonnement, Horizon Universal / Edge) ont évolué : vérifie sur le site d'Omnissa ce que propose l'évaluation actuellement.
{.is-warning}

### 4.2 Préparer le compte svc-horizon dans vCenter

Horizon va se connecter à vCenter avec `svc-horizon`. Il faut d'abord que vCenter le connaisse :

- **Si tu as branché vCenter sur l'AD** (page 02, §5.4) : utilise `svc-horizon@SRV-GRP3.lan`.
- **Sinon** : crée un utilisateur local SSO : vSphere Client ➜ *Administration ➜ Single Sign On ➜ Users and Groups ➜ Users ➜ domaine vsphere.local ➜ Add* : `svc-horizon`.

Puis donne-lui des droits : *Administration ➜ Access Control ➜ Global Permissions ➜ Add* ➜ l'utilisateur, ☑ *Propagate to children*.

| Contexte | Rôle à donner |
|---|---|
| **Lab** | `Administrator` (simple et sans surprise) |
| **Production** | Un rôle dédié « Horizon » avec uniquement les privilèges listés dans la doc Horizon (voir encadré) |

> **Privilèges d'un rôle Horizon en production (Instant Clones), dans les grandes lignes :** *Folder* (créer, supprimer) · *Datastore* (allouer de l'espace, parcourir, opérations de fichiers) · *Virtual Machine* (configuration, inventaire, interaction, provisioning, snapshots) · *Resource* (assigner une VM à un resource pool) · *Global* (agir en tant que vCenter, gérer les attributs personnalisés) · *Host ➜ Configuration* (paramètres avancés) · *Profile-driven storage* (afficher) · *Cryptographic operations* si tu utilises le vTPM. La liste exacte est dans la documentation « Horizon Installation » de ta version : copie-la telle quelle.
{.is-info}

### 4.3 Ajouter vCenter dans Horizon

*Settings ➜ Servers ➜ vCenter Servers ➜ Add*

| Écran | Champ | Valeur | Explication |
|---|---|---|---|
| vCenter Information | Server address | `vcsa01.srv-grp3.lan` | Toujours le FQDN (cohérent avec le certificat) |
| | User name | `svc-horizon@SRV-GRP3.lan` (ou `@vsphere.local`) | |
| | Port | 443 | |
| | Advanced settings | Par défaut | Limites d'opérations simultanées (provisioning, power on...). À augmenter en production sur de gros environnements. |
| Certificat | *View Certificate* ➜ **Accept** | | vCenter a un certificat signé par sa propre autorité (VMCA), qu'Horizon ne connaît pas encore |
| Storage | ☑ *Enable View Storage Accelerator*, cache par défaut | | Cache en RAM sur chaque ESXi des blocs les plus lus de la golden image. Accélère les démarrages et ouvertures de session. Consomme de la RAM de l'hôte (taille configurable). |
| | *Reclaim VM disk space* | Par défaut | Récupère l'espace des disques thin |
| Ready to Complete | *Submit* | | |

L'état de vCenter doit passer au **vert** dans *Monitor ➜ Dashboard* (*System Health*).

### 4.4 Déclarer le compte de domaine pour les Instant Clones

*Settings ➜ Instant Clone Domain Accounts ➜ Add*

| Champ | Valeur |
|---|---|
| Domain | `SRV-GRP3.lan` |
| User name | `svc-icdomain` |
| Password | (le mot de passe du compte) |

C'est ce compte qui créera les comptes ordinateur `vdi-01`, `vdi-02`... dans l'OU `VDI-Ordinateurs`, grâce à la délégation faite en page 02.

### 4.5 La base de données Events (facultatif en lab)

*Settings ➜ Event Configuration ➜ Event Database ➜ Edit* : SQL Server ou PostgreSQL. Sans elle, la console n'affiche qu'un historique très limité. En lab, tu peux t'en passer au début ; en production, c'est indispensable pour l'audit et le dépannage.

---

## Étape 5 : construire la golden image

![Pipeline de la golden image](/vdi/04-golden-image-pipeline.svg)

### 5.1 Quel Windows choisir ?

| Option | Avantages | Inconvénients |
|---|---|---|
| **Windows 11 Enterprise** (recommandé) | Ce que tu verras en entreprise, supporté | Exige **UEFI, Secure Boot, vTPM** et un disque d'**au moins 64 Go** |
| Windows 11 Enterprise LTSC | Moins d'applications intégrées, cycle long | Cas d'usage spécifiques |
| Windows Server 2022 en bureau mono-utilisateur | Pas de TPM, léger | Moins représentatif de la VDI réelle |

> **Correction par rapport au résumé initial :** le disque de la golden image doit faire **64 Go minimum** pour Windows 11 (60 Go fait échouer la vérification de l'installeur). Et pour un pool Instant Clone, **le disque des clones est celui de la golden image** : on ne choisit pas « 40 Go » dans le pool. En revanche, on peut ajuster vCPU et RAM des clones dans le pool (*compute profile*). Comme le disque est en thin et partagé (replica), un disque de 64 Go ne consomme presque rien par clone.
{.is-warning}

### 5.2 Activer le vTPM (Windows 11)

Le vTPM chiffre des données de la VM, donc vCenter a besoin d'un **fournisseur de clés** :

1. vSphere Client ➜ sélectionne `vcsa01` ➜ *Configure ➜ Security ➜ Key Providers ➜ Add ➜ Add Native Key Provider* ➜ nom `NKP-LAB` ☐ « Use key provider only with TPM protected ESXi hosts » (ton ESXi nested n'a pas de TPM physique : **décoche** cette case).
2. **Backup** du key provider (bouton *Back-up*) : obligatoire pour l'activer. Garde le fichier `.p12` et son mot de passe en lieu sûr.
3. Sur la VM Golden Image (éteinte) : *Edit Settings ➜ Add New Device ➜ Trusted Platform Module*. Vérifie *VM Options ➜ Boot Options* : **EFI** et ☑ *Secure Boot*.

> Les Instant Clones avec vTPM sont supportés dans les versions récentes d'Horizon, avec une option dédiée dans l'assistant du pool. Vérifie les notes de version de 2412. Si ça bloque dans ton lab, rabats-toi sur Windows Server 2022 pour avancer, puis reviens sur Windows 11.
{.is-info}

### 5.3 Paramètres de la VM

| Paramètre | Valeur |
|---|---|
| Guest OS | Microsoft Windows 11 (64-bit) |
| vCPU / RAM | 2 / 4 Go |
| Disque | **64 Go** thin, contrôleur NVMe ou PVSCSI |
| Réseau | VMXNET3, `VM Network` |
| Firmware | EFI + Secure Boot + vTPM |
| CD/DVD | ISO Windows 11 (déconnecté après l'installation) |
| Dossier vCenter | `Golden` |

### 5.4 Installer Windows

1. Démarre la VM, ouvre la console (*Launch Web Console*), appuie sur une touche pour démarrer sur l'ISO.
2. Choisis **Windows 11 Enterprise**, installation personnalisée sur le disque vierge.
3. À l'OOBE (premier démarrage) : *Options de connexion ➜ Joindre un domaine à la place* pour créer un **compte local** (ex. `labadmin`). Ne connecte pas de compte Microsoft.
4. Refuse toutes les options de télémétrie, localisation, etc.
5. **Windows Update** jusqu'à ce qu'il n'y ait plus rien à installer (plusieurs redémarrages).

### 5.5 VMware Tools, puis réseau

1. vSphere Client ➜ VM ➜ *Guest OS ➜ Install VMware Tools* ➜ dans la VM, lance `setup64.exe` ➜ **Complète** (ou Typique) ➜ redémarre.
2. Vérifie que l'état des Tools est **« Running »** dans vCenter et que la VM affiche son IP.
3. La carte réseau reste en **DHCP** (une IP .100-.199 doit apparaître) :
   ```powershell
   ipconfig /all          # DNS doit être 192.168.50.10, suffixe SRV-GRP3.lan
   Resolve-DnsName cs01.srv-grp3.lan
   ```

> Si la VM n'obtient pas d'IP DHCP : DHCP de SRV-GRP3 arrêté, DHCP VMware encore actif, ou promiscuité non configurée sur vSwitch0 (page 02 §4.5).
{.is-warning}

### 5.6 Jonction au domaine et applications

```powershell
Rename-Computer -NewName "GOLD-W11" -Restart
Add-Computer -DomainName "SRV-GRP3.lan" `
  -OUPath "OU=VDI-Ordinateurs,OU=LAB,DC=SRV-GRP3,DC=lan" -Credential (Get-Credential) -Restart
```

Pourquoi joindre la golden image au domaine ? Pour qu'elle reçoive les **GPO** pendant sa préparation et que le premier clone (cp-template) parte d'un état « domaine » propre. ClonePrep donnera ensuite à chaque clone son propre compte ordinateur.

Installe ensuite les **applications communes à tous les utilisateurs** : navigateur, lecteur PDF, 7-Zip, Office si tu en as une licence... Reste léger : chaque application ajoutée pèse sur tous les bureaux.

> **Règle d'or :** tout ce qui est installé dans la golden image se retrouve dans **tous** les bureaux. Les applications réservées à un service passent par App Volumes ou par une autre golden image (page 06).
{.is-info}

---

## Étape 6 : installer l'Horizon Agent

Copie l'installeur dans la VM (étape 1) et lance-le en administrateur.

| # | Écran | Choix | Explication |
|---|---|---|---|
| 1 | Welcome / License | Accepter | |
| 2 | Network protocol | IPv4 | |
| 3 | **Custom Setup** | Voir le tableau ci-dessous | Les fonctions disponibles dans les bureaux |
| 4 | **Remote Desktop Protocol Configuration** | ☑ *Enable the Remote Desktop capability on this computer* | L'agent s'appuie sur des composants RDP, et ça te donne un accès de secours |
| 5 | Ready to Install | *Install* | |
| 6 | Finish | **Redémarrer** | |

### Les fonctions de l'agent (Custom Setup)

| Fonction | Choix en lab | Ce que c'est |
|---|---|---|
| Core | ☑ (obligatoire) | Le cœur de l'agent |
| **Horizon Instant Clone Agent** | ☑ **OBLIGATOIRE** | Sans lui, impossible de créer un pool Instant Clone : l'image ne sera pas proposée ou la personnalisation échouera |
| USB Redirection | ☐ (désactivé par défaut) | Redirige les clés et périphériques USB du poste vers le bureau. Désactivé par défaut pour des raisons de sécurité : à activer seulement si nécessaire. |
| Client Drive Redirection | ☑ | Voir les disques du PC client dans le bureau |
| Real-Time Audio-Video | ☑ | Webcam et micro optimisés |
| Scanner / Serial Port Redirection | Au besoin | Périphériques spécifiques |
| Smartcard Redirection | Au besoin | Cartes à puce |
| Printer Redirection / Integrated Printing | ☑ | Imprimer sur l'imprimante du poste client |
| Geolocation Redirection | Au besoin | Position du client |
| Help Desk Plugin | ☑ | Infos détaillées sur la session dans la console (helpdesk) |
| Horizon Performance Tracker | ☑ (utile pour apprendre) | Affiche dans le bureau les statistiques du protocole |
| Optimisation multimédia / Teams | ☑ si proposé | Déporte l'audio et la vidéo des visios vers le poste client |
| Horizon Monitoring Service, etc. | Par défaut | Remontée d'indicateurs |

> **Ordre impératif : Tools avant l'agent.** Si tu mets à jour les VMware Tools plus tard, réinstalle ou répare l'agent ensuite, sinon tu risques des problèmes d'affichage ou d'USB.
{.is-danger}

> Sur une VM gérée par vCenter, l'agent ne demande **pas** de s'enregistrer auprès d'un Connection Server. C'est normal : c'est le pool qui fera le lien. La question n'apparaît que pour des machines « non gérées » (PC physiques, VM d'un autre hyperviseur), dans un pool manuel.
{.is-info}

---

## Étape 7 : optimiser, arrêter, prendre le snapshot

### 7.1 Optimiser avec OSOT

L'**Omnissa OS Optimization Tool** désactive tout ce qui ne sert à rien en VDI et coûte du CPU, de la RAM ou des IOPS : tâches planifiées, indexation, applications du Store, effets visuels, télémétrie, Windows Update automatique...

1. Lance OSOT en administrateur ➜ **Analyze** avec le modèle correspondant à ta version de Windows.
2. Passe en revue les éléments (tout est documenté). Par exemple, si tu veux garder Windows Search, décoche son optimisation.
3. **Optimize** ➜ redémarre.
4. **Generalize** : *ne le fais pas* au début. Il lance Sysprep, utile seulement dans certains cas.
5. **Finalize** juste avant l'arrêt final : nettoyage (journaux, fichiers temporaires, cache), libération de l'adresse DHCP, etc.

### 7.2 Les réglages à vérifier à la main

```powershell
powercfg /setactive SCHEME_MIN                 # profil « Performances élevées »
powercfg /change standby-timeout-ac 0          # jamais de mise en veille
powercfg /change monitor-timeout-ac 0
powercfg /hibernate off                        # pas d'hibernation
```

- Windows Update automatique : désactivé (OSOT le fait). Les mises à jour se feront à la main, sur la golden image, puis par Push Image.
- Antivirus : en VDI, il faut une configuration adaptée (exclusions, mode « VDI » de l'éditeur), voir page 06.

### 7.3 Arrêt propre et snapshot

1. Déconnecte l'ISO du lecteur CD (*Edit Settings ➜ CD/DVD ➜ Client Device*).
2. Dans la VM : `shutdown /s /t 0` (un vrai arrêt, pas une mise en veille, pas un arrêt brutal depuis vCenter).
3. vSphere Client ➜ VM éteinte ➜ *Snapshots ➜ Take Snapshot* :
   - **Name :** `gold-v1`
   - **Description :** `Win11 Ent 24H2 - KB du 11/2026 - Tools 12.x - Agent 2412 - OSOT`
   - (L'option « Include virtual machine's memory » est grisée : la VM est éteinte, c'est ce qu'on veut.)

> **Documente chaque snapshot.** Dans 6 mois, tu auras `gold-v1` à `gold-v8` et tu voudras savoir ce qui a changé entre chacun. La description du snapshot est faite pour ça.
{.is-success}

---

## Étape 8 : créer le pool Instant Clone

*Inventory ➜ Desktops ➜ Add*

| # | Écran | Champ | Valeur en lab | Explication |
|---|---|---|---|---|
| 1 | Type | | **Automated Desktop Pool** | Horizon crée les VM lui-même |
| 2 | vCenter Server | | **Instant Clone** + `vcsa01.srv-grp3.lan` | |
| 3 | User Assignment | | **Floating** | N'importe quel bureau libre (page 03) |
| 4 | Storage Optimization | | Par défaut (pas de vSAN) | Avec vSAN, Horizon applique des politiques de stockage adaptées |
| 5 | Desktop Pool ID | ID | `pool-w11-ic` | Identifiant unique, sans espace, **non modifiable** ensuite |
| | | Display name | `Bureau Windows 11` | Ce que voit l'utilisateur dans son client |
| | | Access group | `/` | Délégation d'administration (page 05) |
| 6 | **Provisioning Settings** | Enable provisioning | ☑ | |
| | | Stop provisioning on error | ☑ | Évite de créer 50 VM en erreur |
| | | Naming pattern | `vdi-{n:fixed=2}` | Donne `vdi-01`, `vdi-02`... (15 caractères max pour un nom NetBIOS) |
| | | Maximum machines | `2` | Taille du pool |
| | | Number of spare (powered on) machines | `1` | VM prêtes d'avance pour le prochain utilisateur |
| | | Provisioning timing | *All machines up-front* | Crée toutes les VM d'un coup (l'autre mode crée à la demande) |
| | | Add vTPM device | ☑ si la golden image a un vTPM | |
| 7 | **vCenter Settings** | Golden image | `Golden Image` | |
| | | Snapshot | `gold-v1` | |
| | | VM folder location | `DC-LAB/VDI` | |
| | | Cluster | `CL-LAB` | |
| | | Resource pool | `CL-LAB` | |
| | | Datastores | `ds-esx01`, overcommit *Unbounded* (lab) | En production, on répartit sur plusieurs datastores |
| | | Networks | *Use network from current golden image* | |
| | | Compute profile | Golden image (ou 2 vCPU / 4 Go) | |
| 8 | **Desktop Pool Settings** | State | Enabled | |
| | | Session types | Desktop | |
| | | Log off after disconnect | *After 30 minutes* | Une session déconnectée est fermée au bout de 30 min, et la VM recyclée |
| | | Allow users to restart machines | Yes | |
| 9 | **Remote Display Settings** | Default display protocol | **Blast** | |
| | | Allow users to choose protocol | Yes | Utile pour comparer avec PCoIP |
| | | 3D renderer | Manage using vSphere Client / désactivé | Pas de GPU dans ton lab |
| | | HTML Access | ☑ | |
| 10 | **Guest Customization** | Domain | `SRV-GRP3.lan (svc-icdomain)` | Le compte déclaré à l'étape 4.4 |
| | | AD container | `OU=VDI-Ordinateurs,OU=LAB` | Chemin **relatif** au domaine (sans `DC=...`) |
| | | Allow reuse of pre-existing computer accounts | ☑ | Si un compte `vdi-01` existe déjà (ancien essai), Horizon le réutilise au lieu d'échouer |
| | | ClonePrep scripts | Vides | Scripts lancés pendant la personnalisation |
| 11 | Ready to Complete | | ☑ **Entitle users after this wizard finishes** ➜ *Submit* | Ouvre directement l'écran des droits |

### Ce qui se passe ensuite (et pourquoi c'est long)

1. **Publishing the image** : Horizon crée `cp-template`, `cp-replica`, `cp-parent` (page 03 §7). Dans ton lab nested, compte **15 à 45 minutes**. Suis l'avancement dans *Inventory ➜ Desktops ➜ pool-w11-ic ➜ Summary* et dans les *Recent Tasks* de vCenter.
2. **Provisioning** : création de `vdi-01` et `vdi-02`.
3. **Customizing** : ClonePrep renomme, joint au domaine, renouvelle l'IP.
4. **Available** : les bureaux sont prêts. 🎉

Onglet **Machines** du pool : chaque VM doit passer à **Available**. Si tu vois *Error*, *Customization failed* ou *Agent unreachable*, va directement au tableau de dépannage à la fin de cette page.

---

## Étape 9 : donner les droits (entitlements)

Si tu as coché la case à l'étape 8, l'écran s'ouvre tout seul. Sinon : *Inventory ➜ Desktops ➜ pool-w11-ic ➜ Entitlements ➜ Add Entitlement* ➜ *Add* ➜ recherche `GRP-VDI-USERS` ➜ *OK*.

> **Donne toujours les droits à des groupes**, jamais à des utilisateurs un par un. Arrivée ou départ d'un collaborateur : on modifie uniquement le groupe dans l'AD, sans toucher à Horizon.
{.is-success}

---

## Étape 10 : se connecter avec le client Horizon

### 10.1 Installer le client sur ton PC

1. Télécharge **Omnissa Horizon Client for Windows** dans la section « Horizon Clients » de la page de téléchargement d'Horizon.
2. Installe avec les options par défaut (IPv4), redémarre si demandé.

### 10.2 Première connexion

1. Lance le client ➜ **+ Ajouter un serveur** ➜ `cs01.srv-grp3.lan`.
2. Avertissement de certificat (auto-signé) ➜ *Continuer*. En lab uniquement, tu peux changer le comportement dans *⋮ ➜ Configurer SSL ➜ Avertir avant de se connecter à des serveurs non fiables*.
3. Identifiant `vdiuser1`, mot de passe, domaine `SRV-GRP3`.
4. Double-clic sur **Bureau Windows 11**. 🎉

### 10.3 Tester aussi

| Test | Comment | Ce que tu dois observer |
|---|---|---|
| HTML Access | Navigateur ➜ `https://cs01.srv-grp3.lan` ➜ *Horizon HTML Access* | Le bureau dans un onglet |
| Changer de protocole | Clic droit sur le bureau dans le client ➜ PCoIP | Session en PCoIP |
| Statistiques | Dans le bureau, lance *Horizon Performance Tracker* | Protocole, codec, bande passante, latence |
| Déconnexion vs fermeture de session | Ferme la fenêtre, puis reconnecte-toi | Tu retrouves ta session (état *Disconnected* dans la console) |
| Non-persistance | Crée un fichier sur le bureau Windows, **déconnecte-toi** (Démarrer ➜ Se déconnecter), reconnecte-toi | Le fichier a disparu : la VM a été recréée |
| Console | *Monitor ➜ Sessions* | La session de vdiuser1, son bureau, son protocole, sa durée |

---

## Dépannage : les erreurs les plus fréquentes

| Symptôme | Causes probables | Où vérifier / que faire |
|---|---|---|
| Le client affiche « Impossible de se connecter au serveur » | DNS de ton PC, pare-feu de cs01, service Connection Server arrêté | `nslookup cs01.srv-grp3.lan`, `Test-NetConnection cs01.srv-grp3.lan -Port 443`, services sur cs01 |
| La console `/admin` refuse ton compte | Compte pas membre de `GRP-HZ-ADMINS`, domaine mal saisi | *Settings ➜ Administrators* (via un autre admin) |
| vCenter en rouge dans le dashboard | Certificat non accepté, mot de passe de `svc-horizon` changé, droits insuffisants | *Settings ➜ Servers ➜ vCenter ➜ Edit ➜ View certificate* |
| La golden image n'apparaît pas dans l'assistant du pool | Pas de snapshot, VM dans un datacenter non vu, ou agent sans l'option Instant Clone | Snapshots de la VM, réinstaller l'agent |
| Le publishing reste bloqué ou échoue | Tools pas *Running*, pas d'IP DHCP, réseau nested (promiscuité), datastore plein | Événements du pool, *Recent Tasks* de vCenter |
| « Customization operation timed out » / échec ClonePrep | DNS du clone, droits de `svc-icdomain`, OU inexistante, heure décalée | `dsacls` sur l'OU, chemin de l'OU dans le pool, DHCP |
| Machines en « Agent unreachable » | Le bureau ne joint pas cs01 (DNS, pare-feu 4002), agent mal installé | Depuis le bureau (console vSphere) : `Test-NetConnection cs01.srv-grp3.lan -Port 4002` |
| « Toutes les sources de bureau disponibles sont occupées » | Pool plein (max 2 !), machines en erreur | Onglet *Machines* du pool, augmenter *Maximum machines* |
| Écran noir puis déconnexion | Port 22443 bloqué, problème de pilote d'affichage (Tools/agent dans le mauvais ordre) | Essayer PCoIP ; réinstaller Tools puis agent |
| « Le domaine n'est pas disponible » à l'ouverture de session Windows | Relation d'approbation cassée, heure, DNS | Recréer le pool / Push Image, vérifier NTP |
| « Aucune licence » ou sessions refusées | Licence absente ou expirée | *Settings ➜ Product Licensing and Usage* |

### Où trouver les journaux

| Composant | Emplacement habituel |
|---|---|
| Connection Server | `C:\ProgramData\VMware\VDM\logs` sur les anciennes versions ; sur les versions renommées Omnissa, un dossier équivalent sous `C:\ProgramData\Omnissa\...`. Vérifie sur ton cs01. |
| Agent (dans le bureau) | Même logique, sous `C:\ProgramData\...\VDM\logs` (fichiers `debug-*.txt`) |
| ClonePrep / personnalisation | Journaux de l'agent + `C:\Windows\Temp` du clone |
| Client Horizon | `%TEMP%` de l'utilisateur, dossier du client (`vmware-<utilisateur>` ou équivalent Omnissa) |
| vCenter | *Monitor ➜ Tasks / Events* dans le vSphere Client |
| Console Horizon | *Monitor ➜ Events* (limité sans base Events) |

> La console Horizon propose aussi la collecte des journaux (*Troubleshooting* / *Log collection* selon la version). C'est ce que demandera le support Omnissa en cas de ticket.
{.is-info}

---

## Récapitulatif de l'état du lab après cette page

```
✅ SRV-GRP3     AD + DNS + NTP + DHCP
✅ esx01        ESXi nested, vSwitch0 en promiscuité
✅ vcsa01       vCenter, DC-LAB / CL-LAB, svc-horizon autorisé
✅ cs01         Connection Server 2412, licence, vCenter ajouté, svc-icdomain déclaré
✅ Golden Image Windows 11 + Tools + Agent (Instant Clone) + OSOT, snapshot gold-v1
✅ pool-w11-ic  2 Instant Clones flottants, droit pour GRP-VDI-USERS
✅ Client       vdiuser1 connecté en Blast
```

---

⬅ [03 - La VDI et Horizon](/hyperviseur/VDI/03-vdi-et-horizon) · **Page suivante :** [05 - Administrer Horizon au quotidien](/hyperviseur/VDI/05-horizon-administration) ➜