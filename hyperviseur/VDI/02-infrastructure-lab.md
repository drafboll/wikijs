---
title: 02 - Mise en place de l'infrastructure du lab
description: Architecture, plan d'adressage, AD/DNS/NTP/DHCP, ESXi imbriqué, vCenter, comptes de service, checklist avant Horizon
published: 1
date: 2026-11-11T13:45:14.660Z
tags: 
editor: markdown
dateCreated: 2026-11-11T13:45:14.660Z
---

# 02 - Mise en place de l'infrastructure du lab

> **Objectif de la page :** construire et **valider** le socle sur lequel Horizon va reposer : réseau, Active Directory, DNS, heure, DHCP, ESXi et vCenter. Une VDI instable vient presque toujours d'un socle mal préparé, donc on prend le temps de tout vérifier ici.
{.is-info}

**Parcours du cours :** [01 Bases](/hyperviseur/VDI/01-bases-virtualisation) ➜ **02 Infrastructure** ➜ [03 VDI & Horizon](/hyperviseur/VDI/03-vdi-et-horizon) ➜ [04 Installation](/hyperviseur/VDI/04-installation-horizon) ➜ [05 Administration](/hyperviseur/VDI/05-horizon-administration) ➜ [06 Golden image](/hyperviseur/VDI/06-horizon-golden-image) ➜ [07 Production](/hyperviseur/VDI/07-horizon-production)

---

## 1. L'architecture cible

![Architecture du lab](/vdi/02-architecture-lab.svg)

Tout le lab vit dans **un seul réseau** : le réseau NAT **VMnet8** de VMware Workstation (`192.168.50.0/24`). En entreprise, on séparerait management, serveurs et bureaux dans des VLAN différents (voir page 07), mais pour apprendre, un réseau plat est plus simple.

### Plan d'adressage

| Machine | Nom d'hôte | FQDN | IP | Rôle |
|---|---|---|---|---|
| Passerelle NAT VMware | - | - | 192.168.50.2 | Sortie Internet |
| Hôte Windows (carte VMnet8) | - | - | 192.168.50.1 | Poste d'administration + client Horizon |
| SRV-GRP3 | SRV-GRP3 | `srv-grp3.srv-grp3.lan` | 192.168.50.10 | AD DS, DNS, NTP, DHCP |
| esx01 | ESX01 | `esx01.srv-grp3.lan` | 192.168.50.11 | Hyperviseur |
| vcsa01 | vcsa01 | `vcsa01.srv-grp3.lan` | 192.168.50.12 | vCenter |
| Horizon | cs01 | `cs01.srv-grp3.lan` | 192.168.50.13 | Connection Server |
| Golden Image | GOLD-W11 (exemple) | - | DHCP (ou .20) | Modèle de bureau |
| Bureaux | vdi-01, vdi-02... | - | DHCP .100 à .199 | Bureaux des utilisateurs |

> **Le nom de VM et le nom d'hôte sont deux choses différentes.** Dans vCenter, ta VM s'appelle « Horizon », mais le Windows à l'intérieur s'appellera `cs01`. C'est le nom d'hôte (et le FQDN) qui compte pour le DNS, les certificats et le client Horizon. En entreprise, on essaie de garder les deux identiques pour éviter les confusions.
{.is-info}

### Budget des ressources

| VM | vCPU | RAM | Disque | Où elle tourne |
|---|---|---|---|---|
| SRV-GRP3 | 2 | 4 Go | existant | Workstation |
| esx01 | 8 | 48 Go | 32 Go (boot) + 300 Go (thin, datastore) | Workstation |
| vcsa01 (Tiny) | 2 | 12 à 14 Go | ~350 Go promis, thin | esx01 |
| cs01 | 4 | 8 Go | 80 Go | esx01 |
| Golden Image | 2 | 4 Go | 60 Go | esx01 (éteinte une fois prête) |
| vdi-01, vdi-02 | 2 | 4 Go chacun | 40 Go (partagés en Instant Clone) | esx01 |

Total dans esx01 : 14 + 8 + 4 + 8 = **34 Go** sur 48 Go : ça laisse de la marge à ESXi lui-même (environ 2 à 4 Go). Sur ton PC : 16 (Windows) + 4 + 48 = **68 Go** sur 80.

> **Le CPU est le vrai goulot de ton lab.** Ton Ryzen 5 5500 a 12 threads. esx01 en reçoit 8, et tout tourne dedans. Les opérations lourdes (déploiement de vCenter, création de pool, mise à jour Windows) seront lentes : c'est normal, sois patient.
{.is-warning}

---

## 2. Vérifications sur ton PC (niveau 0 et 1)

### 2.1 Virtualisation matérielle

1. **BIOS :** `Advanced ➜ CPU Configuration ➜ SVM Mode = Enabled`.
2. **Windows :** ouvre `msinfo32` et regarde en bas de la page « Résumé système » :
   - « Sécurité basée sur la virtualisation » doit être **Non activée** ;
   - si tu vois « Un hyperviseur a été détecté », Hyper-V est actif.
3. Si besoin, désactive dans **Sécurité Windows ➜ Sécurité de l'appareil ➜ Isolation du noyau ➜ Intégrité de la mémoire**, et dans « Fonctionnalités Windows » : Hyper-V, Plateforme de machine virtuelle, Plateforme de l'hyperviseur Windows, WSL2 / Docker Desktop. Puis redémarre.

```powershell
# Dans un PowerShell administrateur : état de l'hyperviseur Windows au démarrage
bcdedit /enum | findstr -i hypervisorlaunchtype
# Pour le désactiver (redémarrage nécessaire) :
bcdedit /set hypervisorlaunchtype off
```

### 2.2 Le réseau VMnet8

Ouvre **Workstation ➜ Edit ➜ Virtual Network Editor** (bouton « Change Settings » en administrateur) :

| Paramètre | Valeur |
|---|---|
| VMnet8 | NAT |
| Subnet IP | 192.168.50.0 |
| Subnet mask | 255.255.255.0 |
| NAT Settings ➜ Gateway | 192.168.50.2 |
| **Use local DHCP service** | **Décoché** (c'est SRV-GRP3 qui fera le DHCP, voir §3.4) |

Ensuite, sur ton Windows, la carte **« VMware Network Adapter VMnet8 »** doit avoir l'IP `192.168.50.1`. Pour que ton PC trouve `cs01.srv-grp3.lan`, mets son **DNS** sur `192.168.50.10` (uniquement sur cette carte, ta carte Internet garde son DNS normal).

> **Alternative sans toucher au DNS de l'hôte :** ajoute des lignes dans `C:\Windows\System32\drivers\etc\hosts` (bloc-notes lancé en administrateur) :
> ```
> 192.168.50.11  esx01.srv-grp3.lan   esx01
> 192.168.50.12  vcsa01.srv-grp3.lan  vcsa01
> 192.168.50.13  cs01.srv-grp3.lan    cs01
> ```
{.is-info}

---

## 3. Les services de base sur SRV-GRP3

![Dépendances AD DNS NTP](/vdi/02-dependances-ad-dns-ntp.svg)

### 3.1 Pourquoi Horizon a besoin d'un Active Directory

- vSphere peut fonctionner sans AD (comptes locaux `root` et `administrator@vsphere.local`).
- **Horizon, non.** Le Connection Server **doit être membre d'un domaine**. Il authentifie les utilisateurs contre l'AD, lit leurs groupes pour savoir à quels bureaux ils ont droit, et crée les comptes ordinateur des bureaux Instant Clone.
- Les bureaux eux-mêmes sont membres du domaine : l'utilisateur ouvre sa session Windows avec son compte AD.

### 3.2 Confirmer le domaine

Sur SRV-GRP3, dans un PowerShell administrateur :

```powershell
Get-ADDomain | Select-Object DNSRoot, NetBIOSName, DistinguishedName, PDCEmulator
Get-ADForest | Select-Object Name, ForestMode
```

Tu devrais voir `DNSRoot = SRV-GRP3.lan` et `DistinguishedName = DC=SRV-GRP3,DC=lan`. **Note bien le `NetBIOSName`** : c'est lui que tu taperas dans « Domaine » sur le client Horizon et dans les comptes au format `NETBIOS\utilisateur`. Dans la suite du cours, j'écris `SRV-GRP3\...`, remplace-le si ton NetBIOS est différent.

> Les noms DNS ne sont pas sensibles à la casse : `SRV-GRP3.lan` et `srv-grp3.lan` sont identiques. J'écris les FQDN en minuscules, c'est la convention.
{.is-info}

### 3.3 Le DNS : enregistrements A et PTR

Chaque serveur doit être résolu **dans les deux sens** :
- **A** (direct) : `cs01.srv-grp3.lan ➜ 192.168.50.13` ;
- **PTR** (inverse) : `192.168.50.13 ➜ cs01.srv-grp3.lan`.

vCenter est particulièrement strict : si le PTR est absent au moment du déploiement, l'installation peut échouer ou produire un certificat incorrect.

```powershell
# 1. Créer la zone inverse si elle n'existe pas
Add-DnsServerPrimaryZone -NetworkId "192.168.50.0/24" -ReplicationScope Forest

# 2. Créer les enregistrements A + PTR
$zone = "SRV-GRP3.lan"
Add-DnsServerResourceRecordA -ZoneName $zone -Name "esx01"  -IPv4Address 192.168.50.11 -CreatePtr
Add-DnsServerResourceRecordA -ZoneName $zone -Name "vcsa01" -IPv4Address 192.168.50.12 -CreatePtr
Add-DnsServerResourceRecordA -ZoneName $zone -Name "cs01"   -IPv4Address 192.168.50.13 -CreatePtr

# 3. Redirecteur vers Internet (pour que les VM résolvent aussi les noms publics)
Add-DnsServerForwarder -IPAddress 192.168.50.2, 8.8.8.8

# 4. Vérifier
Resolve-DnsName cs01.srv-grp3.lan
Resolve-DnsName 192.168.50.13
```

> **Une fois cs01 joint au domaine**, il enregistrera lui-même son nom. Si l'enregistrement A existe déjà avec la bonne IP, aucun problème.
{.is-info}

### 3.4 L'heure (NTP)

L'authentification **Kerberos** de l'AD tolère un écart de **5 minutes maximum** entre les machines. Au-delà : ouvertures de session refusées, jonction au domaine impossible, erreurs de certificat. La chaîne de temps doit être :

```
Internet (pool.ntp.org) ➜ SRV-GRP3 (PDC) ➜ esx01, vcsa01, cs01, bureaux
```

Sur SRV-GRP3 :

```powershell
w32tm /config /manualpeerlist:"0.fr.pool.ntp.org,0x8 1.fr.pool.ntp.org,0x8" /syncfromflags:manual /reliable:yes /update
Restart-Service w32time
w32tm /resync
w32tm /query /status
```

> **Piège classique en VM :** désactive la synchronisation de l'heure par l'hyperviseur sur le contrôleur de domaine (Workstation ➜ VM Settings ➜ Options ➜ VMware Tools ➜ décoche « Synchronize guest time with host »). Un DC doit prendre l'heure du NTP, pas de l'hôte.
{.is-warning}

Les machines du domaine (cs01, bureaux) prennent automatiquement l'heure du DC. Pour ESXi et vCenter, on la configure à la main (§4.4 et §5).

### 3.5 Le DHCP pour les bureaux

Chaque Instant Clone démarre avec une nouvelle identité et demande une IP en **DHCP**. On installe le rôle sur SRV-GRP3 :

```powershell
Install-WindowsFeature DHCP -IncludeManagementTools
Add-DhcpServerInDC -DnsName "srv-grp3.srv-grp3.lan" -IPAddress 192.168.50.10

Add-DhcpServerv4Scope -Name "VDI-LAB" -StartRange 192.168.50.100 -EndRange 192.168.50.199 `
    -SubnetMask 255.255.255.0 -LeaseDuration (New-TimeSpan -Hours 8) -State Active
Set-DhcpServerv4OptionValue -ScopeId 192.168.50.0 -Router 192.168.50.2 `
    -DnsServer 192.168.50.10 -DnsDomain "SRV-GRP3.lan"

Get-DhcpServerv4Scope
```

> **Durée de bail courte :** en VDI non persistante, des bureaux sont détruits et recréés en permanence. Avec un bail de 8 jours (valeur par défaut), la plage se remplit d'adresses « fantômes ». En production, on met souvent **1 à 8 heures**.
{.is-info}

> **Ne garde qu'un seul serveur DHCP.** Si le DHCP de VMware (VMnet8) est encore actif, il répondra aussi et distribuera un DNS qui ne connaît pas ton domaine : jonction au domaine des clones en échec. D'où la case décochée au §2.2.
{.is-danger}

### 3.6 Organisation de l'AD : OU, comptes et groupes

On crée une arborescence propre, comme en entreprise :

```
SRV-GRP3.lan
└── OU=LAB
    ├── OU=Serveurs          cs01 (et plus tard d'autres serveurs Horizon)
    ├── OU=VDI-Ordinateurs   les bureaux vdi-xx (créés par Horizon)
    ├── OU=Utilisateurs      vdiuser1, vdiuser2...
    ├── OU=Groupes           GRP-VDI-USERS, GRP-HZ-ADMINS
    └── OU=Comptes-Service   svc-vcenter, svc-horizon, svc-icdomain
```

| Compte / groupe | Type | Usage |
|---|---|---|
| `svc-vcenter` | Compte de service | vCenter l'utilise pour lire l'AD (source d'identité) |
| `svc-horizon` | Compte de service | Horizon l'utilise pour se connecter à **vCenter** |
| `svc-icdomain` | Compte de service | Horizon l'utilise pour **créer les comptes ordinateur** des Instant Clones dans l'OU VDI-Ordinateurs |
| `vdiuser1`, `vdiuser2` | Utilisateurs de test | Se connecter aux bureaux |
| `GRP-VDI-USERS` | Groupe | Droit d'accès au pool de bureaux |
| `GRP-HZ-ADMINS` | Groupe | Administrateurs de la console Horizon |
| Ton compte admin (`adm-...`) | Utilisateur | Administrer, membre de GRP-HZ-ADMINS |

> **Pourquoi trois comptes de service distincts ?** C'est le **principe du moindre privilège** : chaque compte n'a que les droits dont il a besoin. Si l'un fuit, les dégâts sont limités, et dans les journaux tu sais immédiatement quel composant a fait quoi. Le résumé initial n'en prévoyait que deux : `svc-icdomain` est indispensable pour les Instant Clones (page 04).
{.is-warning}

Script complet (à adapter si besoin) :

```powershell
Import-Module ActiveDirectory
$dn = (Get-ADDomain).DistinguishedName      # DC=SRV-GRP3,DC=lan
$nb = (Get-ADDomain).NetBIOSName

# OU
New-ADOrganizationalUnit -Name "LAB" -Path $dn
$lab = "OU=LAB,$dn"
"Serveurs","VDI-Ordinateurs","Utilisateurs","Groupes","Comptes-Service" |
    ForEach-Object { New-ADOrganizationalUnit -Name $_ -Path $lab }

# Mot de passe saisi de façon masquée
$pwd = Read-Host "Mot de passe des comptes de lab" -AsSecureString

# Comptes de service (mot de passe qui n'expire pas : sinon tout casse dans 42 jours)
"svc-vcenter","svc-horizon","svc-icdomain" | ForEach-Object {
    New-ADUser -Name $_ -SamAccountName $_ -UserPrincipalName "$_@SRV-GRP3.lan" `
        -Path "OU=Comptes-Service,$lab" -AccountPassword $pwd -Enabled $true `
        -PasswordNeverExpires $true -CannotChangePassword $true
}

# Utilisateurs de test
"vdiuser1","vdiuser2" | ForEach-Object {
    New-ADUser -Name $_ -SamAccountName $_ -UserPrincipalName "$_@SRV-GRP3.lan" `
        -Path "OU=Utilisateurs,$lab" -AccountPassword $pwd -Enabled $true
}

# Groupes
New-ADGroup -Name "GRP-VDI-USERS" -GroupScope Global -Path "OU=Groupes,$lab"
New-ADGroup -Name "GRP-HZ-ADMINS" -GroupScope Global -Path "OU=Groupes,$lab"
Add-ADGroupMember "GRP-VDI-USERS" -Members vdiuser1, vdiuser2

# Délégation : svc-icdomain peut créer / supprimer / gérer les ordinateurs de l'OU VDI
$ouVdi = "OU=VDI-Ordinateurs,$lab"
dsacls $ouVdi /I:T /G "$nb\svc-icdomain:CCDC;computer"
dsacls $ouVdi /I:S /G "$nb\svc-icdomain:GA;;computer"
```

> `Read-Host -AsSecureString` évite d'écrire le mot de passe en clair dans le script ou l'historique PowerShell. Garde ces mots de passe dans un gestionnaire (KeePass, Bitwarden...), jamais dans le wiki.
{.is-info}

---

## 4. ESXi (esx01)

### 4.1 La VM dans Workstation

| Paramètre | Valeur | Pourquoi |
|---|---|---|
| Guest OS | VMware ESXi 8 | Workstation choisit le bon matériel virtuel |
| Processeurs | 1 socket × 8 cœurs | Laisse 4 threads à Windows |
| ☑ Virtualize Intel VT-x/EPT or AMD-V/RVI | **Coché** | Sans ça, pas de VM 64 bits dans l'ESXi |
| Mémoire | 48 Go | |
| Carte réseau | NAT (VMnet8), type VMXNET3 | ESXi 8 a abandonné beaucoup de pilotes anciens |
| Disque 1 | 32 Go | Installation d'ESXi |
| Disque 2 | 300 Go, thin | Datastore `ds-esx01` |

### 4.2 Le réseau de management (DCUI)

Sur la console jaune et grise d'ESXi (**DCUI**), touche `F2` ➜ *Configure Management Network* :

| Menu | Valeur |
|---|---|
| Network Adapters | vmnic0 (vérifier qu'il est « Connected ») |
| IPv4 Configuration | Static : `192.168.50.11` / `255.255.255.0`, passerelle `192.168.50.2` |
| DNS Configuration | Primaire `192.168.50.10`, secondaire `8.8.8.8`, hostname `esx01` |
| Custom DNS Suffixes | `srv-grp3.lan` |
| IPv6 | Désactivé (optionnel, évite du bruit en lab, redémarrage requis) |

Valide avec `Échap`, réponds `Y` pour redémarrer le management network, puis **Test Management Network** : les pings vers la passerelle, le DNS et la résolution de `esx01` doivent être `OK`.

> **DNS secondaire 8.8.8.8 :** c'est pratique pour Internet, mais un serveur DNS public ne connaît pas `srv-grp3.lan`. Si SRV-GRP3 est éteint ou répond lentement, ESXi peut demander à 8.8.8.8 et obtenir « nom inconnu ». En entreprise, on ne met **que** des DNS internes. Dans ton lab, c'est acceptable tant que SRV-GRP3 est allumé en premier.
{.is-warning}

### 4.3 Dépanner le ping vers esx01

Le résumé mentionne un problème de ping non résolu. Méthode de diagnostic, du plus simple au plus profond :

| # | Test | Où | Si ça échoue |
|---|---|---|---|
| 1 | La VM esx01 est-elle sur **VMnet8** et « Connected » ? | Workstation ➜ VM Settings ➜ Network Adapter | Cocher « Connected » et « Connect at power on » |
| 2 | La carte VMnet8 de Windows a-t-elle `192.168.50.1` ? | `ipconfig` sur l'hôte | Virtual Network Editor ➜ « Restore Defaults » puis remettre le sous-réseau |
| 3 | ESXi voit-il un lien ? | DCUI ➜ Network Adapters (Status = Connected) | Changer le type de carte (VMXNET3 ⇄ E1000E) dans le `.vmx` : `ethernet0.virtualDev = "vmxnet3"` |
| 4 | L'IP est-elle bien appliquée ? | DCUI ➜ écran d'accueil (affiche l'IP) | Revoir §4.2, *Restart Management Network* |
| 5 | Ping depuis ESXi vers la passerelle | DCUI ➜ `Alt+F1` (shell, si activé) ➜ `vmkping 192.168.50.2` | Problème côté VMnet8 / NAT |
| 6 | Ping depuis l'hôte | `ping 192.168.50.11` | Vérifier qu'aucun autre appareil n'a la même IP (`arp -a`) |
| 7 | Accès web | `https://192.168.50.11/ui` | Si le web marche mais pas le ping : ce n'est pas grave, c'est l'ICMP qui est filtré quelque part |

> ESXi répond au ping par défaut. Si **le web fonctionne**, le réseau est bon : ne perds pas de temps sur le ping.
{.is-success}

### 4.4 Réglages post-installation

Dans le **Host Client** (`https://192.168.50.11/ui`, compte `root`) :

1. **Heure :** *Gérer ➜ Système ➜ Date et heure ➜ Modifier les paramètres NTP* ➜ « Utiliser NTP », serveur `192.168.50.10`, politique « Démarrer et arrêter avec l'hôte », puis *Actions ➜ Démarrer* le service NTP.
2. **Sécurité du vSwitch (obligatoire en nested)** : voir §4.5.
3. **Datastore** `ds-esx01` : déjà créé en VMFS-6 sur le disque de 300 Go.
4. **SSH** (optionnel, utile pour apprendre) : *Gérer ➜ Services ➜ TSM-SSH ➜ Démarrer*. À désactiver en production.

### 4.5 Le réglage réseau indispensable en nested

![Réseau nested et mode promiscuité](/vdi/02-reseau-nested.svg)

Les VM qui tournent dans esx01 (vCenter, cs01, les bureaux) envoient des trames réseau avec **leurs propres adresses MAC**. Or la carte virtuelle d'esx01 dans Workstation n'a qu'**une** MAC. Par défaut, le vSwitch refuse ces trames : les VM imbriquées n'ont alors pas de réseau, ou un réseau intermittent.

**Host Client ➜ Mise en réseau ➜ Commutateurs virtuels ➜ vSwitch0 ➜ Modifier les paramètres ➜ Sécurité :**

| Option | Valeur |
|---|---|
| Mode promiscuité | **Accepter** |
| Modifications d'adresse MAC | **Accepter** |
| Transmissions forgées | **Accepter** |

Fais de même sur le port group **VM Network** s'il ne fait pas hériter ces réglages. En ligne de commande (SSH sur esx01) :

```bash
esxcli network vswitch standard policy security set -v vSwitch0 \
  --allow-promiscuous=true --allow-mac-change=true --allow-forged-transmits=true
esxcli network vswitch standard policy security get -v vSwitch0
```

> **Côté Windows :** Workstation doit pouvoir mettre la carte VMnet8 en mode promiscuité. Sur un hôte Windows, c'est autorisé par défaut. Si les bureaux n'obtiennent toujours pas d'IP DHCP, c'est le premier endroit à revérifier.
{.is-warning}

> **En production, on ne fait JAMAIS ça** sur des ESXi physiques : ces trois options restent sur « Rejeter » pour des raisons de sécurité. C'est uniquement lié au fait que ton ESXi est lui-même une VM.
{.is-danger}

---

## 5. vCenter (vcsa01)

### 5.1 Le déploiement en deux étapes

![Déploiement VCSA](/vdi/02-vcsa-deploiement.svg)

| Étape | Ce qui se passe | Durée en nested |
|---|---|---|
| **Stage 1 : Deploy** | L'installeur (sur ton PC) envoie l'appliance OVA sur esx01, configure son réseau et la démarre | 15 à 40 min |
| **Stage 2 : Set up** | Configuration des services : NTP, SSH, domaine SSO `vsphere.local`, CEIP | 20 à 60 min |

Si le stage 2 est interrompu, tu peux le reprendre depuis `https://192.168.50.12:5480` (interface d'administration de l'appliance, **VAMI**).

> **Si ta VCSA a été déployée avec l'IP comme nom système** (et non `vcsa01.srv-grp3.lan`), le certificat et l'URL utiliseront l'IP. Ça fonctionne, mais Horizon affichera des avertissements de certificat. En lab, tu peux vivre avec. Pour corriger proprement, le plus simple est de redéployer la VCSA avec le FQDN une fois le DNS en place.
{.is-info}

### 5.2 Les URL et comptes de vCenter

| Usage | URL | Compte |
|---|---|---|
| vSphere Client (gestion quotidienne) | `https://vcsa01.srv-grp3.lan/ui` | `administrator@vsphere.local` |
| VAMI (administration de l'appliance : mises à jour, sauvegarde, NTP) | `https://vcsa01.srv-grp3.lan:5480` | `root` |
| Host Client de l'ESXi | `https://esx01.srv-grp3.lan/ui` | `root` |

### 5.3 L'inventaire

Déjà en place d'après ton résumé :

```
vcsa01
└── DC-LAB              (Datacenter)
    └── CL-LAB          (Cluster : DRS off, HA off)
        └── 192.168.50.11 (esx01)
```

Améliorations conseillées :

1. **Dossiers de VM** : clic droit sur `DC-LAB` ➜ *New Folder ➜ New VM and Template Folder* : `Infra` (vcsa01, cs01), `Golden` (Golden Image), `VDI` (où Horizon rangera les bureaux).
2. **NTP de la VCSA** : VAMI ➜ *Time* ➜ NTP `192.168.50.10`.
3. **Licences** : *Administration ➜ Licensing*. En évaluation, tu as un nombre de jours limité (vérifie le compteur). Les conditions Broadcom ont beaucoup changé : vérifie sur le site officiel ce qui s'applique à ton cas.

### 5.4 Brancher vCenter sur l'AD (optionnel mais formateur)

Permet de te connecter à vCenter avec ton compte du domaine.

1. *Administration ➜ Single Sign On ➜ Configuration ➜ Identity Provider ➜ Identity Sources ➜ Add*.
2. Type : **Active Directory over LDAP** (« AD over LDAP »).
3. Paramètres :

| Champ | Valeur |
|---|---|
| Identity source name | SRV-GRP3 |
| Base DN for users | `OU=LAB,DC=SRV-GRP3,DC=lan` |
| Base DN for groups | `OU=LAB,DC=SRV-GRP3,DC=lan` |
| Domain name | `SRV-GRP3.lan` |
| Domain alias | le NetBIOS (ex. `SRV-GRP3`) |
| Username | `svc-vcenter@SRV-GRP3.lan` |
| Primary server URL | `ldap://srv-grp3.srv-grp3.lan:389` |

4. *Administration ➜ Global Permissions ➜ Add* : groupe `GRP-HZ-ADMINS`, rôle `Administrator`, ☑ Propagate to children.

> En production, on utilise **LDAPS (port 636)** pour que le mot de passe ne circule pas en clair. En lab, LDAP simple suffit.
{.is-info}

---

## 6. Ordre de démarrage et d'arrêt du lab

C'est important en nested : si tu démarres dans le désordre, les services ne se trouvent pas.

**Démarrage :**
1. **SRV-GRP3** (attendre la fin du démarrage complet, environ 2 min : AD et DNS doivent être prêts)
2. **esx01** (dans Workstation)
3. **vcsa01** (démarrage automatique conseillé : Host Client ➜ VM ➜ *Démarrage automatique*). Compte **10 à 15 min** avant que le vSphere Client réponde.
4. **cs01**
5. Les bureaux : Horizon les allume lui-même.

**Arrêt :** dans l'ordre inverse. Arrête proprement les VM (« Shut down guest OS »), puis esx01 (*Arrêter* depuis le Host Client), puis SRV-GRP3. Évite de « suspendre » esx01 : l'horloge des VM imbriquées devient incohérente au réveil.

---

## 7. Checklist avant de passer à Horizon

| ✓ | Vérification | Commande / endroit |
|---|---|---|
| ☐ | Le domaine est bien `SRV-GRP3.lan`, NetBIOS noté | `Get-ADDomain` |
| ☐ | Les A et PTR de esx01, vcsa01, cs01 répondent | `Resolve-DnsName` (dans les deux sens) |
| ☐ | L'heure du DC est synchronisée sur Internet | `w32tm /query /status` |
| ☐ | NTP configuré sur esx01 et vcsa01 | Host Client / VAMI |
| ☐ | Une seule source DHCP (Windows), plage .100-.199 active | `Get-DhcpServerv4Scope` |
| ☐ | vSwitch0 en promiscuité / MAC changes / forged = Accepter | Host Client |
| ☐ | vSphere Client accessible par FQDN | `https://vcsa01.srv-grp3.lan/ui` |
| ☐ | esx01 « Connected » dans CL-LAB, pas d'alarme rouge | vSphere Client |
| ☐ | Datastore `ds-esx01` : plus de 150 Go libres | vSphere Client ➜ Storage |
| ☐ | OU, comptes `svc-*`, `vdiuser1`, groupes créés | `Get-ADUser -Filter 'Name -like "svc-*"'` |
| ☐ | Délégation de `svc-icdomain` sur l'OU VDI-Ordinateurs | `dsacls "OU=VDI-Ordinateurs,..."` |
| ☐ | L'hôte Windows résout `cs01.srv-grp3.lan` | `nslookup cs01.srv-grp3.lan` |

---

## 8. Quiz

<details>
<summary><b>1. Pourquoi décocher le DHCP de VMnet8 ?</b></summary>

Pour qu'il n'y ait qu'un seul serveur DHCP (SRV-GRP3). Sinon, les bureaux peuvent recevoir une configuration du DHCP VMware, avec un DNS qui ne connaît pas le domaine, et leur jonction au domaine échoue.
</details>

<details>
<summary><b>2. Quelle est la tolérance de Kerberos sur l'heure, et que se passe-t-il au-delà ?</b></summary>

5 minutes. Au-delà, les authentifications échouent : impossible d'ouvrir une session ou de joindre le domaine.
</details>

<details>
<summary><b>3. Les VM de mon ESXi imbriqué n'ont pas de réseau alors que l'ESXi répond. Pourquoi ?</b></summary>

Le vSwitch rejette les trames des VM imbriquées, dont la MAC diffère de celle de la carte d'esx01. Il faut accepter le mode promiscuité, les changements de MAC et les transmissions forgées.
</details>

<details>
<summary><b>4. À quoi sert svc-icdomain et pourquoi ne pas utiliser le compte administrateur du domaine ?</b></summary>

Horizon l'utilise pour créer, supprimer et réinitialiser les comptes ordinateur des Instant Clones dans l'OU VDI. On limite ses droits à cette OU (moindre privilège) : s'il est compromis, il ne peut rien faire d'autre.
</details>

<details>
<summary><b>5. Dans quel ordre démarrer le lab ?</b></summary>

SRV-GRP3 ➜ esx01 ➜ vcsa01 ➜ cs01 ➜ (bureaux gérés par Horizon).
</details>

---

⬅ [01 - Les bases](/hyperviseur/VDI/01-bases-virtualisation) · **Page suivante :** [03 - La VDI et Horizon](/hyperviseur/VDI/03-vdi-et-horizon) ➜