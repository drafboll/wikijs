---
title: 06 - Golden image, cycle de vie, profils et applications
description: Bien construire et maintenir une golden image, Push Image, stratégie de test, profils utilisateurs (FSLogix, DEM), App Volumes, antivirus et activation en VDI
published: 1
date: 2026-11-11T13:48:19.619Z
tags: 
editor: markdown
dateCreated: 2026-11-11T13:48:19.619Z
---

# 06 - Golden image, cycle de vie, profils et applications

> **Objectif de la page :** maîtriser ce qui fait **80 % du travail récurrent** d'un administrateur VDI : faire vivre la golden image (mises à jour, tests, déploiement, retour arrière) et construire un bureau non persistant où l'utilisateur retrouve pourtant **tout** : profil, réglages, applications et fichiers.
{.is-info}

**Parcours du cours :** [01 Bases](/hyperviseur/VDI/01-bases-virtualisation) ➜ [02 Infrastructure](/hyperviseur/VDI/02-infrastructure-lab) ➜ [03 VDI & Horizon](/hyperviseur/VDI/03-vdi-et-horizon) ➜ [04 Installation](/hyperviseur/VDI/04-installation-horizon) ➜ [05 Administration](/hyperviseur/VDI/05-horizon-administration) ➜ **06 Golden image** ➜ [07 Production](/hyperviseur/VDI/07-horizon-production)

---

## 1. La golden image est un produit

En VDI non persistante, la golden image n'est pas « une VM parmi d'autres » : c'est **le poste de travail de toute l'entreprise**. Une erreur dedans se retrouve chez tous les utilisateurs au prochain Push Image. On la traite donc comme un logiciel :

| Principe | En pratique |
|---|---|
| **Versionnée** | `gold-v1`, `gold-v2`... avec une description de chaque changement |
| **Documentée** | Un journal des modifications (changelog) dans le wiki |
| **Testée** | Jamais de mise en production sans passage sur un pool de test |
| **Réversible** | On garde le snapshot précédent pour revenir en arrière en quelques minutes |
| **Minimaliste** | Seulement ce qui est commun à tous les utilisateurs |
| **Reproductible** | Idéalement, on sait la reconstruire de zéro (procédure écrite ou automatisée) |

### Exemple de journal des modifications

| Version | Date | Auteur | Changements | Pools |
|---|---|---|---|---|
| gold-v1 | 11/11/2026 | moi | Win11 Ent 24H2, Tools, Agent 2412, OSOT | pool-w11-ic |
| gold-v2 | 09/12/2026 | moi | Cumulatif de décembre, Edge à jour, FSLogix installé | pool-test ➜ pool-w11-ic |
| gold-v3 | ... | ... | ... | ... |

---

## 2. Bien construire la golden image

### 2.1 L'ordre d'installation complet

| Ordre | Composant | Remarque |
|---|---|---|
| 1 | Windows + toutes les mises à jour | Installation propre depuis l'ISO, jamais une VM « recyclée » |
| 2 | **VMware Tools** | Pilotes réseau, stockage, affichage |
| 3 | Applications de base | Navigateur, Office, lecteur PDF, runtimes (.NET, VC++) |
| 4 | Agents tiers | Antivirus / EDR (en mode VDI), **FSLogix**, outils de supervision |
| 5 | **Horizon Agent** | Option Instant Clone |
| 6 | **DEM Agent** (FlexEngine) | Si DEM est utilisé |
| 7 | **App Volumes Agent** | **Toujours en dernier** parmi les agents Omnissa |
| 8 | Optimisation OSOT | Analyze ➜ Optimize |
| 9 | Finalisation | OSOT Finalize, arrêt propre, snapshot |

> **Si tu mets à jour les Tools**, mets ensuite à jour (ou répare) l'Horizon Agent, puis les agents qui suivent. L'ordre des pilotes compte.
{.is-warning}

### 2.2 Ce qu'on met dans l'image, et ce qu'on n'y met pas

| ✅ Dans l'image | ❌ Pas dans l'image |
|---|---|
| Applications utilisées par (presque) tout le monde | Applications d'un seul service ➜ App Volumes ou applis publiées |
| Polices, certificats racine de l'entreprise | Données utilisateur, fichiers sur le bureau |
| Agents (Horizon, DEM, App Volumes, FSLogix, antivirus) | Comptes utilisateurs, profils locaux |
| Paramètres machine (GPO de préférence) | Paramètres utilisateur ➜ DEM / GPO |
| Pilotes d'impression universels | Logiciels « au cas où » |

### 2.3 Les pièges spécifiques à certaines applications

| Application | Piège | Solution |
|---|---|---|
| **Microsoft 365 Apps (Office)** | Activation par utilisateur liée à la machine : en non persistant, la machine change à chaque session | Installer avec l'**activation d'ordinateurs partagés** (*Shared Computer Activation*, paramètre de l'outil de déploiement Office) |
| **OneDrive** | Installé par utilisateur par défaut, cache local perdu | Installation **par machine** (`/allusers`), Fichiers à la demande, et profil dans FSLogix |
| **Teams** | Installation par utilisateur, cache lourd | Installation par machine pour la VDI, optimisation Horizon (page 05), cache dans le profil FSLogix |
| **Navigateurs** | Mises à jour automatiques qui cassent la cohérence | Désactiver la mise à jour automatique, mettre à jour dans l'image |
| **Applications du Store (Appx)** | Réinstallées pour chaque utilisateur, ralentissent la première ouverture de session | Supprimer les applications inutiles provisionnées (OSOT le propose) |
| **Antivirus / EDR** | Chaque clone se déclare comme une nouvelle machine, scans simultanés sur tous les bureaux (tempête d'IO) | Mode VDI de l'éditeur, script d'inscription « non persistant », exclusions, signatures partagées |

### 2.4 L'antivirus en VDI

Un antivirus classique dans 300 bureaux qui démarrent à 8 h 30 et lancent tous une analyse et une mise à jour de signatures : le stockage s'effondre. Les bonnes pratiques :

- utiliser le **mode VDI / non persistant** de l'éditeur (Microsoft Defender for Endpoint, CrowdStrike, SentinelOne... ont tous une procédure spécifique) ;
- faire l'**analyse complète sur la golden image**, avant le snapshot, puis la marquer comme analysée ;
- **signatures partagées** sur un partage réseau plutôt que téléchargées par chaque bureau ;
- **exclusions** recommandées : conteneurs FSLogix (`*.vhd`, `*.vhdx`), processus des agents Horizon / App Volumes / DEM (listes fournies par Omnissa et l'éditeur) ;
- **pas d'analyse planifiée** dans les clones (ils vivent quelques heures).

### 2.5 L'activation de Windows

Les clones doivent être activés automatiquement : on utilise **KMS** ou l'**activation via Active Directory** (ADBA). Les clés **MAK** (activation unique, comptée) sont à éviter : chaque clone consommerait une activation.

---

## 3. Le cycle de mise à jour : Push Image

![Cycle de vie de l'image](/vdi/06-cycle-vie-image.svg)

### 3.1 La procédure mensuelle

1. **Allumer** la golden image (le pool continue de tourner sur `gold-v1` : l'image et les clones sont indépendants).
2. **Mettre à jour** : Windows Update, applications, éventuellement Tools puis agents.
3. **Réappliquer l'optimisation** : certaines mises à jour réactivent des services. Relance OSOT (*Analyze* puis *Optimize*).
4. **Finaliser** : OSOT *Finalize*, puis `shutdown /s /t 0`.
5. **Snapshot** `gold-v2` avec sa description.
6. **Tester** sur un pool de test (§4).
7. **Planifier le Push Image** sur le pool de production.
8. **Mettre à jour le journal** des modifications.
9. Une fois la nouvelle version validée en production (une à deux semaines), **supprimer les snapshots trop anciens** (on garde la version courante et la précédente).

### 3.2 Le Push Image dans la console

*Inventory ➜ Desktops ➜ pool ➜ Maintain ➜ Schedule* (le menu peut s'appeler « Push Image » selon la version)

| Champ | Choix | Explication |
|---|---|---|
| Golden image / Snapshot | `Golden Image` / `gold-v2` | Tu peux aussi changer de golden image (nouvelle VM) |
| Start time | La nuit, ou « maintenant » en lab | |
| **Force users to log off** | | Les utilisateurs connectés reçoivent un message, puis sont déconnectés après un délai. Tous les bureaux passent sur la nouvelle image rapidement. |
| **Wait for users to log off** | ✅ en général | Les machines occupées ne sont remplacées qu'à la déconnexion de leur utilisateur. Aucune coupure, mais la transition peut prendre du temps. |
| Stop at first error | ☑ | Si la publication échoue, rien n'est remplacé |
| Logoff message / délai | « Mise à jour de votre bureau dans 15 minutes » | Si tu forces la déconnexion |

Pendant l'opération, le pool affiche l'**image en cours** et l'**image en attente** (pending). Tu peux **annuler** tant que le remplacement n'a pas commencé.

> **Image secondaire (déploiement « canari ») :** les versions récentes d'Horizon permettent de pousser une **image secondaire** sur quelques machines d'un pool de production seulement, de la faire tester par des utilisateurs volontaires, puis de la **promouvoir** en image principale. C'est la meilleure méthode quand on n'a pas de pool de test représentatif. Cherche « secondary image » dans la documentation de ta version.
{.is-success}

### 3.3 Le retour arrière

Un problème découvert après le Push Image ? On refait simplement un Push Image **vers `gold-v1`**. C'est pour ça qu'on garde toujours le snapshot précédent.

### 3.4 Garder une chaîne de snapshots courte

Chaque snapshot ajoute un disque delta, et une longue chaîne ralentit la golden image et peut finir par poser des problèmes. Deux habitudes :
- garder **2 à 3 snapshots** maximum (courant, précédent, éventuellement un « avant changement majeur ») ;
- une à deux fois par an, **cloner** la golden image vers une nouvelle VM (le clone a un disque « à plat », sans chaîne), ou la reconstruire de zéro, surtout lors d'un changement de version majeure de Windows.

### 3.5 Automatiser la construction (pour aller plus loin)

Les équipes VDI matures ne construisent plus leur image à la main :

| Outil | Rôle |
|---|---|
| **HashiCorp Packer** (provider vSphere) | Construit la VM de zéro à partir de l'ISO, de façon reproductible |
| MDT / MECM (SCCM) task sequences | Installation et configuration automatisées |
| Scripts PowerShell, Chocolatey, winget | Installation des applications |
| OSOT en ligne de commande | Optimisation scriptée |

---

## 4. Tester avant de déployer

### 4.1 Le pool de test

Crée un deuxième pool, `pool-w11-test`, de 1 ou 2 machines, sur la même golden image, avec des droits pour un groupe `GRP-VDI-TEST` (toi et quelques utilisateurs pilotes). Chaque nouvelle version passe **d'abord** sur ce pool.

### 4.2 La recette (checklist de validation)

| ✓ | Test |
|---|---|
| ☐ | Le pool se publie et les machines passent à *Available* |
| ☐ | Connexion en Blast, en interne et (si disponible) via l'UAG |
| ☐ | Durée d'ouverture de session normale (pas de régression) |
| ☐ | Profil retrouvé (fond d'écran, favoris, signature Outlook...) |
| ☐ | Applications principales : Office, navigateur, applications métier |
| ☐ | Impression (redirection et imprimantes réseau) |
| ☐ | Teams : appel audio / vidéo optimisé |
| ☐ | USB, presse-papiers, lecteurs clients selon les politiques |
| ☐ | Antivirus actif et inscrit correctement |
| ☐ | Déconnexion / reconnexion, fermeture de session |
| ☐ | Aucun nouvel événement d'erreur dans la console |

---

## 5. Les profils utilisateurs : retrouver ses affaires sur un bureau jetable

![Couches du bureau](/vdi/06-couches-bureau.svg)

Dans un pool flottant, la VM est détruite à chaque déconnexion. Pour que l'utilisateur retrouve son environnement, on stocke à l'extérieur :

| Quoi | Exemple | Solution |
|---|---|---|
| **Fichiers** | Documents, Bureau, Images | Redirection de dossiers (GPO) vers un serveur de fichiers, ou OneDrive |
| **Profil Windows** | Registre utilisateur (`NTUSER.DAT`), `AppData`, caches Outlook / Teams / OneDrive | **FSLogix Profile Container** |
| **Paramètres ciblés** | Réglages d'une application, imprimante par défaut, lecteurs réseau | **DEM** |
| **Applications** | Logiciels d'un service | **App Volumes** |

### 5.1 Les options de gestion de profil

| Solution | Principe | Avantages | Limites |
|---|---|---|---|
| Profil itinérant Windows (roaming) | Le profil est copié depuis un partage à l'ouverture, recopié à la fermeture | Natif | **Lent** (copie de milliers de fichiers), corruptions : à éviter en VDI |
| Profil obligatoire (mandatory) | Profil en lecture seule, identique pour tous | Très rapide | Aucune personnalisation conservée (souvent combiné avec DEM) |
| **FSLogix Profile Container** | Le profil entier est dans un disque VHDX sur un partage, **monté** à l'ouverture de session (pas de copie) | Rapide, transparent pour les applications, gère bien Outlook / Teams / OneDrive | Un VHDX par utilisateur à stocker et sauvegarder |
| **DEM** (personnalisation) | Sauvegarde et restaure des réglages précis (clés de registre, fichiers de config) | Granulaire, multi-OS, migrations faciles | Il faut définir ce qu'on sauvegarde |
| App Volumes Writable Volume | Disque inscriptible par utilisateur, monté à la connexion | Conserve les applications installées par l'utilisateur | Plus rare aujourd'hui |

> **Le couple le plus fréquent en entreprise :** **FSLogix** pour le profil + **DEM** pour l'environnement (lecteurs, imprimantes, Smart Policies). FSLogix est fourni par Microsoft avec la plupart des licences Microsoft 365 / Windows Enterprise : vérifie les droits de ton entreprise.
{.is-info}

### 5.2 TP : mettre en place FSLogix dans ton lab

**1. Le partage sur SRV-GRP3** (en lab, sur le DC ; en production, sur un serveur de fichiers dédié ou un NAS) :

```powershell
New-Item -ItemType Directory -Path "C:\Profils"
New-SmbShare -Name "profils$" -Path "C:\Profils" -FullAccess "Tout le monde"   # les droits fins sont sur NTFS

# Droits NTFS recommandés pour FSLogix
$acl = Get-Acl "C:\Profils"
$acl.SetAccessRuleProtection($true, $false)   # coupe l'héritage
$rules = @(
  New-Object Security.AccessControl.FileSystemAccessRule("Administrateurs","FullControl","ContainerInherit,ObjectInherit","None","Allow"),
  New-Object Security.AccessControl.FileSystemAccessRule("SYSTEM","FullControl","ContainerInherit,ObjectInherit","None","Allow"),
  New-Object Security.AccessControl.FileSystemAccessRule("CREATOR OWNER","Modify","ContainerInherit,ObjectInherit","InheritOnly","Allow"),
  New-Object Security.AccessControl.FileSystemAccessRule("SRV-GRP3\GRP-VDI-USERS","Modify","None","None","Allow")
)
$rules | ForEach-Object { $acl.AddAccessRule($_) }
Set-Acl "C:\Profils" $acl
```

> Les noms de groupes intégrés (« Administrateurs », « Tout le monde ») dépendent de la langue de Windows. Sur un Windows anglais : « Administrators », « Everyone ». Le principe : les utilisateurs peuvent **créer** leur dossier à la racine (ce dossier seulement), et n'ont des droits que sur **leurs propres** dossiers (CREATOR OWNER).
{.is-warning}

**2. Dans la golden image :** installe l'agent FSLogix (`FSLogixAppsSetup.exe`, téléchargé sur le site de Microsoft), puis configure-le. Par GPO (ADMX FSLogix) ou directement dans le registre :

```powershell
$k = "HKLM:\SOFTWARE\FSLogix\Profiles"
New-Item $k -Force | Out-Null
New-ItemProperty $k -Name Enabled -Value 1 -PropertyType DWord -Force
New-ItemProperty $k -Name VHDLocations -Value "\\srv-grp3.srv-grp3.lan\profils$" -PropertyType MultiString -Force
New-ItemProperty $k -Name VolumeType -Value "VHDX" -PropertyType String -Force
New-ItemProperty $k -Name SizeInMBs -Value 30000 -PropertyType DWord -Force             # 30 Go max (dynamique)
New-ItemProperty $k -Name IsDynamic -Value 1 -PropertyType DWord -Force                # grossit à l'usage
New-ItemProperty $k -Name DeleteLocalProfileWhenVHDShouldApply -Value 1 -PropertyType DWord -Force
New-ItemProperty $k -Name FlipFlopProfileDirectoryName -Value 1 -PropertyType DWord -Force   # dossiers nommés "utilisateur_SID"
```

| Valeur | Rôle |
|---|---|
| `Enabled` | Active les conteneurs de profil |
| `VHDLocations` | Où stocker les VHDX (plusieurs chemins possibles) |
| `VolumeType` | VHDX (recommandé) ou VHD |
| `SizeInMBs` / `IsDynamic` | Taille maximale du disque / allocation dynamique (comme du thin) |
| `DeleteLocalProfileWhenVHDShouldApply` | Supprime un éventuel profil local en conflit |
| `FlipFlopProfileDirectoryName` | Nom de dossier lisible (`vdiuser1_S-1-5-...` au lieu de `S-1-5-..._vdiuser1`) |

**3.** Optimise, arrête, snapshot `gold-v2`, **Push Image**.

**4. Test :** connecte `vdiuser1`, change le fond d'écran, crée un favori dans le navigateur, déconnecte-toi. Dans `C:\Profils` sur SRV-GRP3, un dossier `vdiuser1_S-1-5-...` contient `Profile_vdiuser1.vhdx`. Reconnecte-toi (sur un autre clone) : le fond d'écran et le favori sont là. 🎉

> **Exclure les administrateurs :** les comptes du groupe local « FSLogix Profile Exclude List » n'utilisent pas de conteneur. Mets-y les comptes d'administration pour qu'ils gardent un profil local.
{.is-info}

### 5.3 Dynamic Environment Manager (DEM)

DEM se compose de :

| Élément | Rôle |
|---|---|
| **Console de gestion DEM** | Où l'on crée les configurations (sur un poste d'administration) |
| **Partage de configuration** | `\\serveur\dem-config$` : les fichiers de config créés par la console, lus par les agents |
| **Partage d'archives de profils** | `\\serveur\dem-profiles$\%username%` : les réglages sauvegardés de chaque utilisateur |
| **Agent FlexEngine** | Dans la golden image, applique la configuration à l'ouverture et la sauvegarde à la fermeture de session |
| GPO DEM | Indique aux agents où sont les partages |

Ce que DEM sait faire :

| Fonction | Exemple |
|---|---|
| **Personnalisation** (*Personalization*) | Sauvegarder et restaurer les réglages de Notepad++, des signatures Outlook, des paramètres Windows (souris, clavier, région). Des modèles prêts à l'emploi existent pour les applications courantes. |
| **Environnement utilisateur** (*User Environment*) | Lecteurs réseau, imprimantes, raccourcis, associations de fichiers, variables d'environnement, clés de registre, paramètres ADMX, élévation de privilèges ciblée |
| **Conditions** | Appliquer selon le groupe AD, le nom du poste client, l'adresse IP du client, l'heure, le type de connexion... |
| **Horizon Smart Policies** | Selon le contexte : activer ou désactiver USB, presse-papiers, lecteurs clients, impression, limiter la bande passante. Par exemple : « depuis l'extérieur via UAG, pas de presse-papiers sortant ». |
| **Migration** | Faciliter le passage de Windows 10 à 11, ou d'Office 2019 à M365, en ne gardant que les bons réglages |

> DEM existe en édition **Standard** (personnalisation de base) et **Enterprise** (toutes les fonctions, dont les Smart Policies), selon l'édition d'Horizon achetée.
{.is-info}

---

## 6. App Volumes : livrer les applications à la demande

### 6.1 Le principe

Au lieu d'installer une application dans la golden image (pour tout le monde) ou dans chaque VM (impossible en non persistant), App Volumes :

1. **capture** l'application une fois sur une machine de packaging, dans un disque virtuel (le **package**) ;
2. **attache** ce disque, en lecture seule, au bureau de l'utilisateur à l'ouverture de session, selon ses groupes AD ;
3. **fusionne** le contenu dans Windows : l'application apparaît comme installée (menu Démarrer, registre, fichiers).

### 6.2 Les composants

| Composant | Rôle |
|---|---|
| **App Volumes Manager** | Serveur Windows + base SQL Server. Console web, gère les packages et les affectations, parle à vCenter pour attacher les disques. Au moins 2 en production derrière un répartiteur. |
| **App Volumes Agent** | Dans la golden image (installé en dernier) et sur la machine de packaging |
| **Machine de packaging** | VM propre (même OS que la golden image) servant à capturer les applications |
| **Packages** | Disques VMDK (ou VHD) sur les datastores, en lecture seule |
| **Applications** | Regroupent les versions successives d'un même logiciel. Le marqueur **CURRENT** désigne la version livrée. |
| **Assignments** | Affectation d'une application à un groupe AD, un utilisateur, un ordinateur ou une OU |
| **Writable Volumes** | Disque inscriptible par utilisateur (optionnel) |

### 6.3 Les avantages

- Une golden image **plus petite et plus stable** (moins de mises à jour).
- Une application **mise à jour une seule fois**, pour tous ses utilisateurs, en changeant le marqueur CURRENT, avec retour arrière immédiat.
- Une seule golden image pour des services aux besoins très différents.

### 6.4 Les limites

- Certaines applications s'y prêtent mal : pilotes noyau, antivirus, applications intégrées à Windows, services qui doivent démarrer avant l'ouverture de session.
- Une infrastructure de plus à maintenir (serveurs, base SQL, packaging).

### 6.5 Les alternatives

| Alternative | Remarque |
|---|---|
| Applications publiées (RDS) | L'application tourne ailleurs, seule la fenêtre est affichée |
| **MSIX app attach** (Microsoft) | Principe similaire, natif Microsoft, surtout dans l'écosystème Azure |
| Liquidware FlexApp, Numecent Cloudpaging | Solutions tierces |
| Plusieurs golden images | Simple, mais multiplie la maintenance |

### 6.6 Grille de décision : où mettre une application ?

| Question | Si oui ➜ |
|---|---|
| (Presque) tout le monde l'utilise ? | **Golden image** |
| Un service ou un groupe de personnes ? | **App Volumes** |
| Application ancienne, lourde, ou à isoler ? | **Application publiée (RDS)** |
| Installée par l'utilisateur lui-même ? | Pool **persistant**, ou **Writable Volume** |
| Application web ? | Rien à installer : le navigateur |

---

## 7. Plusieurs golden images : une stratégie

Une entreprise moyenne a généralement :

| Golden image | Pools | Usage |
|---|---|---|
| `GI-W11-STANDARD` | pool-standard, pool-test | Tous les utilisateurs bureautiques (+ App Volumes) |
| `GI-W11-DEV` | pool-dev (dédié, persistant ou non) | Développeurs, outils lourds |
| `GI-W11-GPU` | pool-cad | CAO / 3D avec vGPU |
| `GI-WS2022-RDS` | farm-rds | Applications publiées |

L'objectif est d'en avoir **le moins possible** : chaque golden image doit être mise à jour, testée et documentée chaque mois.

---

## 8. Quiz

<details>
<summary><b>1. Dans quel ordre installe-t-on Tools, Horizon Agent, DEM Agent et App Volumes Agent ?</b></summary>

VMware Tools ➜ Horizon Agent ➜ DEM Agent ➜ App Volumes Agent (toujours en dernier).
</details>

<details>
<summary><b>2. Différence entre « Force users to log off » et « Wait for users to log off » lors d'un Push Image ?</b></summary>

Force : les utilisateurs sont prévenus puis déconnectés après un délai, la bascule est rapide. Wait : les machines occupées ne sont remplacées qu'à la déconnexion naturelle de leur utilisateur, sans coupure, mais plus lentement.
</details>

<details>
<summary><b>3. Le Push Image de gold-v2 provoque des plantages d'une application. Que fais-tu ?</b></summary>

Un nouveau Push Image vers gold-v1 (retour arrière), puis j'analyse le problème sur la golden image ou sur le pool de test.
</details>

<details>
<summary><b>4. Pourquoi les profils itinérants classiques sont-ils déconseillés en VDI ?</b></summary>

Ils copient tout le profil à l'ouverture et à la fermeture de session : c'est lent, ça charge le réseau et le stockage, et ça se corrompt facilement. FSLogix monte un disque à la place, sans copie.
</details>

<details>
<summary><b>5. Où mettre une application utilisée uniquement par le service comptabilité ?</b></summary>

Dans App Volumes (package affecté au groupe AD de la comptabilité), ou en application publiée. Pas dans la golden image commune.
</details>

<details>
<summary><b>6. Pourquoi faut-il l'activation d'ordinateurs partagés pour Microsoft 365 Apps en VDI non persistante ?</b></summary>

Parce que l'activation classique est liée à la machine, qui change à chaque session. L'activation partagée lie la licence à l'utilisateur et fonctionne sur des machines éphémères.
</details>

<details>
<summary><b>7. Pourquoi ne pas utiliser une clé MAK pour activer les clones ?</b></summary>

Chaque clone consommerait une activation du quota MAK. On utilise KMS ou l'activation par Active Directory.
</details>

---

⬅ [05 - Administration](/hyperviseur/VDI/05-horizon-administration) · **Page suivante :** [07 - Horizon en production et dépannage](/hyperviseur/VDI/07-horizon-production) ➜
