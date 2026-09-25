---
title: Intégration de Teleport avec Active Directory
description: 
published: 1
date: 2026-09-25T15:45:46.293Z
tags: 
editor: markdown
dateCreated: 2025-06-28T10:46:54.499Z
---

# Tuto complet : Intégration de Teleport avec Active Directory
 
## Prérequis

* Serveur Linux (ex : Rocky Linux) avec Teleport installé
* Active Directory avec autorité de certification interne (ADCS) fonctionnelle
* Domaine configuré (ex : `alpha.lan`)

---

## Étape 1 : Créer un compte de service restrictif

Teleport nécessite un compte de service pour se connecter à votre domaine Active Directory. Pour une sécurité maximale, créez un compte de service dédié avec des autorisations restreintes.

Pour créer le compte de service :

Ouvrez PowerShell sur un ordinateur de domaine Windows.

Créez un compte de service avec un mot de passe généré aléatoirement en copiant et collant le script suivant dans la console PowerShell :

```powershell
$Name="Teleport Service Account"
$SamAccountName="svc-teleport"

# Générer un mot de passe complexe
Add-Type -AssemblyName 'System.Web'
do {
   $Password=[System.Web.Security.Membership]::GeneratePassword(15,1)
} until ($Password -match '\\d')
$SecureStringPassword=ConvertTo-SecureString $Password -AsPlainText -Force

# Créer le compte de service
New-ADUser `
  -Name $Name `
  -SamAccountName $SamAccountName `
  -AccountPassword $SecureStringPassword `
  -Enabled $true
```
Le mot de passe généré pour le compte de service est immédiatement supprimé. Teleport n'a pas besoin de ce mot de passe, car il utilise des certificats x509 pour l'authentification LDAP. Vous pouvez réinitialiser le mot de passe du compte de service si vous devez effectuer une authentification par mot de passe.

### Définir les permissions LDAP nécessaires

Définissez les autorisations minimales qui doivent être accordées au compte de service en exécutant le script suivant dans la console PowerShell :

```powershell
$DomainDN=$((Get-ADDomain).DistinguishedName)

# Créer le conteneur Teleport dans CDP
New-ADObject -Name "Teleport" -Type "container" -Path "CN=CDP,CN=Public Key Services,CN=Services,CN=Configuration,$DomainDN"

# Autoriser la création de conteneurs dans CDP
# Et la création/suppression d’objets cRLDistributionPoint
# Et l'écriture sur le champ certificateRevocationList
# Et la lecture sur NTAuthCertificates

$BasePath="CN=Teleport,CN=CDP,CN=Public Key Services,CN=Services,CN=Configuration,$DomainDN"
dsacls "CN=CDP,CN=Public Key Services,CN=Services,CN=Configuration,$DomainDN" /I:T /G "$($SamAccountName):CC;container;"
dsacls "$BasePath" /I:T /G "$($SamAccountName):CCDC;cRLDistributionPoint;"
dsacls "$BasePath" /I:T /G "$($SamAccountName):WP;certificateRevocationList;"
dsacls "CN=NTAuthCertificates,CN=Public Key Services,CN=Services,CN=Configuration,$DomainDN" /I:T /G "$($SamAccountName):RP;cACertificate;"
```

### Récupérer le SID du compte de service

```powershell
Get-AdUser -Identity $SamAccountName | Select SID
```
Vous utiliserez cette valeur pour le sidchamp lorsque vous configurerez les ldapparamètres dans une étape ultérieure.

---

## Étape 2 : Empêcher les connexions interactives

Les étapes suivantes modifient les objets de stratégie de groupe (GPO). La propagation des modifications apportées aux stratégies de groupe à tous les hôtes peut prendre du temps. Vous pouvez forcer l'application immédiate des modifications sur votre hôte actuel en ouvrant PowerShell et en exécutant gpupdate.exe /force. Cependant, la propagation de la modification aux autres hôtes du domaine peut prendre du temps.

Pour sécuriser le compte `svc-teleport`, créons un GPO qui empêche les connexions locales/RDP.

```powershell
$GPOName="Block teleport-svc Interactive Login"
New-GPO -Name $GPOName | New-GPLink -Target $((Get-ADDomain).DistinguishedName)
```

Dans l'éditeur de GPO :

* Aller dans `Computer Configuration > Policies > Windows Settings > Security Settings > Local Policies > User Rights Assignment`
* Modifier `Deny log on locally` et `Deny log on through Remote Desktop Services`
* Ajouter `svc-teleport`
![1.png](/teleport/1.png)
---

## Étape 3 : Créer un GPO pour autoriser les connexions via Teleport
Pour activer l’accès aux sessions de bureau Windows via Teleport, vous devez configurer un objet de stratégie de groupe qui permet aux ordinateurs Windows d’approuver l’autorité de certification Teleport et d’accepter l’authentification par carte à puce basée sur un certificat.

Vous devez procéder comme suit pour configurer l’objet de stratégie de groupe :

Exporter un certificat signé par l’autorité de certification Teleport pour un cluster Teleport existant.
Créez un nouvel objet de stratégie de groupe et importez le certificat Teleport signé.
Publiez le certificat Teleport signé sur le domaine Active Directory.
Publiez le certificat Teleport signé dans le magasin NTAuth.
Activer l'authentification par carte à puce.
Autoriser les connexions au bureau à distance.
Vous devez répéter ces étapes si vous faites pivoter l’autorité de certification de l’utilisateur Teleport.

### Exporter le certificat utilisateur de Teleport

```powershell
curl.exe -fo user-ca.cer https://teleport.alpha.lan/webapi/auth/export?type=windows
```
![2.png](/teleport/2.png)

### Créer un GPO

```powershell
$GPOName="Teleport Access Policy"
New-GPO -Name $GPOName | New-GPLink -Target $((Get-ADDomain).DistinguishedName)
```

### Dans le GPO :

* Aller dans `Computer Configuration > Policies > Windows Settings > Security Settings > Public Key Policies`
* Importer `user-ca.cer` dans `Trusted Root Certification Authorities`

![3.png](/teleport/3.png)

Cliquez avec le bouton droit sur Autorités de certification racines de confiance , puis cliquez sur Importer .

Utilisez l’assistant pour sélectionner et importer le certificat de téléportation.

### Publier le certificat dans AD

Pour publier le certificat Teleport dans le domaine Active Directory :

Ouvrez PowerShell et exécutez la commande suivante en utilisant le chemin d’accès au user-ca.cer fichier que vous avez exporté depuis Teleport :

```powershell
certutil -dspublish -f user-ca.cer RootCA
```
Cette commande permet aux contrôleurs de domaine de faire confiance à l'autorité de certification Teleport afin que l'authentification par carte à puce basée sur un certificat via Teleport puisse réussir.


Publier l'autorité de certification de téléportation dans le magasin

Pour que l'authentification avec les certificats émis par Teleport réussisse, l'autorité de certification Teleport doit également être publiée dans le magasin NTAuth de l'entreprise. Teleport publie régulièrement son autorité de certification une fois l'authentification effectuée, mais cette étape doit être effectuée manuellement la première fois pour que Teleport dispose d'un accès LDAP.

```powershell
certutil -dspublish -f user-ca.cer NTAuthCA
```
Forcez la récupération de l'AC depuis LDAP en exécutant la commande suivante :
```powershell
certutil -pulse  # Forcer la propagation LDAP
```

### Activer l’authentification smartcard

Teleport effectue une authentification basée sur un certificat en émulant une carte à puce.

Pour ajouter `l’authentification par carte à puce` à votre objet de stratégie de groupe :

* Vérifiez que l’ Teleport Access Policyobjet de stratégie de groupe est ouvert dans l’éditeur de stratégie de groupe.

* Développer `Computer Configuration > Policies > Windows Settings > Security Settings > System Services`

* Double-cliquez sur Carte à puce et sélectionnez Définir ce paramètre de stratégie .

* Sélectionnez Automatique , puis cliquez sur OK .

![4.png](/teleport/4.png)

### Autoriser les connexions RDP

* Vérifiez que l’ Teleport Access Policyobjet de stratégie de groupe est ouvert dans l’éditeur de stratégie de groupe.

* Développer `Computer Configuration > Policies > Administrative Templates > Windows Components > Remote Desktop Services > Remote Desktop Session Host > Connections`

* Cliquez avec le bouton droit sur `Autoriser les utilisateurs à se connecter à distance à l’aide des services Bureau à distance` , sélectionnez `Modifier` , `sélectionnez Activé` , puis cliquez sur `OK` .
![5.png](/teleport/5.png)
* Sous Hôte de session Bureau à distance, sélectionnez `Sécurité` .

* Cliquez avec le bouton droit sur `Exiger l'authentification de l'utilisateur pour les connexions à distance à l'aide de l'authentification au niveau du réseau` , sélectionnez `Modifier` , sélectionnez `Désactiv`é , puis cliquez sur OK .

* Faites un clic droit sur `Toujours demander le mot de passe lors de la connexion` , sélectionnez Modifier , sélectionnez Désactivé , puis cliquez sur OK .

* L'authentification par carte à puce basée sur un certificat Teleport génère un code PIN aléatoire pour chaque session de bureau et le transmet au bureau lors de l'établissement de la connexion RDP. Le code PIN n'étant jamais communiqué à l'utilisateur Teleport, la stratégie « `Toujours demander le mot de passe à la connexion` » doit être désactivée pour que l'authentification réussisse.

* Développez Configuration ordinateur, Stratégies, Paramètres Windows, Paramètres de sécurité pour sélectionner Pare-feu Windows avec sécurité avancée .
![6.png](/teleport/6.png)

### Ouvrir le pare-feu

* Développez `Configuration ordinateur, Stratégies, Paramètres Windows, Paramètres de sécurité pour sélectionner Pare-feu Windows avec sécurité avancée`

* Cliquez avec le bouton droit sur `Règles entrantes` , sélectionnez `Nouvelle règle` .

- Sous Prédéfini, sélectionnez Bureau à distance , puis cliquez sur Suivant .
- Sélectionnez Mode utilisateur (TCP-in) , puis cliquez sur Suivant .
- Sélectionnez Autoriser la connexion , puis cliquez sur Terminer .
![7.png](/teleport/7.png)

### Activer RemoteFX (amélioration des performances RDP)

* Pour finaliser la configuration de l' Teleport Access Policyobjet de stratégie de groupe, vous devez activer RemoteFX. RemoteFX est une technologie de compression qui améliore considérablement les performances des connexions Bureau à distance.

* Vérifiez que l’ Teleport Access Policyobjet de stratégie de groupe est ouvert dans l’éditeur de stratégie de groupe.

* DévelopperComputer `Configuration > Policies > Administrative Templates > Windows Components > Remote Desktop Services > Remote Desktop Session Host > Remote Session Environment > RemoteFX for Windows Server 2008 R2`

* Cliquez avec le bouton droit sur Configurer RemoteFX , sélectionnez Modifier , sélectionnez Activé , puis cliquez sur OK .

![8.png](/teleport/8.png)

```powershell
gpupdate.exe /force
```

---

## Étape 4 : Créer un modèle de certificat RDP ECC (si nécessaire)

Le client Teleport RDP nécessite des algorithmes cryptographiques sécurisés pour établir des connexions TLS. Cependant, Windows Server 2012 R2 ne prend pas en charge ces algorithmes par défaut.

Pour créer un modèle de certificat qui utilise la courbe elliptique P-384 et SHA384 comme algorithme de signature :

Cliquez sur Démarrer, Panneau de configuration et Outils d’administration pour sélectionner `Autorité de certification` .

Ouvrez votre ordinateur CA, cliquez avec le bouton droit sur `Modèles de certificats` , puis sélectionnez Gérer .

Sélectionnez le modèle `Ordinateur` , faites un clic droit, puis sélectionnez `Dupliquer le modèle` .
![9.png](/teleport/9.png)
Sélectionnez l’ onglet `Compatibilité` :
Modifiez l’autorité de certification sur Windows Server 2012 R2 , puis cliquez sur OK .
Modifiez le destinataire du certificat sur Windows Server 2012 R2 , puis cliquez sur OK .
![10.png](/teleport/10.png)
Sélectionnez l’ onglet `Général` :
Modifiez le nom d'affichage du modèle en RemoteDesktopAccess .
Vérifiez que le nom du modèle est également RemoteDesktopAccess .

Sélectionnez l’ onglet `Cryptographie` :
Changer la catégorie de fournisseur en fournisseur de stockage de clés .
Changez le nom de l'algorithme en ECDH_P384 .
Modifier le hachage de la demande en SHA384 .
![11.png](/teleport/11.png)
Sélectionnez l’ onglet `Extensions` :
Sélectionnez Politiques d’application , puis cliquez sur Modifier .
Supprimer toutes les entrées de la liste.
![12.png](/teleport/12.png)
Sélectionnez l’ onglet `Sécurité` :
Sélectionnez Ordinateurs du domaine et accordez au groupe les autorisations de lecture et d’inscription .
![13.png](/teleport/13.png)
Cliquez sur OK pour créer le modèle.

Revenez à la console de l’autorité de certification, cliquez avec le bouton droit sur Modèles de certificats .

Cliquez sur Nouveau , sélectionnez Modèle de certificat à émettre , puis sélectionnez RemoteDesktopAccess .

Cliquez sur OK .
![14.png](/teleport/14.png)


### GPO : forcer l’usage de ce modèle

Pour mettre à jour l’objet de stratégie de groupe Teleport afin d’utiliser le nouveau modèle de certificat :

Ouvrez l’ Teleport Access Policyobjet de stratégie de groupe dans l’éditeur de stratégie de groupe.

* Sécurité RDP > `Computer Configuration > Policies > Administrative Templates > Windows Components > Remote Desktop Services > Remote Desktop Session Host > Security`
* Activer `Certificate Services Client - Auto-Enrollment`
* Double-cliquez sur `Client des services de certificats - Inscription automatique` , puis sélectionnez Activé dans le modèle de configuration.

![6.png](/teleport/6.png)

```powershell
gpupdate.exe /force
```

---

## Étape 5 : Exporter le certificat de l’ADCS (LDAPS)

Sur le serveur ADCS :

Teleport se connecte à votre contrôleur de domaine via LDAPS. Vous devez donc indiquer à Teleport que le certificat envoyé par votre contrôleur de domaine lors de la connexion initiale est approuvé. Si le certificat de votre contrôleur de domaine est approuvé par le référentiel système du système exécutant Teleport, vous pouvez ignorer cette étape.

Si vous ne parvenez pas à obtenir le certificat d'autorité de certification LDAP, vous pouvez ignorer la vérification TLS en définissant insecure_skip_verify: true. Cependant, vous ne devez pas ignorer la vérification TLS dans les environnements de production.

Pour exporter un certificat CA :

* Cliquez sur Démarrer, Panneau de configuration et Outils d’administration pour sélectionner Autorité de certification .
* Sélectionnez votre ordinateur CA, faites un clic droit, puis sélectionnez Propriétés .
* Dans l’onglet Général, cliquez sur Afficher le certificat .
* Sélectionnez Détails , puis cliquez sur Copier dans un fichier .
* Cliquez sur Suivant dans l'assistant d'exportation de certificat et assurez-vous que le binaire codé DER X.509 (.CER) est sélectionné.
* Sélectionnez un nom et un emplacement pour le certificat et cliquez sur l’assistant.
* Transférez le fichier exporté vers le système sur lequel vous exécutez Teleport. Vous pouvez ajouter ce certificat au référentiel de confiance de votre système ou fournir le chemin d'accès à la der_ca_filevariable de configuration.
![15.png](/teleport/15.png)
---

## Étape 6 : Installer et configurer Teleport Desktop Service

### Générer un token

```bash
tctl tokens add --type=windowsdesktop
```
Copiez le jeton d’invitation dans un fichier sur l’hôte Linux qui exécutera le service de bureau Windows.

### Configuration `/etc/teleport.yaml`

```yaml
version: v3
teleport:
  nodename: SVR-TELEPORT-01
  data_dir: /var/lib/teleport
  join_params:
    token_name: "2d2c8fa927994f3728d9b66430a5abb7"
    method: token
  log:
    output: stderr
    severity: INFO
    format:
      output: text
auth_service:
  enabled: "yes"
  listen_addr: 0.0.0.0:3025
  cluster_name: teleport.alpha.lan
  proxy_listener_mode: multiplex
ssh_service:
  enabled: "yes"
proxy_service:
  enabled: "yes"
  web_listen_addr: 0.0.0.0:443
  public_addr: teleport.alpha.lan:443
  https_keypairs:
    - cert_file: /etc/ssl/certs/teleport.alpha.lan.crt
      key_file: /etc/ssl/private/teleport.alpha.lan.key
windows_desktop_service:
  enabled: true
  public_addr: "teleport.alpha.lan:3028"
  listen_addr: "0.0.0.0:3028"
  ldap:
    addr: 'SVR-DC-01.alpha.lan:636'
    domain: 'alpha.lan'
    username: 'ALPHA\svc-teleport'
    sid: 'S-1-5-21-4124969712-4057984771-8924275-1130'
    insecure_skip_verify: true
    der_ca_file: "/home/teleport/ca.der"
  discovery:
    base_dn: '*'
```

### Activer le service

```bash
sudo systemctl enable teleport
sudo systemctl start teleport
journalctl -fu teleport
```
![16.png](/teleport/16.png)

---

## Si vous rencontrez des erreurs 

### Vérifier la synchronisation temporelle

```bash
timedatectl
```

Sur Windows :

```powershell
w32tm /resync /nowait
```

Les horloges doivent être synchronisées à la seconde près sinon LDAPS/TLS peut échouer.

## Étape 8 : Créer un rôle Teleport pour l'accès RDP

Les utilisateurs de Teleport doivent disposer des autorisations appropriées pour accéder aux bureaux Windows distants. Par exemple, vous pouvez créer un rôle donnant accès à tous les libellés de bureaux Windows et à l'utilisateur local « Administrateur ».

Pour créer un rôle permettant d’accéder aux postes de travail Windows :

Créez un fichier appelé windows-desktop-admins.yamlavec le contenu suivant :

```yaml
kind: role
metadata:
  # insert the name of your role here:
  name: desktop-admins
spec:
  allow:
    # List of windows desktop access labels that users can open desktop sessions to
    windows_desktop_labels:
      "*": "*"
    # Windows logins a user is allowed to use for desktop sessions.
    windows_desktop_logins:
      - '{{internal.windows_logins}}'
      - Administrateur
version: v5

```

Appliquer le rôle :

```bash
sudo /opt/teleport/system/bin/tctl create -f windows-desktop-admins.yaml
```

Ajouter le rôle à votre utilisateur :

Attribuez le windows-desktop-adminsrôle à votre utilisateur Teleport en exécutant les commandes appropriées pour votre fournisseur d'authentification :

```bash
ROLES=$(tsh status -f json | jq -r '.active.roles | join(",")')
sudo /opt/teleport/system/bin/tctl users update $(tsh status -f json | jq -r '.active.username') --set-roles "$ROLES,windows-desktop-admins"
```
Maintenant que vous disposez d'un hôte Linux exécutant le service Bureau Windows et d'un rôle permettant aux utilisateurs Teleport de se connecter aux ordinateurs Windows, vous pouvez utiliser l'utilisateur Teleport auquel ce windows-desktop-adminsrôle a été attribué pour vous connecter aux bureaux Windows depuis l'interface Web Teleport. Vous pouvez également vous connecter via Teleport Connect.

Reconnectez-vous à Teleport pour prendre en compte le rôle.

Dans l’interface Web :

* Se connecter avec l’utilisateur
* Aller dans **Ressources > Type: Bureaux**
* Cliquer sur **Connecter** pour établir la session RDP

---

## Découverte LDAP avancée

Dans la configuration `teleport.yaml` :

```yaml
windows_desktop_service:
  enabled: true
  discovery:
    base_dn: '*'
    filters:
      - '(location=Oakland)'
      - '(!(primaryGroupID=516))'
    label_attributes:
      - 'location'
      - 'department'
```

Les machines AD auront des labels comme :

* `ldap/location=Oakland`
* `ldap/department=Engineering`

---

## Sécurisation supplémentaire

Modifier le droit "Ajouter des ordinateurs au domaine" dans GPO par défaut :

* Aller dans `GPMC > Forêt > Domaine > Stratégie contrôleur de domaine par défaut`
* Modifier > `Configuration ordinateur > Paramètres Windows > Paramètres de sécurité > Stratégies locales > Attribution des droits utilisateur`
* Retirer le groupe `Utilisateurs authentifiés` de "Ajouter des postes de travail au domaine"

```
```

