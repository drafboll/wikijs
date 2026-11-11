---
title: 07 - Horizon en production et dépannage
description: Différences lab / production, haute disponibilité, UAG, certificats, True SSO, Cloud Pod Architecture, dimensionnement, sauvegarde, mises à jour, sécurité, méthode de dépannage, glossaire complet
published: 1
date: 2026-11-11T13:48:57.657Z
tags: 
editor: markdown
dateCreated: 2026-11-11T13:48:57.656Z
---

# 07 - Horizon en production et dépannage

> **Objectif de la page :** comprendre ce qui sépare ton lab d'une vraie plateforme d'entreprise, pour te repérer dès les premiers jours dans ton nouveau poste : accès externe, haute disponibilité, certificats, multi-sites, dimensionnement, sauvegarde, mises à jour, sécurité. La page se termine par une méthode de dépannage complète et le glossaire de tout le cours.
{.is-info}

**Parcours du cours :** [01 Bases](/hyperviseur/VDI/01-bases-virtualisation) ➜ [02 Infrastructure](/hyperviseur/VDI/02-infrastructure-lab) ➜ [03 VDI & Horizon](/hyperviseur/VDI/03-vdi-et-horizon) ➜ [04 Installation](/hyperviseur/VDI/04-installation-horizon) ➜ [05 Administration](/hyperviseur/VDI/05-horizon-administration) ➜ [06 Golden image](/hyperviseur/VDI/06-horizon-golden-image) ➜ **07 Production**

---

## 1. Du lab à la production

| Sujet | Ton lab | En production |
|---|---|---|
| Hôtes ESXi | 1, imbriqué | Plusieurs clusters physiques : **management** et **bureaux** séparés |
| Stockage | Un fichier VMDK sur ton disque | vSAN all-flash ou baie SAN / NAS, plusieurs datastores |
| Réseau | Un seul réseau plat NAT | VLAN séparés (management, serveurs, bureaux, DMZ), pare-feu, souvent NSX |
| Connection Servers | 1 | 2 à 7 par pod, derrière un **répartiteur de charge** |
| Accès externe | Aucun | **UAG** en DMZ (au moins 2), avec **MFA** |
| Certificats | Auto-signés | Émis par une autorité (interne ou publique) |
| DHCP / DNS / AD | Sur un seul DC | Plusieurs DC, DHCP redondant (failover) |
| Base Events | Aucune | SQL Server ou PostgreSQL |
| Profils | Locaux ou FSLogix sur le DC | FSLogix / DEM sur serveurs de fichiers redondants (DFS, cluster, NAS) |
| Supervision | La console | Outils dédiés, alertes, SIEM |
| Sauvegarde | Aucune | Quotidienne et testée |
| Changements | Quand tu veux | Processus de changement (CAB), fenêtres de maintenance |

---

## 2. L'architecture de référence

![Architecture de production](/vdi/07-architecture-production.svg)

### 2.1 Les principes

- **Clusters séparés.** Le cluster *management* héberge les serveurs d'infrastructure (vCenter, Connection Servers, App Volumes, SQL, fichiers). Les clusters *bureaux* (ou *resource clusters*) n'hébergent que les bureaux et serveurs RDS. Une tempête de démarrages de bureaux ne doit jamais ralentir les Connection Servers.
- **Souvent un vCenter dédié à la VDI**, séparé de celui des serveurs de l'entreprise : Horizon le sollicite énormément (création et suppression permanentes de VM).
- **Redondance partout** : au moins 2 de chaque composant (Connection Servers, UAG, App Volumes Managers, Enrollment Servers), avec des **règles d'anti-affinité** DRS pour ne pas les mettre sur le même hôte.
- **N+1** sur les clusters de bureaux : si un hôte tombe, les autres absorbent ses bureaux.
- **Le modèle « bloc et pod »** : on ajoute de la capacité par blocs standardisés (un cluster de bureaux = un bloc), et on regroupe les blocs dans un pod.

### 2.2 Les limites à connaître

Omnissa publie des **maximums de configuration** pour chaque version (nombre de sessions par Connection Server, par pod, nombre de pods en CPA, VM par vCenter...). Ordres de grandeur :

| Élément | Ordre de grandeur |
|---|---|
| Connection Servers par pod | jusqu'à 7 |
| Sessions par Connection Server | quelques milliers |
| Sessions par pod | environ 20 000 |
| Pods dans une fédération CPA | plusieurs dizaines |

> Vérifie toujours ces valeurs dans la documentation de **ta** version : elles évoluent à chaque release.
{.is-warning}

---

## 3. La haute disponibilité des Connection Servers

### 3.1 Les serveurs réplicas

On installe d'autres Connection Servers avec l'option **Replica Server**, en indiquant le nom d'un Connection Server existant. Ils répliquent automatiquement la base AD LDS : la configuration est la même partout, et n'importe lequel peut répondre aux utilisateurs.

> **TP possible dans ton lab (si la RAM le permet) :** une VM `cs02` (192.168.50.14, 6 Go) jointe au domaine, installation en *Replica Server* pointant sur `cs01.srv-grp3.lan`. Tu verras `cs02` apparaître dans *Settings ➜ Servers ➜ Connection Servers*, et une modification faite sur l'un est visible sur l'autre.
{.is-success}

### 3.2 Le répartiteur de charge

Devant les Connection Servers, un répartiteur de charge présente une **adresse unique** (VIP), par exemple `vdi.entreprise.local`.

| Réglage | Recommandation |
|---|---|
| Ports | TCP 443 (et 8443 / 4172 si les passerelles des CS sont utilisées) |
| **Persistance (affinité)** | **Indispensable** : un client doit rester sur le même Connection Server pendant toute sa session (affinité par IP source ou cookie) |
| Surveillance de santé | Une requête HTTPS vers une URL de santé du Connection Server (voir la doc de ton répartiteur et d'Horizon) |
| Certificat | Le certificat des Connection Servers doit contenir le **nom de la VIP** (SAN) |
| Paramètre Horizon | Dans chaque Connection Server : *External URL* et, selon la version, liste des origines autorisées (*CORS / Origin check*) pour accepter le nom de la VIP |

---

## 4. L'accès externe avec l'Unified Access Gateway (UAG)

### 4.1 Le rôle de l'UAG

L'UAG est une **appliance Linux durcie**, placée en DMZ. Il permet aux télétravailleurs d'accéder à leurs bureaux **sans VPN** :

1. il présente le portail Horizon sur Internet (443) ;
2. il **authentifie** l'utilisateur, souvent avec une **MFA** (RADIUS, SAML, certificat) **avant** même que la requête n'atteigne le réseau interne ;
3. il relaie les échanges avec les Connection Servers ;
4. il relaie les flux de protocole (Blast 8443, PCoIP 4172) vers les bureaux.

### 4.2 Les cartes réseau (« arms »)

| Mode | Cartes | Usage |
|---|---|---|
| **One-arm** | 1 carte | Une seule interface en DMZ pour tout. Le plus simple. |
| **Two-arm** | 2 cartes | Une vers Internet, une vers l'interne (management + backend) |
| **Three-arm** | 3 cartes | Internet, management, backend séparés. Le plus cloisonné. |

### 4.3 Le déploiement

| Étape | Détail |
|---|---|
| Déploiement de l'OVA | Via vSphere Client, ou **mieux** avec le script PowerShell `uagdeploy.ps1` et un fichier `.ini` (reproductible, versionnable) |
| Console d'administration | `https://ip-de-l-uag:9443/admin` |
| Paramètres Horizon (*Edge Service Settings*) | URL du Connection Server (ou de la VIP interne) et son **empreinte de certificat** ; *Blast External URL* `https://vdi.entreprise.fr:8443` ; *PCoIP External URL* `IP_publique:4172` ; tunnel activé |
| Certificat | Certificat public pour `vdi.entreprise.fr` |
| Authentification | RADIUS (MFA), SAML (Omnissa Access, Entra ID, Okta...), certificats / cartes à puce |
| Haute disponibilité | 2 UAG ou plus derrière un répartiteur externe (ou la HA intégrée à l'UAG) |
| Sauvegarde | Export de la configuration (JSON / INI) depuis la console d'administration |

### 4.4 Les flux à ouvrir (vue simplifiée)

| De | Vers | Ports |
|---|---|---|
| Internet | UAG | TCP 443, TCP + UDP 8443 (Blast), TCP + UDP 4172 (PCoIP si utilisé) |
| UAG | Connection Servers | TCP 443 |
| UAG | Bureaux (VLAN VDI) | TCP + UDP 22443 (Blast), TCP + UDP 4172 (PCoIP), TCP 32111 (USB), TCP 9427 (MMR / CDR) |
| Admin | UAG | TCP 9443 |

> La liste officielle et à jour des ports d'Horizon est publiée par Omnissa (document « Network Ports in Horizon »). C'est **la** référence à donner à l'équipe réseau.
{.is-info}

---

## 5. Les certificats

### 5.1 Où en trouve-t-on ?

| Composant | Certificat | Remarque |
|---|---|---|
| Connection Server | Friendly name `vdm`, magasin *Ordinateur local ➜ Personnel* | SAN : FQDN du serveur + FQDN de la VIP |
| UAG | Certificat public | Pour `vdi.entreprise.fr` |
| vCenter | Géré par la VMCA (autorité intégrée) | Horizon doit l'accepter (empreinte) |
| Enrollment Server / ADCS | Certificats d'inscription | Pour True SSO |
| Bureaux (agent) | Auto-signé par défaut | Peut être remplacé (environnements sensibles) |

### 5.2 Remplacer le certificat d'un Connection Server

1. **Demander le certificat** (avec une autorité interne ADCS, via `certlm.msc` ➜ *Personnel ➜ Certificats ➜ Toutes les tâches ➜ Demander un nouveau certificat*), modèle de type *Serveur Web* avec :
   - Nom du sujet : `CN=cs01.srv-grp3.lan` ;
   - SAN (DNS) : `cs01.srv-grp3.lan`, et le nom de la VIP le cas échéant ;
   - **clé privée exportable**.
2. Dans `certlm.msc` : sur l'**ancien** certificat, *Propriétés* ➜ renomme le nom convivial (par exemple `vdm-old`).
3. Sur le **nouveau** certificat : nom convivial **`vdm`**.
4. Redémarre le service **Horizon Connection Server** (ou le serveur).
5. Vérifie dans la console (*Monitor ➜ Dashboard*) que le certificat est reconnu comme valide.

> **Pièges classiques :** clé privée non exportable, ancien certificat encore nommé `vdm` (deux certificats avec ce nom), chaîne de l'autorité absente des postes clients, nom de la VIP oublié dans le SAN.
{.is-warning}

---

## 6. True SSO

**Le problème :** quand un utilisateur s'authentifie par **SAML** (portail Omnissa Access, Entra ID...) ou par MFA sans mot de passe, Horizon ne connaît pas son mot de passe AD. Il ne peut donc pas ouvrir automatiquement sa session Windows : l'utilisateur devrait le retaper.

**La solution :** True SSO.
1. L'utilisateur s'authentifie (SAML, MFA...).
2. Le Connection Server demande à l'**Enrollment Server** un certificat pour cet utilisateur.
3. L'Enrollment Server l'obtient auprès de l'autorité de certification d'entreprise (**ADCS**), avec un modèle de certificat dédié (type carte à puce).
4. Le bureau ouvre la session Windows avec ce certificat éphémère, comme une carte à puce virtuelle.

Composants nécessaires : ADCS (autorité d'entreprise), modèle de certificat True SSO, 2 Enrollment Servers, et la configuration dans Horizon (`vdmUtil` en ligne de commande, ou la console selon les versions).

---

## 7. Cloud Pod Architecture (CPA) : le multi-sites

![Cloud Pod Architecture](/vdi/07-cpa.svg)

### 7.1 Pourquoi

- **Plus de capacité** que ce qu'un pod supporte.
- **Plusieurs datacenters** : les utilisateurs sont servis au plus près, et un site peut prendre le relais si l'autre tombe (plan de reprise d'activité).

### 7.2 Comment

| Étape | Où |
|---|---|
| Initialiser la fédération sur le premier pod | *Settings ➜ Cloud Pod Architecture ➜ Initialize* |
| Joindre les autres pods | *Join*, depuis un Connection Server du pod à ajouter |
| Créer des **sites** et y ranger les pods | *Settings ➜ Sites* |
| Créer des **global entitlements** | *Inventory ➜ Global Entitlements* : regrouper des pools de plusieurs pods sous un droit unique |
| Définir la **portée** | *All sites*, *Within site* ou *Within pod* : où chercher un bureau pour l'utilisateur |
| Optionnel : **home site** | Site préféré d'un utilisateur ou d'un groupe |

Une base **LDAP globale**, répliquée entre tous les Connection Servers de la fédération, stocke les droits globaux et l'emplacement des sessions. Résultat : l'utilisateur voit une seule icône, retrouve sa session où qu'il se connecte, et bascule sur l'autre site en cas de panne.

> Devant une fédération CPA, on place un **GSLB** (répartition globale, souvent basée sur le DNS) qui envoie l'utilisateur vers le site le plus proche ou disponible.
{.is-info}

---

## 8. Dimensionner une plateforme

### 8.1 Les profils d'utilisateurs

| Profil | Usage | Bureau type |
|---|---|---|
| **Task worker** | Une ou deux applications (centre d'appels, saisie) | RDS, ou 2 vCPU / 4 Go |
| **Knowledge worker** | Bureautique, navigateur, Teams, applications métier | 2 vCPU / 6-8 Go |
| **Power user** | Développeurs, analystes, beaucoup d'applications simultanées | 4 vCPU / 8-16 Go |
| **Graphiste / CAO** | 3D, vidéo | 4-8 vCPU / 16-32 Go + vGPU |

### 8.2 Exemple de calcul

> 500 knowledge workers, 2 vCPU / 6 Go chacun, concurrence de 100 % (tout le monde en même temps). Hôtes : 2 processeurs × 32 cœurs = 64 cœurs, 1 024 Go de RAM.
{.is-info}

| Ressource | Calcul | Bureaux par hôte |
|---|---|---|
| **RAM** | (1 024 Go − ~64 Go pour ESXi et marge) / 6 Go | ≈ 160 |
| **CPU** | 64 cœurs × 6 vCPU par cœur (ratio courant en bureautique) / 2 vCPU | ≈ 192 |
| **Retenu** | le plus petit des deux | **160** |
| **Nombre d'hôtes** | 500 / 160 = 3,1 ➜ 4 hôtes, **+1 pour N+1** | **5 hôtes** |

À compléter par :
- **Stockage** : capacité (replicas, deltas, swap, App Volumes) et surtout **performances** (IOPS et latence) aux pics de démarrage (*boot storm*) et d'ouverture de session (*login storm*) du matin. Le all-flash est la norme aujourd'hui.
- **Réseau** : bande passante par utilisateur (page 03) × nombre de sessions par site.
- **DHCP** : une adresse par bureau + marge, baux courts.
- **Licences** : Horizon (par utilisateur nommé ou par session simultanée selon le contrat), Windows (VDA / M365), vSphere.

> **Mesurer plutôt que deviner :** avant un projet, on fait un **assessment** des postes existants (outils comme Lakeside SysTrack, Liquidware Stratusphere, ou les données de supervision) pendant quelques semaines, puis on valide avec un **pilote** et un outil de charge (par exemple Login VSI / Login Enterprise).
{.is-success}

---

## 9. Sauvegarder et restaurer

| Quoi | Comment | Fréquence |
|---|---|---|
| **Configuration Horizon** (AD LDS) | Sauvegarde automatique des Connection Servers (*Settings ➜ Servers ➜ Connection Servers ➜ Backup*) ; export manuel avec `vdmexport` | Quotidienne + avant chaque changement |
| **vCenter** | Sauvegarde basée sur les fichiers depuis la VAMI (`https://vcsa:5480` ➜ *Backup*) vers un partage SFTP / SMB / FTP | Quotidienne |
| **Golden images** | Sauvegarde de la VM (outil de sauvegarde : Veeam, Commvault...) ou export OVF | À chaque nouvelle version |
| **Profils et données** | Sauvegarde des partages (FSLogix, DEM, redirection) | Quotidienne |
| **Bases SQL** (Events, App Volumes) | Sauvegardes SQL | Quotidienne |
| **Packages App Volumes** | Sauvegarde des datastores / export | À chaque nouveau package |
| **UAG** | Export de la configuration (JSON / INI) | À chaque changement |
| **Bureaux Instant Clone** | **Rien** : ils sont jetables et recréés depuis l'image | - |

Exemple d'export de la configuration Horizon (sur un Connection Server, dans le dossier des outils d'Horizon, `...\Server\tools\bin`) :

```bat
vdmexport -f C:\Backup\horizon-config.LDF
```

Le fichier est chiffré avec le **mot de passe de récupération** saisi à l'installation (étape 3 de la page 04). Pour restaurer : `vdmimport`, en suivant la procédure de la documentation (elle impose d'arrêter les services et de restaurer sur tous les Connection Servers).

> **Les snapshots de Connection Servers sont dangereux** quand il y a plusieurs réplicas : revenir à un snapshot d'un seul serveur désynchronise la réplication AD LDS (problème de « USN rollback »). Si tu dois prendre des snapshots avant une mise à jour, **arrête tous les Connection Servers du pod et snapshotte-les ensemble**, ou appuie-toi sur `vdmexport`.
{.is-danger}

---

## 10. Les mises à jour

### 10.1 L'ordre de mise à jour d'Horizon

```
1. Lire les notes de version + vérifier l'interopérabilité (vSphere, Windows, App Volumes, DEM, UAG, clients)
2. Sauvegarder (vdmexport, vCenter, bases SQL)
3. Connection Servers : un par un (les autres continuent de servir les utilisateurs)
4. Enrollment Servers
5. UAG (on déploie une nouvelle appliance et on importe la configuration)
6. Horizon Agent dans les golden images ➜ test ➜ Push Image
7. Horizon Client sur les postes (déploiement logiciel)
8. Supprimer les anciennes versions, documenter
```

### 10.2 Le rythme des versions

Omnissa publie plusieurs versions d'Horizon 8 par an (numérotées AAMM), dont certaines en **ESB** (*Extended Service Branch*), maintenues plus longtemps. Beaucoup d'entreprises restent sur les versions ESB pour limiter le nombre de mises à jour majeures. Renseigne-toi sur la politique de ton employeur.

### 10.3 vSphere

Ordre classique : **vCenter d'abord**, puis les **ESXi** (un par un, en mode maintenance avec vMotion), puis les VMware Tools et la version matérielle des VM. Toujours après avoir vérifié la compatibilité avec la version d'Horizon.

---

## 11. La sécurité d'une plateforme VDI

| Domaine | Mesures |
|---|---|
| **Accès** | MFA obligatoire depuis l'extérieur (UAG + RADIUS / SAML), restriction des clients autorisés, versions minimales du client |
| **Fuite de données** | Presse-papiers, USB, lecteurs clients et impression contrôlés par GPO / Smart Policies, selon le contexte (interne / externe) |
| **Filigrane** | L'agent Horizon peut afficher un **filigrane numérique** (nom d'utilisateur, date...) sur le bureau pour dissuader les photos d'écran (paramètre GPO) |
| **Réseau** | VLAN VDI isolés, **micro-segmentation** (NSX ou pare-feu des postes) pour que les bureaux ne puissent pas communiquer entre eux |
| **Administration** | Rôles Horizon au moindre privilège, comptes d'administration séparés des comptes de bureautique, MFA sur la console |
| **Durcissement** | Guides de durcissement Omnissa et CIS pour Windows ; protocoles TLS récents uniquement |
| **Correctifs** | Golden images patchées chaque mois ; Connection Servers, UAG, vSphere maintenus à jour (les UAG exposés à Internet en priorité) |
| **Traçabilité** | Événements Horizon et UAG envoyés au **SIEM** (syslog), base Events conservée |
| **Comptes de service** | Mots de passe longs et gérés, droits minimaux (comme `svc-horizon` et `svc-icdomain`) |

---

## 12. La méthode de dépannage

![Méthode de dépannage](/vdi/07-depannage-methode.svg)

### 12.1 Les bons réflexes

1. **Faire préciser le symptôme** : message exact (capture d'écran), heure, poste utilisé, interne ou externe, un seul utilisateur ou plusieurs ?
2. **Délimiter** : un utilisateur ➜ son compte, son profil, son poste. Un pool ➜ l'image, le provisioning. Tout le monde ➜ l'infrastructure (Connection Servers, UAG, vCenter, AD, réseau, certificats).
3. **Situer l'étape** du parcours de connexion (schéma ci-dessus).
4. **Lire ce qu'Horizon dit déjà** : recherche d'utilisateur, état de la machine (survol), *Events*, *Problem Machines*, *Dashboard*.
5. **Qu'est-ce qui a changé ?** Push Image récent, mise à jour, certificat, GPO, mot de passe d'un compte de service, changement réseau. Dans 80 % des cas, la cause est un changement récent.
6. **Tester au plus simple** : un autre utilisateur, un autre poste, HTML Access au lieu du client, PCoIP au lieu de Blast, interne au lieu d'externe.
7. **Documenter** la cause et la solution dans le wiki : la prochaine fois, ce sera rapide.

### 12.2 Cas pratiques

<details>
<summary><b>Cas 1 : lundi matin, plus personne ne peut se connecter. Message « certificat non valide ».</b></summary>

**Piste :** certificat du Connection Server, de la VIP ou de l'UAG **expiré** pendant le week-end.
**Vérification :** ouvrir l'URL dans un navigateur, regarder la date d'expiration ; *Monitor ➜ Dashboard*.
**Solution :** renouveler le certificat (§5), puis mettre en place une alerte d'expiration à J-30.
</details>

<details>
<summary><b>Cas 2 : après un Push Image, toutes les machines du pool sont en « Customization failed ».</b></summary>

**Piste :** ClonePrep n'arrive pas à finir : souvent la jonction au domaine.
**Vérifications :** le mot de passe de `svc-icdomain` a-t-il changé (ou expiré) ? L'OU existe-t-elle encore ? La golden image obtient-elle une IP et le bon DNS ? L'agent a-t-il été réinstallé avec l'option Instant Clone ? L'heure de la golden image est-elle correcte ?
**Solution :** corriger la cause, puis faire un *Recover* des machines ou un nouveau Push Image (ou revenir à l'image précédente).
</details>

<details>
<summary><b>Cas 3 : un utilisateur obtient un écran noir puis une déconnexion. Les autres n'ont aucun problème.</b></summary>

**Piste :** son poste ou son réseau (pare-feu domestique, Wi-Fi d'hôtel qui bloque UDP, vieux client Horizon).
**Vérifications :** version du client, test en HTML Access, test sur un autre réseau, journaux du client.
**Solution :** mettre à jour le client ; si UDP est filtré, Blast doit basculer en TCP (vérifier la configuration BEAT et de l'UAG).
</details>

<details>
<summary><b>Cas 4 : « Bienvenue » pendant 3 minutes à chaque ouverture de session.</b></summary>

**Piste :** profil ou GPO.
**Vérifications :** *logon breakdown* dans l'outil Helpdesk (quelle phase est longue ?), taille et verrouillage du VHDX FSLogix (journaux FSLogix dans `C:\ProgramData\FSLogix\Logs`), scripts d'ouverture de session, lecteurs réseau vers un serveur injoignable, imprimantes.
**Solution :** selon la phase en cause. Un VHDX encore monté sur un autre bureau (session mal fermée) est un classique.
</details>

<details>
<summary><b>Cas 5 : le pool ne crée plus de machines, les utilisateurs reçoivent « aucune machine disponible ».</b></summary>

**Piste :** provisioning bloqué.
**Vérifications :** *Summary* du pool (provisioning désactivé suite à une erreur ?), espace libre des datastores, état de vCenter dans le dashboard (mot de passe de `svc-horizon` changé ?), *Recent Tasks* de vCenter, plage DHCP pleine.
**Solution :** corriger la cause, réactiver le provisioning.
</details>

<details>
<summary><b>Cas 6 : la visio Teams est saccadée, le CPU de la VM est à 100 % pendant les appels.</b></summary>

**Piste :** l'optimisation multimédia n'est pas active, donc la vidéo est encodée par le CPU de la VM.
**Vérifications :** dans Teams, l'état de l'optimisation ; version du client Horizon et du composant côté agent ; GPO d'optimisation Teams.
**Solution :** activer et mettre à jour l'optimisation ; vérifier la compatibilité de la version de Teams.
</details>

### 12.3 Les outils et journaux

| Besoin | Outil |
|---|---|
| Vue d'ensemble | Horizon Console : *Dashboard*, *Events*, *Problem Machines*, recherche d'utilisateur |
| Détail d'une session | Outil Helpdesk, Horizon Performance Tracker |
| Journaux du Connection Server | Dossier de logs d'Horizon (`...\ProgramData\...\VDM\logs`), raccourci de génération d'un paquet de journaux pour le support |
| Journaux de l'agent | Même dossier dans le bureau (`debug-*.txt`) |
| Journaux du client | Dossier temporaire de l'utilisateur, ou collecte depuis le client (*Collect support data*) |
| Journaux FSLogix | `C:\ProgramData\FSLogix\Logs` |
| Infrastructure | vSphere Client (*Tasks*, *Events*, *Performance*), journaux ESXi (`/var/log/hostd.log`, `vmkernel.log`) |
| Réseau | `Test-NetConnection`, `nslookup`, `tracert`, capture réseau (Wireshark) |
| Ligne de commande Horizon | `vdmadmin` (sur les Connection Servers, pour des opérations avancées) et l'API REST |

---

## 13. Bien démarrer dans ton nouveau poste

### 13.1 Les questions à poser à tes collègues

| Sujet | Question |
|---|---|
| Périmètre | Combien d'utilisateurs, de pools, de golden images, de sites ? VDI, RDS, ou les deux ? |
| Versions | Quelle version d'Horizon, de vSphere, d'UAG, d'App Volumes, de DEM ? Branche ESB ou non ? |
| Architecture | Où est le schéma d'architecture ? Combien de pods, de Connection Servers ? CPA ? |
| Accès externe | UAG ? Quel MFA ? Quel nom public ? |
| Profils | FSLogix, DEM, les deux ? Où sont stockés les profils ? |
| Applications | Golden image, App Volumes, applications publiées : qui décide où va quoi ? |
| Processus | Comment se passe la mise à jour mensuelle des images ? Qui valide ? Y a-t-il un pool de test ? |
| Changements | Quel est le processus de changement (CAB, tickets) ? Les fenêtres de maintenance ? |
| Supervision | Quels outils ? Qui reçoit les alertes ? |
| Support | Quels sont les incidents les plus fréquents ? Y a-t-il une base de connaissances ? |
| Documentation | Où sont les procédures, les comptes de service, les certificats et leurs dates d'expiration ? |
| Contrats | Licences (Horizon, Windows, vSphere), contrat de support Omnissa et Broadcom ? |

### 13.2 Continuer à apprendre

| Ressource | Contenu |
|---|---|
| **Documentation Omnissa** (docs.omnissa.com) | Installation, administration, ports, maximums, notes de version |
| **Omnissa Tech Zone** | Architectures de référence, guides de bonnes pratiques, tutoriels très détaillés |
| **Documentation Broadcom / VMware** | vSphere, vSAN, NSX |
| **Documentation Microsoft** | FSLogix, licences VDI, optimisation de Windows |
| Certifications Omnissa | Parcours de certification Horizon (successeurs de l'ancienne VCP-DTM de VMware) |
| Communautés | Forums Omnissa, blogs d'experts EUC, groupes d'utilisateurs |

> **Ton lab est ton meilleur outil.** Avant de faire une opération en production (Push Image, mise à jour, nouvelle GPO), refais-la dans ton lab. Et quand quelque chose casse en production, essaie de le reproduire dans le lab pour comprendre.
{.is-success}

---

## 14. Glossaire complet du cours

| Terme | Définition |
|---|---|
| **AD DS** | Active Directory Domain Services : annuaire des comptes, groupes et ordinateurs |
| **AD LDS** | Lightweight Directory Services : base de configuration locale des Connection Servers |
| **Agent (Horizon Agent)** | Logiciel installé dans chaque bureau ou serveur RDS |
| **App Volumes** | Livraison d'applications par disques virtuels attachés |
| **Application pool** | Application publiée aux utilisateurs |
| **Assignment** | Affectation (machine à un utilisateur, ou package App Volumes à un groupe) |
| **BEAT** | Transport adaptatif de Blast (UDP / TCP) |
| **Blast Extreme** | Protocole d'affichage par défaut d'Horizon |
| **Boot storm / Login storm** | Pic de charge quand de nombreux bureaux démarrent ou ouvrent une session en même temps |
| **Broker** | Rôle du Connection Server : mettre en relation utilisateur et bureau |
| **ClonePrep** | Personnalisation rapide des Instant Clones (nom, domaine), sans Sysprep |
| **Cluster** | Groupe d'hôtes ESXi partageant leurs ressources |
| **Connection Server** | Serveur central d'Horizon (authentification, droits, provisioning, console) |
| **cp-template / cp-replica / cp-parent** | VM internes utilisées pour fabriquer les Instant Clones |
| **CPA** | Cloud Pod Architecture, fédération de plusieurs pods |
| **CPU Ready** | Temps pendant lequel une VM attend un processeur physique |
| **DaaS** | Desktop as a Service, bureaux hébergés dans le cloud |
| **Datastore** | Espace de stockage des VM |
| **DEM** | Dynamic Environment Manager, gestion des paramètres utilisateurs |
| **Dédié / Flottant** | Machine attribuée à vie / n'importe quelle machine libre |
| **DRS** | Répartition automatique de la charge dans un cluster |
| **Enrollment Server** | Serveur délivrant les certificats pour True SSO |
| **Entitlement** | Droit d'accès d'un utilisateur ou groupe à un pool |
| **ESB** | Extended Service Branch, version d'Horizon maintenue plus longtemps |
| **ESXi** | Hyperviseur de type 1 de VMware (Broadcom) |
| **Farm** | Groupe de serveurs RDSH |
| **FSLogix** | Solution Microsoft de profils dans des disques VHDX |
| **Full Clone** | Copie complète et indépendante d'une VM |
| **Global entitlement** | Droit valable sur plusieurs pods (CPA) |
| **Golden image** | VM modèle des bureaux |
| **GSLB** | Répartition globale de charge entre sites |
| **HA** | Redémarrage automatique des VM d'un hôte en panne |
| **HTML Access** | Accès au bureau par un navigateur web |
| **Horizon Client** | Application de l'utilisateur pour se connecter |
| **Horizon Console** | Interface web d'administration |
| **Hyperviseur** | Logiciel qui crée et exécute des VM |
| **Instant Clone** | Clone créé en quelques secondes par « fork » d'une VM parente |
| **JMS** | Système de messages entre Connection Servers et agents |
| **Nested** | Hyperviseur exécuté dans une VM |
| **Non persistant** | Bureau remis à zéro à chaque session |
| **OSOT** | OS Optimization Tool, optimisation de Windows pour la VDI |
| **PCoIP** | Protocole d'affichage historique |
| **Persistant** | Bureau qui conserve ses modifications |
| **Pod** | Ensemble de Connection Servers partageant la même configuration |
| **Pool** (desktop pool) | Ensemble de bureaux identiques |
| **Port group** | Groupe de ports d'un vSwitch avec une même configuration |
| **Push Image** | Mise à jour d'un pool vers un nouveau snapshot ou une nouvelle image |
| **RDS / RDSH** | Remote Desktop Services / Session Host, bureaux Windows Server partagés |
| **Replica Server** | Connection Server supplémentaire d'un pod |
| **Smart Policies** | Politiques DEM appliquées selon le contexte de connexion |
| **Snapshot** | État figé d'une VM via un disque delta |
| **Spare machines** | Machines prêtes d'avance dans un pool |
| **Thin / Thick** | Disque alloué à l'usage / réservé d'avance |
| **True SSO** | Ouverture de session Windows par certificat après une authentification SAML |
| **UAG** | Unified Access Gateway, passerelle d'accès externe |
| **vCenter / VCSA** | Gestion centralisée de vSphere / son appliance |
| **VDI** | Virtual Desktop Infrastructure, bureaux virtuels au datacenter |
| **vGPU** | Partage d'une carte graphique physique entre plusieurs VM |
| **VIP** | Adresse IP virtuelle d'un répartiteur de charge |
| **VMFS** | Système de fichiers en cluster de VMware |
| **VMkernel (vmk)** | Interface réseau de l'ESXi |
| **vMotion** | Migration à chaud d'une VM entre hôtes |
| **VMware Tools** | Pilotes et services dans la VM |
| **vSAN** | Stockage hyperconvergé de VMware |
| **vSwitch / vDS** | Switch virtuel standard / distribué |
| **vTPM** | Puce TPM virtuelle (nécessaire pour Windows 11) |

---

## 15. Quiz final

<details>
<summary><b>1. Pourquoi sépare-t-on un cluster de management et des clusters de bureaux ?</b></summary>

Pour que les pics de charge des bureaux (démarrages, ouvertures de session) n'affectent jamais les serveurs d'infrastructure (Connection Servers, vCenter, SQL), et pour faire évoluer la capacité des bureaux par blocs.
</details>

<details>
<summary><b>2. Quel réglage est indispensable sur le répartiteur de charge devant les Connection Servers ?</b></summary>

La persistance (affinité) : un client doit rester sur le même Connection Server pendant toute sa session.
</details>

<details>
<summary><b>3. Quels ports ouvrir depuis Internet vers l'UAG pour Blast ?</b></summary>

TCP 443 et TCP + UDP 8443.
</details>

<details>
<summary><b>4. Quel nom convivial doit porter le certificat d'un Connection Server ?</b></summary>

`vdm`, avec une clé privée exportable, dans le magasin Personnel de l'ordinateur local.
</details>

<details>
<summary><b>5. À quoi sert True SSO ?</b></summary>

À ouvrir la session Windows sans mot de passe AD quand l'utilisateur s'est authentifié autrement (SAML, MFA), grâce à un certificat éphémère délivré par l'Enrollment Server et ADCS.
</details>

<details>
<summary><b>6. Pourquoi ne faut-il pas restaurer le snapshot d'un seul Connection Server dans un pod de plusieurs serveurs ?</b></summary>

Cela désynchronise la réplication AD LDS entre les serveurs (USN rollback). On sauvegarde avec vdmexport, ou on snapshotte tous les Connection Servers éteints en même temps.
</details>

<details>
<summary><b>7. Dans quel ordre mettre à jour Horizon ?</b></summary>

Connection Servers (un par un) ➜ Enrollment Servers ➜ UAG ➜ agents dans les golden images (puis Push Image) ➜ clients.
</details>

<details>
<summary><b>8. 500 utilisateurs, 160 bureaux maximum par hôte : combien d'hôtes ?</b></summary>

500 / 160 = 3,1, donc 4 hôtes, plus 1 pour la tolérance de panne (N+1) : 5 hôtes.
</details>

<details>
<summary><b>9. Lundi matin, personne ne peut se connecter. Quelles sont les trois premières choses à vérifier ?</b></summary>

Le dashboard Horizon (santé des composants), les certificats (expiration), et ce qui a changé pendant le week-end (mises à jour, Push Image, réseau, mots de passe de comptes de service).
</details>

---

⬅ [06 - Golden image](/hyperviseur/VDI/06-horizon-golden-image) · 🏠 [Retour au sommaire](/hyperviseur/VDI/00-sommaire)