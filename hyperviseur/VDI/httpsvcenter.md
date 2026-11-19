---
title: 07 - Sécuriser le vCenter et l'Active Directory
description: 
published: 1
date: 2026-11-19T21:29:25.159Z
tags: 
editor: markdown
dateCreated: 2026-11-11T13:48:57.656Z
---

# Remplacer le certificat du vCenter par un certificat LAB-ROOT-CA

> **Objectif** : le vCenter (`https://192.168.50.12` et `https://vcenter.lab.lan`) présente un certificat délivré par **LAB-ROOT-CA**. Le navigateur, Horizon et les scripts lui font alors confiance sans alerte.
>
> **Horizon n'en a pas besoin pour fonctionner** : il accepte aussi le certificat actuel signé par la VMCA. Cette procédure est un exercice à part, à faire **au calme**, le vCenter sauvegardé.

---

## 0. Comprendre avant de commencer

### 0.1 Le certificat concerné
Le vCenter a plusieurs certificats. On remplace uniquement le **certificat SSL de machine**, celui que voient le navigateur et Horizon sur le port 443. Les certificats internes et ceux de l'ESXi ne changent pas : ils restent signés par la VMCA.

### 0.2 Comment ça se passe
```
vCenter                      DC01 (LAB-ROOT-CA)                 vCenter
1. génère la clé privée  ──►  3. vérifie la demande        ──►  5. importe le certificat
   + la demande (CSR)         4. tu approuves, la CA signe        + la racine LAB-ROOT-CA
   (la clé ne sort jamais)                                         → les services redémarrent
```
La **clé privée reste dans le vCenter**. Seule la demande (CSR) circule.

### 0.3 La règle d'or de ton lab : le PNID
Le **PNID** de ton vCenter est **192.168.50.12**. Le vCenter **refuse** un certificat qui ne contient pas son PNID dans le champ *Autre nom de l'objet* (SAN). Le certificat contiendra donc :

| Champ | Valeur |
|---|---|
| Nom commun (CN) | `vcenter.lab.lan` (ce champ n'accepte pas d'IP) |
| SAN | `vcenter.lab.lan` **et** `192.168.50.12` |

### 0.4 Sécurité : pourquoi un modèle avec approbation
Le vCenter n'est pas un ordinateur du domaine. Le nom du certificat ne peut donc pas être tiré de l'AD, comme pour LAB-HorizonServer : il doit être **fourni dans la demande**.

Un modèle « nom fourni dans la demande », ouvert à n'importe qui, permet de demander un certificat au nom de `dc01.lab.lan` ou de l'administrateur : c'est l'attaque **ESC1**. On la bloque par deux protections :

1. Seuls les **Admins du domaine** (tier 0) peuvent faire la demande.
2. Chaque demande **attend ton approbation** dans la console de la CA. Rien n'est délivré automatiquement.

---

## 1. Prérequis

| Point | Vérification |
|---|---|
| Enregistrement DNS `vcenter.lab.lan` → 192.168.50.12 (A et PTR) | Console DNS sur DC01 (créé par `Creation-DNS-LAB.ps1`) |
| Le vCenter est en bonne santé | VAMI `https://192.168.50.12:5480` > **Résumé** : tout est vert |
| Ton PC fait confiance à LAB-ROOT-CA | Certificat racine importé dans *Autorités de certification racines de confiance* (procédure Horizon, partie 9.1) |
| Un créneau tranquille | Comptes 1 h. Les services du vCenter redémarrent pendant environ 15 min |

### 1.1 Le snapshot du vCenter (obligatoire)
C'est ton retour arrière si quelque chose casse.

1. Ouvre le **client hôte de l'ESXi** : `https://192.168.50.11/ui`, connecté en `root`. Le vCenter va être éteint, on ne peut donc pas passer par lui.
2. **Machines virtuelles** > clic droit sur **VCENTER** > **Système d'exploitation invité > Arrêter**. Attends qu'elle soit éteinte (5 à 10 min).
3. Clic droit sur **VCENTER** > **Snapshots > Prendre un snapshot**. Nom : `avant-certificat-LAB-ROOT-CA`. **Prendre un snapshot**.
4. Rallume la VM et attends que `https://192.168.50.12/ui` réponde de nouveau (10 à 15 min).

> Un snapshot pris **VM éteinte** est cohérent. Ne le garde pas plus de quelques jours : supprime-le dès que tout est validé (partie 7).

---

## 2. Créer le modèle LAB-vCenter (sur DC01, en adm-dom-afaye)

1. **Win + R** > `certtmpl.msc`.
2. Clic droit sur **Serveur Web** > **Dupliquer le modèle**.
3. Remplis les onglets :

| Onglet | Réglage |
|---|---|
| **Compatibilité** | Autorité : **Windows Server 2016** · Destinataire : **Windows 10 / Windows Server 2016** |
| **Général** | Nom complet : `LAB-vCenter` · Validité : **2 ans** · Renouvellement : **6 semaines** · ☐ **Publier le certificat dans Active Directory** |
| **Traitement de la demande** | Laisse par défaut. La clé est créée par le vCenter, ces réglages ne s'appliquent pas |
| **Chiffrement** | Taille minimale de la clé : **2048** |
| **Nom du sujet** | ● **Fournir dans la demande** · clique **OK** sur l'avertissement |
| **Conditions d'émission** | ☑ **Approbation du gestionnaire de certificats de l'autorité de certification** |
| **Extensions** | **Stratégies d'application** : uniquement **Authentification du serveur** (c'est le cas par défaut) |
| **Sécurité** | **Utilisateurs authentifiés** : Lecture seule · **Admins du domaine** et **Administrateurs de l'entreprise** : Lecture + **Inscrire** · personne d'autre n'a *Inscrire* |

4. **OK**.

> **Pourquoi 2 ans ?** Une CA privée n'est pas soumise à la limite de 398 jours des navigateurs. Deux ans, c'est un renouvellement de moins à gérer, pour un risque qui reste faible dans le lab. En production, on choisit souvent 1 an.

### 2.1 Publier le modèle
1. **Win + R** > `certsrv.msc` > déplie **LAB-ROOT-CA**.
2. Clic droit sur **Modèles de certificats** > **Nouveau > Modèle de certificat à délivrer** > **LAB-vCenter** > **OK**.

---

## 3. Générer la demande (CSR) dans le vCenter

Connecte-toi au vSphere Client en **adm-vmw-afaye@lab.lan** ou en `administrator@vsphere.local`.

1. **Menu (☰) > Administration > Certificats > Gestion des certificats**.
2. Carte **Certificat SSL de machine** (`__MACHINE_CERT`) > **Actions > Générer une demande de signature de certificat (CSR)**.
3. Remplis le formulaire :

| Champ | Valeur |
|---|---|
| Nom commun | `vcenter.lab.lan` |
| Organisation | `LAB` |
| Unité d'organisation | `vSphere` |
| Pays | France |
| État/province | `Ile-de-France` |
| Ville | `Paris` |
| Adresse e-mail | (vide) |
| Hôte | `192.168.50.12` **(ne change pas cette valeur : c'est le PNID)** |
| Autre nom de l'objet | `vcenter.lab.lan, 192.168.50.12` |
| Taille de la clé | **2048** |

4. **Suivant**. La demande s'affiche (`-----BEGIN CERTIFICATE REQUEST-----`). Clique sur **Télécharger** pour obtenir le fichier `vcenter.csr`, ou sur **Copier**.
5. **Terminer**.

> ⚠️ **Ne régénère pas de CSR** avant d'avoir importé le certificat (partie 5). Chaque nouvelle demande crée une **nouvelle clé privée**, et le certificat obtenu avec l'ancienne demande ne pourrait plus être importé.

---

## 4. Faire signer la demande par LAB-ROOT-CA (sur DC01)

L'inscription web n'est pas installée sur la CA, et c'est voulu (protection contre ESC8). On passe donc par `certreq`, qui ouvre des fenêtres graphiques.

### 4.1 Copier la demande sur DC01
1. Crée le dossier `C:\PKI\vcenter`.
2. Mets-y `vcenter.csr`. Tu peux aussi ouvrir le **Bloc-notes**, coller le texte de la demande (lignes BEGIN et END comprises) et l'enregistrer sous `C:\PKI\vcenter\vcenter.csr`, avec le type **Tous les fichiers**.

### 4.2 Vérifier la demande avant de la soumettre
PowerShell sur DC01 :
```powershell
certutil -dump C:\PKI\vcenter\vcenter.csr
```
Dans la section *Autre nom de l'objet*, tu dois voir **Nom DNS=vcenter.lab.lan** et **Adresse IP=192.168.50.12**. Si l'IP manque, refais la partie 3.

### 4.3 Soumettre la demande
```powershell
certreq -submit -attrib "CertificateTemplate:LAB-vCenter" C:\PKI\vcenter\vcenter.csr
```
1. Une fenêtre demande l'autorité : choisis **LAB-ROOT-CA** > **OK**.
2. Réponse attendue : **« Certificat en attente : Pris en charge par la stratégie »** avec un **RequestId** (par exemple 12). Note ce numéro.

### 4.4 Approuver la demande
1. `certsrv.msc` > **LAB-ROOT-CA > Demandes en attente**.
2. Double-clique sur la demande portant ton numéro. Vérifie le **nom (vcenter.lab.lan)** et le **demandeur (LAB\adm-dom-afaye)**.
3. Clic droit > **Toutes les tâches > Délivrer**.

> 🔒 C'est l'**approbation** : si une demande inconnue apparaît ici un jour, tu la **refuses**.

### 4.5 Récupérer le certificat
1. **Certificats délivrés** > double-clique sur le certificat **vcenter.lab.lan**.
2. Onglet **Détails** > **Copier dans un fichier…** > Suivant > **X.509 encodé en base 64 (.cer)** > `C:\PKI\vcenter\vcenter.cer` > **Terminer**.
3. Onglet **Détails**, contrôle **Autre nom de l'objet** : **vcenter.lab.lan** et **192.168.50.12**.

### 4.6 Exporter la racine LAB-ROOT-CA
Si tu ne l'as pas déjà :

1. `certsrv.msc` > clic droit sur **LAB-ROOT-CA** > **Propriétés** > **Afficher le certificat**.
2. **Détails > Copier dans un fichier** > **Base 64** > `C:\PKI\vcenter\LAB-ROOT-CA.cer`.

Copie **vcenter.cer** et **LAB-ROOT-CA.cer** sur le PC depuis lequel tu ouvres le vSphere Client.

---

## 5. Importer le certificat dans le vCenter

### 5.1 Ajouter d'abord la racine aux certificats approuvés
1. **Administration > Certificats > Gestion des certificats** > section **Certificats racines approuvés** > **Ajouter**.
2. Parcourir > **LAB-ROOT-CA.cer** > ☐ *Démarrer la propagation des racines vers tous les hôtes ESXi* (pas nécessaire) > **Ajouter**.
3. **Contrôle** : une nouvelle carte **LAB-ROOT-CA** apparaît.

### 5.2 Remplacer le certificat SSL de machine
1. Carte **Certificat SSL de machine** > **Actions > Importer et remplacer le certificat**.
2. Choisis **Remplacer par un certificat CA externe lorsque la CSR est générée à partir de vCenter Server (clé privée intégrée)** > **Suivant**.
3. **Certificat SSL de machine** : **Parcourir** > `vcenter.cer`.
4. **Chaîne de certificats** : **Parcourir** > `LAB-ROOT-CA.cer`.
5. **Remplacer**.

Le vCenter redémarre ses services et la session se coupe. **Attends 15 minutes** sans rien toucher.

> Si un message refuse le certificat **avant** le redémarrage, rien n'a été modifié : voir le dépannage (partie 8).

---

## 6. Contrôles

| # | Contrôle | Résultat attendu |
|---|---|---|
| 1 | VAMI `https://192.168.50.12:5480` > **Services** | Tous les services en démarrage automatique sont **démarrés** |
| 2 | Ferme le navigateur, rouvre `https://vcenter.lab.lan/ui`, puis `https://192.168.50.12/ui` | **Cadenas sans alerte** dans les deux cas · certificat délivré par **LAB-ROOT-CA** |
| 3 | Connexion en **adm-vmw-afaye@lab.lan** | OK (la source d'identité LDAPS n'est pas touchée) |
| 4 | **Gestion des certificats** > carte Certificat SSL de machine | Émetteur `LAB-ROOT-CA`, date de fin dans 2 ans |
| 5 | **Hôtes et clusters** | 192.168.50.11 est **connecté** |
| 6 | Si Horizon a déjà le vCenter | Voir 6.1 |

### 6.1 Si le vCenter était déjà ajouté dans Horizon
Horizon avait mémorisé l'**ancien** certificat et va afficher le vCenter **en rouge**.

1. Console Horizon > **Paramètres > Serveurs > vCenter Server** > sélectionne 192.168.50.12 > **Modifier**.
2. **OK** > la fenêtre *Certificat non valide détecté* s'affiche > **Afficher le certificat** > vérifie l'émetteur **LAB-ROOT-CA** > **Accepter**.

---

## 7. Après la validation
- Laisse tourner un ou deux jours, puis **supprime le snapshot** `avant-certificat-LAB-ROOT-CA` : client ESXi ou vCenter > VM VCENTER > **Snapshots > Gérer** > Supprimer.
- Sur DC01, supprime `C:\PKI\vcenter\vcenter.csr`. Il ne contient pas de secret, mais il ne sert plus.
- **Note la date d'expiration** dans ton agenda : environ 2 mois avant, refais les parties 3 à 6. Un nouveau CSR crée une nouvelle clé.

---

## 8. Dépannage

| Symptôme | Cause | Solution |
|---|---|---|
| « Entrez un nom de domaine complet valide » dans le CSR | Une IP dans **Nom commun** | CN = `vcenter.lab.lan`, l'IP va uniquement dans *Autre nom de l'objet* |
| `certreq` : « modèle introuvable » ou « refusé » | Modèle non publié, ou droit *Inscrire* manquant | Parties 2 et 2.1. Lance PowerShell avec `adm-dom-afaye` |
| La demande est délivrée directement, sans attente | Approbation non cochée | Onglet *Conditions d'émission* du modèle (partie 2) |
| L'import refuse : PNID absent ou SAN | L'IP 192.168.50.12 manque dans le SAN | Contrôle 4.2 / 4.5, refais la CSR avec l'IP |
| L'import refuse : chaîne incomplète | La racine n'est pas fournie ou pas au format base 64 | Partie 4.6 (base 64) et 5.2, étape 4 |
| L'import refuse : la clé privée ne correspond pas | Une CSR a été régénérée entre-temps | Refais les parties 3 à 5 avec la **dernière** CSR |
| Le navigateur affiche encore une alerte | Le PC ne fait pas confiance à LAB-ROOT-CA, ou le cache | Importe la racine sur le PC, ferme complètement le navigateur |
| vSphere Client inaccessible après 30 min | Services bloqués | VAMI > **Services**. Démarre ceux qui sont arrêtés. Sinon, retour arrière (8.1) |

### 8.1 Retour arrière
Si le vSphere Client ne revient pas :

1. Client hôte de l'ESXi (`https://192.168.50.11/ui`) > VM **VCENTER** > **Snapshots > Gérer les snapshots**.
2. Sélectionne `avant-certificat-LAB-ROOT-CA` > **Restaurer**. La VM revient éteinte, dans l'état d'avant.
3. Rallume-la et attends 15 min.

Autre solution sans snapshot, en SSH sur le vCenter : `/usr/lib/vmware-vmca/bin/certificate-manager`, **option 8** (« Reset all Certificates »). Elle remet des certificats signés par la VMCA. Note qu'elle remplace **tous** les certificats.

---

## 9. Récapitulatif sécurité

| Mesure | Pourquoi |
|---|---|
| La clé privée générée **dans le vCenter** | Elle ne circule jamais |
| Modèle dédié **LAB-vCenter**, inscription réservée au tier 0 | Personne d'autre ne peut demander ce type de certificat |
| **Approbation du gestionnaire** obligatoire | Bloque l'abus du nom fourni dans la demande (ESC1) |
| Pas d'inscription web | Pas d'exposition à ESC8 |
| SAN limité à `vcenter.lab.lan` et `192.168.50.12` | Le certificat ne vaut que pour le vCenter |
| Snapshot **à froid** avant, supprimé après | Un retour arrière possible, sans garder un snapshot qui grossit |
