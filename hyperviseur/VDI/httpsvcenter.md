---
title: 07 - Sécuriser le vCenter et l'Active Directory
description: 
published: 1
date: 2026-11-20T10:55:39.170Z
tags: 
editor: markdown
dateCreated: 2026-11-11T13:48:57.656Z
---

# 07 - Sécuriser le vCenter et l'Active Directory

> **Objectif de la page :**
> 1. donner au vCenter un **certificat émis par LAB-ROOT-CA** : plus d'alerte dans le navigateur, et Horizon lui fera confiance sans exception ;
> 2. **durcir** le vCenter, l'ESXi et l'AD, maintenant que tout passe en LDAPS.
>
> 📸 *Captures de la partie 1 : modèle, publication et racine de confiance. Le reste est à illustrer au moment où tu le réalises.*

---

## 1. Un certificat LAB-ROOT-CA pour le vCenter

### 1.1 Pourquoi

Aujourd'hui, l'interface web du vCenter présente un certificat signé par la **VMCA**, l'autorité interne du vCenter. Aucune machine ne la connaît :

- le navigateur affiche *« Non sécurisé »* ;
- Horizon affichera *« Certificat non valide détecté »* à l'ajout du vCenter (page 10) ;
- on prend l'habitude de cliquer sur « Accepter », ce qui est exactement ce qu'un attaquant espère.

Les machines du domaine font déjà confiance à LAB-ROOT-CA. Si elle signe le certificat du vCenter, l'alerte disparaît **pour de bonnes raisons**.

### 1.2 Le choix : le mode hybride

![Donner au vCenter un certificat LAB-ROOT-CA](/vdi/07-certificat-vcenter.svg)

| Mode | Principe | Choix |
|---|---|---|
| VMCA seule (défaut) | La VMCA signe tout | Situation actuelle |
| **Hybride** | On remplace **seulement le certificat SSL de la machine** (celui du site web) par un certificat LAB-ROOT-CA. La VMCA continue de signer les certificats des ESXi et des services internes | ✅ **Recommandé par VMware, retenu ici** |
| VMCA subordonnée | La VMCA devient une sous-CA de LAB-ROOT-CA | ❌ Lourd, et donne au vCenter le pouvoir d'émettre des certificats reconnus par tout le domaine |


### 1.3 Ce que doit contenir le certificat

Pour le vCenter, son « FQDN » est son **PNID**, ici l'IP `192.168.50.12` (page 03). À l'import, il vérifie que les **noms DNS** du certificat contiennent ce PNID, **même si c'est une IP**. Une entrée « adresse IP » ne suffit pas.

| Champ | Valeur | Pourquoi |
|---|---|---|
| Nom commun (CN) | `192.168.50.12` | Le PNID |
| Nom alternatif : **nom DNS** | **`192.168.50.12`** | **Obligatoire** : c'est ce que contrôle le vCenter (*« DNS Name must contain machine FQDN »*) |
| Nom alternatif : **adresse IP** | **`192.168.50.12`** | Pour les navigateurs et Horizon, qui comparent une adresse IP aux entrées « adresse IP » |
| Nom alternatif : nom DNS | `vcenter.lab.lan` | Facultatif : pour ouvrir aussi le vCenter par son nom |

**Comment le vérifier sur ton vCenter** (en SSH, `shell`) :

```
/usr/lib/vmware-vmafd/bin/vmafd-cli get-pnid --server-name localhost
```

La commande affiche le PNID : `192.168.50.12`. Le nom d'hôte Linux (`hostname -f`) est `localhost`, mais ce n'est **pas** lui que le vCenter contrôle.

> Le certificat d'origine de la VMCA ne contient d'ailleurs qu'une entrée `IP Address:192.168.50.12` : la VMCA n'est pas soumise au contrôle de l'import, un certificat externe si.

### 1.4 Le modèle LAB-vCenter

Le nom du certificat vient **de la demande** générée par le vCenter (une **CSR**), puisque le vCenter n'est pas dans l'AD. C'est exactement la situation de l'attaque **ESC1**. On la neutralise par trois verrous :

| Verrou | Effet |
|---|---|
| **Approbation obligatoire** par le gestionnaire de la CA | Aucune demande n'est délivrée sans que tu l'aies vue et acceptée |
| Droit *Inscrire* réservé aux **Admins du domaine** | Un utilisateur ordinaire ne peut même pas soumettre de demande |
| Usage limité à **Authentification du serveur** | Le certificat ne permet pas d'ouvrir une session dans le domaine |

**Créer le modèle.** Sur DC01, `certtmpl.msc` > clic droit sur **Serveur Web** > **Dupliquer le modèle** :

| Onglet | Réglage |
|---|---|
| **Compatibilité** | Windows Server 2016 / Windows 10 – Windows Server 2016 |
| **Général** | Nom complet et nom du modèle : `LAB-vCenter` · validité **1 an** · renouvellement 6 semaines · ☐ Publier dans AD |
| **Traitement de la demande** | ☐ Autoriser l'exportation de la clé privée (la clé est générée **sur le vCenter** avec openssl : ce réglage ne la concerne pas) |
| **Chiffrement** | **Fournisseur de stockage de clés**, RSA, **2048**, SHA256 · ● *Les demandes peuvent utiliser un fournisseur disponible…* (la clé vient du vCenter : le fournisseur Windows ne sert pas) |
| **Nom du sujet** | ● **Fournir dans la demande** (l'avertissement de sécurité est normal : les deux verrous suivants y répondent) |
| **Conditions d'émission** | ☑ **Approbation du gestionnaire de certificat de l'autorité de certification** |
| **Extensions > Stratégies d'application** | Seulement **Authentification du serveur** |
| **Sécurité** | **Admins du domaine** : Lecture + Inscrire · **Utilisateurs authentifiés** : Lecture seulement · aucun autre droit *Inscrire* |

**OK**.

Les onglets, tels qu'ils sont réglés dans le lab :

**Général :** `LAB-vCenter` en nom complet et en nom du modèle, 1 an, 6 semaines, pas de publication dans l'AD.

![LAB-vCenter - onglet Général](/vdi/07-securiser-vcenter-ad/01-modele-general.png)

**Traitement de la demande :** ☐ *Autoriser l'exportation de la clé privée*.

![LAB-vCenter - onglet Traitement de la demande](/vdi/07-securiser-vcenter-ad/02-modele-traitement-demande.png)

**Chiffrement :** fournisseur de stockage de clés, RSA 2048, SHA256.

![LAB-vCenter - onglet Chiffrement](/vdi/07-securiser-vcenter-ad/03-modele-chiffrement.png)

**Nom du sujet :** ● **Fournir dans la demande**.

![LAB-vCenter - onglet Nom du sujet](/vdi/07-securiser-vcenter-ad/04-modele-nom-du-sujet.png)

**Conditions d'émission :** ☑ **Approbation du gestionnaire de certificat de l'autorité de certification**. C'est le verrou principal contre ESC1.

![LAB-vCenter - onglet Conditions d'émission](/vdi/07-securiser-vcenter-ad/05-modele-conditions-emission.png)

**Extensions :** *Stratégies d'application* = **Authentification du serveur** seulement.

![LAB-vCenter - onglet Extensions](/vdi/07-securiser-vcenter-ad/06-modele-extensions.png)

**Sécurité :** les *Utilisateurs authentifiés* n'ont que la **Lecture** : ils ne peuvent pas demander ce certificat. Seuls *Admins du domaine* et *Administrateurs de l'entreprise* ont *Inscrire*.

![LAB-vCenter - onglet Sécurité](/vdi/07-securiser-vcenter-ad/07-modele-securite.png)

> **Onglet Compatibilité :** vérifie qu'il est bien sur **Windows Server 2016** / **Windows 10 – Windows Server 2016**, et pas sur les valeurs par défaut (Windows Server 2003 / Windows XP). Le fournisseur de stockage de clés n'est proposé qu'avec une compatibilité récente.

**Publier le modèle.** `certsrv.msc` > **LAB-ROOT-CA** > clic droit sur **Modèles de certificats** > **Nouveau > Modèle de certificat à délivrer** > **LAB-vCenter** > **OK**.

![certsrv - activer le modèle LAB-vCenter](/vdi/07-securiser-vcenter-ad/08-certsrv-activer-modele.png)

La CA délivre maintenant trois modèles, un par usage :

![certsrv - les trois modèles publiés](/vdi/07-securiser-vcenter-ad/09-certsrv-modeles-publies.png)

> **Ménage :** si un essai précédent a laissé un autre modèle (par exemple `LAB-vCenterServer`), clic droit > **Supprimer** dans `certsrv.msc` ne fait que le **dépublier**. Pour le supprimer vraiment, fais aussi clic droit > **Supprimer** dans `certtmpl.msc`.

### 1.5 La procédure, étape par étape

> Tous les fichiers sont rangés sur DC01 dans **`C:\PKI\vcenter\`**. Toutes les commandes se lancent sur DC01, en **adm-dom-afaye**, dans un PowerShell ouvert avec **clic droit > Exécuter en tant qu'administrateur**.

#### Étape 1 : vérifier les prérequis

1. Le modèle est publié :
   ```
   certutil -CATemplates
   ```
   `LAB-vCenter` doit apparaître dans la liste, avec `LAB-DomainController` et `LAB-HorizonServer`.
2. Dans `certtmpl.msc` > **LAB-vCenter**, contrôle les trois onglets qui comptent :

| Onglet | Attendu |
|---|---|
| Nom du sujet | ● **Fournir dans la demande** |
| Conditions d'émission | ☑ **Approbation du gestionnaire de certificat** |
| Sécurité | **Admins du domaine** : ☑ Lecture, ☑ Inscrire |

#### Étape 2 : préparer le vCenter

1. 📸 **Snapshot de la VM VCENTER**, sans la mémoire, pour pouvoir revenir en arrière.
2. **Exporte la racine LAB-ROOT-CA** : `certsrv.msc` > clic droit sur **LAB-ROOT-CA** > **Propriétés** > **Afficher le certificat** > onglet **Détails** > **Copier dans un fichier** > **X.509 encodé en base 64 (.CER)** > `C:\PKI\vcenter\lab-root-ca.cer`. Copie-le sur le PC d'où tu ouvres le vSphere Client.
3. vSphere Client > **Menu ☰ > Administration > Certificats > Gestion des certificats** > onglet **Racine de confiance** :
   - si **LAB-ROOT-CA** est déjà dans la liste, passe à l'étape 3 ;
   - sinon, **Ajouter un certificat racine approuvé** > `lab-root-ca.cer` > **Ajouter**.

![vCenter - LAB-ROOT-CA dans les racines de confiance](/vdi/07-securiser-vcenter-ad/10-vcenter-racine-confiance.png)

> Sans la racine dans le vCenter, il refuserait le futur certificat, faute de pouvoir vérifier sa chaîne. Les deux lignes **CA** sont la VMCA elle-même : on n'y touche pas.

#### Étape 3 : générer la CSR avec openssl, sur le vCenter

> **Pourquoi pas le formulaire du vSphere Client ?** Son bouton *Générer une demande de signature de certificat (CSR)* exige un nom de domaine complet (*« Entrez un nom de domaine complet valide »*) et ne permet pas de mettre l'IP `192.168.50.12` en **nom DNS**, alors que l'import l'exige. On génère donc la clé et la CSR avec `openssl`, directement sur l'appliance, puis on importe le certificat **avec sa clé privée** (étape 8).

**3.1 Ouvrir un accès SSH temporaire**

1. VAMI `https://192.168.50.12:5480` (compte `root`) > **Accès** > **Modifier** > ☑ **Activer la connexion SSH** > **OK**.
2. Sur ton PC, ouvre un PowerShell :
   ```
   ssh root@192.168.50.12
   ```
   Accepte l'empreinte (`yes`), saisis le mot de passe root. À l'invite `Command>`, tape :
   ```
   shell
   ```

**3.2 Écrire le fichier de configuration**

Colle ce bloc d'un seul coup dans la session SSH (clic droit pour coller) :

```
mkdir -p /root/csr && cd /root/csr
cat > vcenter.cfg <<'EOF'
[ req ]
default_bits       = 2048
prompt             = no
distinguished_name = dn
req_extensions     = ext

[ dn ]
CN = 192.168.50.12
O  = LAB
OU = vSphere
C  = FR
ST = Ile-de-France
L  = Paris

[ ext ]
keyUsage         = critical, digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth
subjectAltName   = DNS:192.168.50.12, IP:192.168.50.12, DNS:vcenter.lab.lan
EOF
```

**3.3 Générer la clé et la CSR**

```
openssl req -new -nodes -newkey rsa:2048 -keyout vcenter.key -out vcenter.csr -config vcenter.cfg
chmod 600 vcenter.key
```

**3.4 Vérifier sur place**

```
openssl req -in vcenter.csr -noout -text | grep -A1 "Subject Alternative Name"
```

La réponse doit être `DNS:192.168.50.12, IP Address:192.168.50.12, DNS:vcenter.lab.lan`.

**3.5 Copier la CSR sur DC01**

```
cat vcenter.csr
```

Sélectionne tout le bloc, de `-----BEGIN CERTIFICATE REQUEST-----` à `-----END CERTIFICATE REQUEST-----` inclus, et copie-le. Sur DC01, ouvre le **Bloc-notes**, colle, puis **Enregistrer sous** `C:\PKI\vcenter\vcenter.csr` avec **Type : Tous les fichiers** (en remplaçant l'ancien).

> **Garde la session SSH ouverte** : on reviendra chercher la clé privée à l'étape 8.

#### Étape 4 : vérifier la CSR

```
certutil -dump C:\PKI\vcenter\vcenter.csr
```

Dans la section **Autre nom de l'objet**, les trois lignes doivent apparaître :

```
Nom DNS=192.168.50.12
Adresse IP=192.168.50.12
Nom DNS=vcenter.lab.lan
```

La première ligne est la plus importante : sans elle, le vCenter refusera le certificat.

S'il en manque une, **refais l'étape 3**. Une CSR incomplète serait signée sans problème, mais le vCenter refuserait le certificat à l'import.

#### Étape 5 : soumettre la demande

```
certreq -submit -attrib "CertificateTemplate:LAB-vCenter" C:\PKI\vcenter\vcenter.csr
```

Choisis **LAB-ROOT-CA** si on te le demande. Réponse attendue : *« Le certificat demandé est en attente »*, avec un **identifiant de requête** (par exemple `5`). **Note-le.**

> 🔒 Grâce à l'approbation obligatoire, rien n'est encore délivré : la demande attend ta décision.

#### Étape 6 : approuver la demande

1. `certsrv.msc` > **LAB-ROOT-CA** > **Demandes en attente**.
2. Double-clic sur ta demande (même identifiant). Vérifie que le nom et les noms alternatifs sont bien `192.168.50.12` (en nom DNS **et** en adresse IP) et `vcenter.lab.lan`.
3. Clic droit sur la demande > **Toutes les tâches > Délivrer**. Elle passe dans **Certificats délivrés**.

> 🔒 **C'est ici que joue l'approbation :** une demande au nom de `dc01.lab.lan` ou d'un administrateur ne passerait jamais cette étape.

#### Étape 7 : récupérer le certificat

```
certreq -retrieve 5 C:\PKI\vcenter\vcenter.cer
```

Remplace `5` par ton identifiant. Choisis **LAB-ROOT-CA** si on te le demande. Copie `vcenter.cer` sur le PC d'où tu ouvres le vSphere Client, à côté de `lab-root-ca.cer`.

> Le fichier obtenu est au format Base 64 (il commence par `-----BEGIN CERTIFICATE-----`), celui qu'attend le vCenter.

#### Étape 8 : importer et remplacer

1. **Récupère la clé privée** sur ton PC. Dans la session SSH :
   ```
   cat /root/csr/vcenter.key
   ```
   Copie tout le bloc, de `-----BEGIN PRIVATE KEY-----` à `-----END PRIVATE KEY-----` inclus. Sur le PC, colle-le dans le Bloc-notes et enregistre-le sous `vcenter.key` (**Type : Tous les fichiers**), à côté de `vcenter.cer` et `lab-root-ca.cer`.
   > 🔒 Ce fichier est la clé du vCenter : ne le copie nulle part ailleurs, et supprime-le dès l'étape 10.
2. **Gestion des certificats** > onglet **SSL de machine** > **Importer et remplacer un certificat**.
3. Choisis **Remplacer par un certificat externe (avec clé privée)**.
4. Renseigne :

| Champ | Fichier |
|---|---|
| Certificat SSL de la machine | `vcenter.cer` |
| Clé privée | `vcenter.key` |
| Chaîne de certificats | `lab-root-ca.cer` |

5. **Remplacer**. Les services du vCenter redémarrent : compte **10 à 15 minutes** en imbriqué. La page devient indisponible pendant ce temps, c'est normal.

#### Étape 9 : contrôler

1. Ferme le navigateur et rouvre `https://192.168.50.12/ui` depuis la VM Horizon ou une machine du domaine : **cadenas**, certificat **délivré par LAB-ROOT-CA**, aucune alerte.
2. **Gestion des certificats > SSL de machine** : *Émetteur* = **LAB-ROOT-CA**, état **Valide**, fin de validité dans un an.
3. L'hôte **192.168.50.11** est toujours **Connecté** dans l'inventaire.
4. Depuis ton PC (hors domaine) : l'alerte disparaît une fois LAB-ROOT-CA importé dans ses **Autorités de certification racines de confiance** (page 10, partie 7).
5. Tout fonctionne ? **Supprime le snapshot** du vCenter.

#### Étape 10 : faire le ménage

Les demandes ratées et le certificat délivré pour l'ancienne CSR ne servent plus.

1. **La clé privée :**
   - sur le PC, supprime `vcenter.key` avec **Maj + Suppr** (sans passer par la corbeille) ;
   - dans la session SSH : `rm -rf /root/csr`, puis `exit` deux fois.
2. **Le SSH :** VAMI > **Accès** > **Modifier** > ☐ **Activer la connexion SSH** > **OK**.
3. **Les anciens certificats :** `certsrv.msc` > **Certificats délivrés** : clic droit sur chaque **ancien** certificat du vCenter (ceux sans `DNS:192.168.50.12`, émis lors des essais) > **Toutes les tâches > Révoquer le certificat** > motif **Remplacé**.
4. Les demandes de **Demandes ayant échoué** peuvent rester : elles servent d'historique.
5. Note la date d'expiration du nouveau certificat à côté de celles de DC01 et d'Horizon.

> 🔁 **Renouvellement :** ce certificat expire dans **1 an**. Refais les étapes 2 à 10 un mois avant (en gardant le fichier `vcenter.cfg` de l'étape 3.2 dans tes notes).

> **Retour arrière :** si le vCenter ne revient pas après l'étape 8, rétablis le snapshot. Le compte `administrator@vsphere.local` reste disponible tout du long.

### 1.6 Les pièges rencontrés

| Message | Cause | Solution |
|---|---|---|
| *« Les autorisations sur le modèle de certificat n'autorisent pas l'utilisateur actuel à s'inscrire… 0x80094012 (CERTSRV_E_TEMPLATE_DENIED) »* | Console **non élevée** : l'UAC désactive le groupe *Admins du domaine* dans le jeton, donc le droit *Inscrire* ne s'applique pas | **Exécuter en tant qu'administrateur**, puis soumettre à nouveau la même CSR. Sinon, vérifier l'onglet **Sécurité** du modèle et redémarrer le service `certsvc` |
| *« Le modèle de certificat demandé n'est pas pris en charge par cette autorité de certification… 0x80094800 (CERTSRV_E_UNSUPPORTED_CERT_TYPE) »* | Modèle **non publié** sur LAB-ROOT-CA, ou mauvais nom dans `certreq` | Publier le modèle (1.4), vérifier avec `certutil -CATemplates`, utiliser le **nom du modèle** (onglet Général) |
| *« Replace cert Failed: … DNS in Subject Alternative Name is not correct. DNS Name must contain machine FQDN. »* | Le « FQDN » du vCenter est son **PNID**, l'IP `192.168.50.12`, et il doit figurer **en nom DNS** dans le certificat. Ni `vcenter.lab.lan`, ni `localhost` (le nom d'hôte Linux), ni une simple entrée « adresse IP » ne suffisent | Nouvelle CSR avec `DNS:192.168.50.12` (étape 3), contrôle avec `certutil -dump` (étape 4). L'ancien certificat reste en place et valide en attendant |
| *« Entrez un nom de domaine complet valide »* dans le formulaire *Générer une CSR* du vSphere Client | Le formulaire n'accepte que des FQDN : impossible d'y mettre l'IP en nom DNS | Générer la CSR avec `openssl` sur l'appliance (étape 3), puis importer **avec la clé privée** (étape 8) |
| L'import refuse la clé privée | Format de clé non reconnu par ta version | Dans la session SSH : `openssl rsa -in vcenter.key -out vcenter-rsa.key -traditional`, puis importer `vcenter-rsa.key` (il commence par `-----BEGIN RSA PRIVATE KEY-----`) |

---

## 2. Durcir le vCenter

### 2.1 Le SSO

**Menu ☰ > Administration > Single Sign-On > Configuration** > onglet **Comptes locaux** :

| Stratégie | Réglage conseillé |
|---|---|
| Mot de passe | 15 caractères minimum, durée de vie 90 jours (les comptes AD suivent la stratégie de l'AD) |
| Verrouillage | **5** tentatives, déverrouillage après **15 minutes** |
| Jeton | Laisse les valeurs par défaut |

> `administrator@vsphere.local` devient un **compte de secours** : mot de passe long dans le coffre, utilisé seulement si l'AD est indisponible ou pour les certificats.

### 2.2 L'appliance (VAMI)

`https://192.168.50.12:5480`, compte `root` :

| Menu | Réglage |
|---|---|
| **Accès** | ☐ Connexion SSH · ☑ Connexion DCUI · ☐ Shell de console. Réactive SSH seulement le temps d'une opération |
| **Heure** | Mode **NTP**, serveur `192.168.50.10` |
| **Administration** | Expiration du mot de passe root activée, avec un e-mail ou un rappel |
| **Sauvegarde** | Planifie une **sauvegarde fichier** quotidienne vers un partage SMB (sur le PC par exemple), avec chiffrement par mot de passe |
| **Syslog** | Optionnel : envoyer les journaux vers un serveur de logs |

> ⚠️ **Jamais de modification du nom d'hôte** dans *Réseau* (page 03).

### 2.3 Le vSphere Client

- **Administration > Déploiement > Configuration du client** : délai d'expiration de session **15 à 30 minutes**.
- **Administration > Déploiement > Programme d'amélioration du produit** : **quitter** le programme.
- Les droits restent donnés **à des groupes AD**, jamais à des comptes nominatifs (page 06).

### 2.4 L'ESXi

| Réglage | Où | Valeur |
|---|---|---|
| **Mode de verrouillage** | **192.168.50.11 > Configurer > Système > Profil de sécurité > Mode de verrouillage > Modifier** | **Normal** : l'ESXi ne s'administre plus que par le vCenter. La DCUI reste accessible en secours |
| Services | **Configurer > Système > Services** | SSH et ESXi Shell **arrêtés**, démarrage **manuel** |
| Mot de passe root | Coffre | Long et unique |

> En mode verrouillage normal, si le vCenter est perdu, on désactive le verrouillage depuis la **DCUI** de l'ESXi (console de la VM dans Workstation). Teste-le une fois pour savoir le faire.

---

## 3. Durcir l'Active Directory

Tous ces réglages se font sur DC01, en **adm-dom-afaye**.

### 3.1 Signature LDAP et liaison de canal

Maintenant que le vCenter est en LDAPS, on peut **interdire le LDAP non signé**.

**Étape 1 : observer.** Vérifie que plus personne ne fait de LDAP en clair.

1. **Win + R** > `regedit` > `HKLM\SYSTEM\CurrentControlSet\Services\NTDS\Diagnostics` > **16 LDAP Interface Events** = `2`.
2. Utilise le lab normalement pendant quelques heures.
3. **Observateur d'événements > Journaux des applications et des services > Directory Service** > **Filtrer** sur l'ID **2889**. Chaque événement nomme une IP et un compte qui font du LDAP non signé. **La liste doit être vide.**
4. Remets la valeur à `0`.

**Étape 2 : exiger.** **Gestion des stratégies de groupe** > clic droit sur l'OU **Domain Controllers** > **Créer un objet GPO… et le lier ici** > `GPO-DC-Hardening` > **Modifier** > **Configuration ordinateur > Stratégies > Paramètres Windows > Paramètres de sécurité > Stratégies locales > Options de sécurité** :

| Paramètre | Valeur |
|---|---|
| Contrôleur de domaine : conditions requises pour la signature de serveur LDAP | **Exiger la signature** |
| Contrôleur de domaine : conditions requises pour le jeton de liaison de canal LDAP du serveur | **Lorsque pris en charge** |
| Sécurité réseau : niveau d'authentification LAN Manager | **Envoyer uniquement des réponses NTLM version 2. Refuser LM et NTLM** |
| Serveur réseau Microsoft : communications signées numériquement (toujours) | **Activé** |

`gpupdate /force`, puis reconnecte-toi au vCenter en `adm-vmw-afaye` pour vérifier.

> **Liaison de canal :** empêche de relayer une authentification NTLM volée vers LDAPS. « Lorsque pris en charge » est sans risque. Tu peux tester « Toujours » ensuite, et revenir en arrière si le vCenter ne se connecte plus.

### 3.2 TLS 1.2 minimum

SSL 2.0 et 3.0, TLS 1.0 et 1.1 sont obsolètes. Le plus simple en interface graphique est l'outil gratuit **IIS Crypto** (Nartac) : bouton **Best Practices**, décoche TLS 1.0 et 1.1 dans *Server Protocols* et *Client Protocols*, **Apply**, redémarre DC01.

Après le redémarrage, refais le test `ldp.exe` (page 06, partie 3) et une connexion au vCenter.

### 3.3 Protéger les comptes d'administration

| Mesure | Comment |
|---|---|
| **Utilisateurs protégés** | Ajoute `adm-dom-afaye` au groupe **Protected Users** (`lab.lan > Users`) : plus de NTLM, pas de mise en cache du mot de passe, tickets Kerberos de 4 heures. Teste ensuite tes connexions. Pour `adm-vmw` et `adm-hzn`, teste un par un : un service qui exige NTLM ne les accepterait plus |
| **Tiering par GPO** | GPO liée à `LAB > Serveurs` : **Attribution des droits utilisateur > Interdire l'ouverture d'une session locale** et **… par les services Bureau à distance** pour **Admins du domaine**. Le tier 0 ne se connecte plus aux serveurs |
| **Comptes de service** | Groupe `GG-SVC-DenyInteractive` (dans `LAB > Groupes`) avec `svc-vcenter-ldap`, `svc-horizon-vc`, `svc-horizon-ic`. GPO liée au domaine : **Interdire l'ouverture d'une session locale**, **… par les services Bureau à distance**, **… en tant que tâche**, **… en tant que service**. ⚠️ **Pas** *Interdire l'accès à cet ordinateur à partir du réseau* : ça bloquerait le bind LDAP |
| **Mots de passe longs** | Donne aux comptes de service un mot de passe de **32 caractères** généré par ton coffre, puis mets-le à jour dans la source d'identité du vCenter (page 06) et dans Horizon (page 10) |

### 3.4 Windows LAPS

Le mot de passe de l'**administrateur local** des serveurs membres doit être unique et changer tout seul. **Windows LAPS** est intégré à Windows Server 2022 et 2025 (à jour).

1. Une seule fois, dans PowerShell administrateur sur DC01 :
   ```
   Update-LapsADSchema
   Set-LapsADComputerSelfPermission -Identity "OU=Serveurs,OU=LAB,DC=lab,DC=lan"
   ```
2. GPO liée à `LAB > Serveurs` > **Configuration ordinateur > Stratégies > Modèles d'administration > Système > LAPS** :
   - **Configurer le répertoire de sauvegarde des mots de passe** : **Activé**, *Active Directory* ;
   - **Paramètres de mot de passe** : 20 caractères, 30 jours.
3. Sur HORIZON, `gpupdate /force`. Le mot de passe apparaît dans **Utilisateurs et ordinateurs AD** > HORIZON > **Propriétés** > onglet **LAPS**.

### 3.5 Petits réglages qui comptent

| Réglage | Où | Pourquoi |
|---|---|---|
| Service **Spouleur d'impression** arrêté et désactivé sur DC01 | `services.msc` | Évite les attaques de type PrintNightmare et la coercition d'authentification |
| Audit avancé | `GPO-DC-Hardening` > **Configuration avancée de la stratégie d'audit** : Ouverture de session de compte, Gestion des comptes, Accès DS, Changement de stratégie | Tracer qui fait quoi sur le domaine |
| Sauvegarde de la CA à jour | Page 05, partie 3.4 | Après tout nouveau modèle |
| Corbeille AD | **Centre d'administration Active Directory** > clic droit sur `lab` > **Activer la Corbeille** | Restaurer un compte supprimé par erreur |

---

## 4. Checklist

| ✓ | Vérification |
|---|---|
| ☐ | Modèle `LAB-vCenter` avec approbation, publié |
| ☐ | LAB-ROOT-CA dans les racines approuvées du vCenter |
| ☐ | Certificat SSL du vCenter délivré par LAB-ROOT-CA, avec 192.168.50.12 en nom DNS et en adresse IP |
| ☐ | Snapshot du vCenter supprimé après validation |
| ☐ | SSO : verrouillage 5 tentatives, compte `administrator@vsphere.local` dans le coffre |
| ☐ | VAMI : SSH coupé, NTP DC01, sauvegarde planifiée |
| ☐ | ESXi : mode de verrouillage normal, SSH arrêté |
| ☐ | Aucun événement 2889, signature LDAP exigée, liaison de canal « lorsque pris en charge » |
| ☐ | TLS 1.0 et 1.1 désactivés, LDAPS et vCenter testés |
| ☐ | `adm-dom-afaye` dans Protected Users, GPO de tiering, comptes de service bloqués en interactif |
| ☐ | Windows LAPS sur OU=Serveurs |
| ☐ | Spouleur arrêté sur DC01, corbeille AD activée |

---

## 5. Quiz

<details><summary><b>1. Pourquoi le modèle LAB-vCenter utilise-t-il « Fournir dans la demande », et comment évite-t-on ESC1 ?</b></summary> <p>Le vCenter n'est pas dans l'AD : son nom doit venir de sa CSR. On neutralise ESC1 par l'approbation obligatoire du gestionnaire de la CA et par un droit d'inscription réservé aux Admins du domaine.</p></details>

<details><summary><b>2. Pourquoi le certificat du vCenter contient-il 192.168.50.12 deux fois, en nom DNS et en adresse IP ?</b></summary> <p>Le vCenter exige que son PNID, même quand c'est une IP, figure parmi les noms DNS du certificat. Les navigateurs et Horizon, eux, comparent une adresse IP aux entrées « adresse IP ». Il faut donc les deux.</p></details>

<details><summary><b>3. Pourquoi le mode hybride plutôt qu'une VMCA subordonnée ?</b></summary> <p>On ne remplace que le certificat visible par les utilisateurs. Une VMCA subordonnée pourrait émettre des certificats reconnus par tout le domaine : c'est trop de pouvoir pour une appliance.</p></details>

<details><summary><b>4. Pourquoi observer l'événement 2889 avant d'exiger la signature LDAP ?</b></summary> <p>Pour repérer les machines qui font encore du LDAP non signé. Exiger la signature sans vérifier pourrait couper l'une d'elles, par exemple le vCenter s'il était resté en LDAP simple.</p></details>

<details><summary><b>5. Pourquoi ne pas interdire l'accès réseau aux comptes de service ?</b></summary> <p>Le bind LDAP de svc-vcenter-ldap et les connexions d'Horizon sont des accès réseau. Les interdire couperait le vCenter et Horizon de l'AD.</p></details>

---

[⬅ 06 - vCenter en LDAPS](/fr/hyperviseur/VDI/06-vcenter-ldaps) · [Sommaire](/fr/hyperviseur/VDI/00-sommaire) · Page suivante : [08 - Préparer Horizon ➜](/fr/hyperviseur/VDI/08-preparation-horizon)
