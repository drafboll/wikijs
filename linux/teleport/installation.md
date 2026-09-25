---
title: Installation Teleport Rocky
description: 
published: 1
date: 2026-09-25T15:45:18.652Z
tags: 
editor: markdown
dateCreated: 2025-06-06T16:41:02.133Z
---

# Guide d'installation de Teleport avec certificat de l'ADCS


## 1. Présentation

Teleport est une plateforme de bastion qui fournit un accès sécurisé aux serveurs, applications, bases de données et clusters Kubernetes. Ici, on installe un nœud **all-in-one** avec l’interface web exposée derrière Caddy en reverse proxy.

Machines utilisées :

* **Teleport** : `192.168.30.40`
* **ADCS** : `192.168.20.10`

---

## 2. Installation de Teleport

### 2.1 Téléchargement et installation

```bash
sudo curl https://cdn.teleport.dev/install.sh | bash -s 17.5.2
```



### 2.2 Configuration `teleport.yaml`

```yaml
sudo /opt/teleport/system/bin/teleport configure -o file --acme --acme-email=e-mail@indiogroup.lan --cluster-name=teleport.indiogroup.lan
```

Active et démarre le service :

```bash
sudo systemctl enable --now teleport
sudo systemctl start teleport
```

ce qui doit etre changé dans le yaml :
```bash
vi /etc/teleport.yaml
```

```bash

version: v3
teleport:
  nodename: SVR-TELEPORT-01
  data_dir: /var/lib/teleport
  join_params:
    token_name: ""
    method: token
  log:
    output: stderr
    severity: INFO
    format:
      output: text
auth_service:
  enabled: "yes"
  listen_addr: 0.0.0.0:3025
  cluster_name: teleport.indiogroup.lan
  proxy_listener_mode: multiplex
ssh_service:
  enabled: "yes"
proxy_service:
  enabled: "yes"
  web_listen_addr: 0.0.0.0:443
  public_addr: teleport.indiogroup.lan:443
  https_keypairs: [] ## mettre les certificats
```

---
## 3. Ouverture des ports (firewalld)

### Sur la machine Teleport :

```bash
sudo firewall-cmd --add-port=3025-3026/tcp --permanent
sudo firewall-cmd --add-port=3028/tcp --permanent
sudo firewall-cmd --add-port=3036/tcp --permanent
sudo firewall-cmd --add-port=3080/tcp --permanent
sudo firewall-cmd --reload
```

### Sur la machine Caddy :

```bash
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --reload
```


* Port `3025` est utilisé en interne pour la communication entre composants Teleport
* Pas besoin d’ouvrir `3022` car `ssh_service` est désactivé
* Le reverse proxy Caddy prend en charge le HTTPS, donc pas besoin de gérer TLS dans Teleport
* En cas de modification du `.yaml`, redémarrer avec :

```bash
sudo systemctl restart teleport
```

# Générer un certificat TLS signé par AD CS pour Teleport

## Objectif
Obtenir un certificat TLS valide pour le serveur `teleport.indiogroup.lan`, signé par l'autorité de certification interne AD CS de votre domaine Active Directory, à utiliser avec le service proxy de Teleport.

---

## Étape 1 : Générer une clé privée et une CSR (Certificate Signing Request)

> À effectuer sur le serveur Linux `SVR-TELEPORT-01`

### 1.1 Créer la clé privée avec ECC

```bash
openssl ecparam -genkey -name prime256v1 -out teleport.indiogroup.lan.key
```

> Cela génère une clé privée ECC (Elliptic Curve Cryptography) plus sécurisée que RSA.

---

### 1.2 Créer un fichier de configuration pour la CSR avec les SAN (Subject Alternative Names)

```bash
nano teleport-san.cnf
```

Contenu du fichier `teleport-san.cnf` :

```ini
[ req ]
prompt = no
distinguished_name = dn
req_extensions = req_ext

[ dn ]
CN = teleport.indiogroup.lan
O = indiogroup
L = Paris
C = FR

[ req_ext ]
subjectAltName = DNS:teleport.indiogroup.lan, DNS:svr-teleport-01.indiogroup.lan, IP:192.168.70.10
```

> Ajoutez ici toutes les IPs ou noms nécessaires pour que le certificat soit accepté par vos clients.

---

### 1.3 Générer la CSR (fichier .csr) à partir de la clé et du fichier `.cnf`

```bash
openssl req -new -key teleport.indiogroup.lan.key -out teleport.indiogroup.lan.csr -config teleport-san.cnf
```

---

### 1.4 Vérifier la CSR

```bash
openssl req -text -noout -verify -in teleport.indiogroup.lan.csr
```

Vous devez voir :
```bash
Certificate request self-signature verify OK
Certificate Request:
    Data:
        Version: 1 (0x0)
        Subject: CN=teleport.indiogroup.lan, O=indiogroup, L=Paris, C=FR
        Subject Public Key Info:
            Public Key Algorithm: id-ecPublicKey
                Public-Key: (256 bit)
                pub:
                    04:80:40:22:a4:fe:57:e9:af:55:4b:87:c3:c9:dc:
                    b4:b8:0f:0e:22:b6:75:0f:2a:2c:b1:16:02:bd:bb:
                    2d:95:7c:b1:df:df:a7:9b:a9:52:81:a1:31:b3:97:
                    14:1c:6b:e4:e8:80:52:ae:07:ed:99:ab:36:74:5f:
                    db:7c:4d:93:59
                ASN1 OID: prime256v1
                NIST CURVE: P-256
        Attributes:
            Requested Extensions:
                X509v3 Subject Alternative Name:
                    DNS:teleport.indiogroup.lan, IP Address:192.168.70.10
    Signature Algorithm: ecdsa-with-SHA256
    Signature Value:
        30:46:02:21:00:b8:dd:10:9d:06:ef:27:1a:05:17:01:31:d4:
        67:a6:21:d2:9e:64:14:c6:16:57:23:45:d5:ca:d5:e7:ee:f1:
        3e:02:21:00:c5:57:51:fb:6e:64:77:a2:0d:16:59:61:4a:e2:
        4a:f8:26:17:6b:6d:e9:ef:d2:87:03:54:af:cc:de:dd:d2:48
```


---


## Étape 2 : Envoyer la CSR à l’AD CS pour signature


La suite des opérations va être effectuée à partir du serveur AD CS (ou d'un serveur disposant des outils d'administration nécessaires). Vous devez disposer d'un modèle de certificat adapté pour "Serveur Web" avec le rôle "Authentification du serveur" pour délivrer le certificat TLS.

Dans la console AD CS (certsrv), sélectionnez un modèle existant, ici « Serveur Web », puis réalisez un clic-droit, "Dupliquer le modèle" , via la section "Modèles de certificats" puis "Gérer". Vous devez ensuite l'ajouter à la liste des modèles publiés. Si besoin, référez-vous à notre cours dédié à AD CS pour obtenir de l'aide à ce sujet.
![1.png](/adcs/1.png)

Voici un aperçu de certaines propriétés de ce modèle dont le nom est indiogroup-ServeurWebLinux-v1.
![2.png](/adcs/2.png)![3.png](/adcs/3.png)


Nous allons maintenant télécharger le fichier CSR depuis le serveur Web, via la commande scp (qui s'appuie sur une connexion SSH). 10.10.10.11 étant l'adresse IP du serveur Web.
> Cette étape se fait sur un poste Windows avec les outils AD CS installés.

### 2.1 Copier la CSR vers votre PC Windows

Depuis Windows (dans un terminal PowerShell ou CMD) ou faire un transfert avec termuis ou mobaxterm

```powershell
scp teleport@192.168.70.10:/home/teleport/teleport.indiogroup.lan.csr C:\TEMP
```

---

### 2.2 Soumettre la CSR à l'AD CS

Dans PowerShell :

```powershell
certreq -submit -attrib "CertificateTemplate:indiogroup-ServeurWebLinux-v1" C:\TEMP\teleport.indiogroup.lan.csr C:\TEMP\teleport.indiogroup.lan.cer
```
![4.png](/adcs/4.png)
> Le nom du modèle `"indiogroup-ServeurWebLinux-v1"` est un exemple. Il doit :
> - Être publié dans la console "certsrv"
> - Avoir comme usage "Authentification de serveur"
> - Accepter les CSR générées depuis Linux (autoriser l'import de clé externe)

Une fenêtre vous demandera de choisir l’autorité de certification. Validez.

---

### 2.3 Récupérer le certificat signé

Une fois le fichier `.cer` récupéré, copiez-le sur votre serveur :

```powershell
scp C:\TEMP\teleport.indiogroup.lan.cer teleport@192.168.70.10:/home/teleport/
```

---

## Étape 3 : Configurer Teleport avec le certificat

### 3.1 Placer les fichiers aux bons emplacements

Sur `SVR-TELEPORT-01` :

```bash
# Crée les dossiers s'ils n'existent pas déjà (le dossier /etc/ssl/private existe normalement)
sudo mkdir -p /etc/ssl/private/
sudo mkdir -p /etc/ssl/certs/

# Copie du certificat (CRT) dans le dossier public
sudo cp teleport.indiogroup.lan.cer /etc/ssl/certs/teleport.indiogroup.lan.crt

# Copie de la clé privée dans le dossier sécurisé
sudo cp teleport.indiogroup.lan.key /etc/ssl/private/teleport.indiogroup.lan.key

# Attribution des bons droits
sudo chmod 644 /etc/ssl/certs/teleport.indiogroup.lan.crt
sudo chmod 600 /etc/ssl/private/teleport.indiogroup.lan.key
sudo chown root:root /etc/ssl/certs/teleport.indiogroup.lan.crt
sudo chown root:root /etc/ssl/private/teleport.indiogroup.lan.key

```

---

### 3.2 Modifier le fichier `/etc/teleport.yaml`

```yaml
proxy_service:
  enabled: "yes"
  web_listen_addr: 0.0.0.0:443
  public_addr: teleport.indiogroup.lan:443
  https_keypairs:
    - key_file: /etc/teleport/certs/teleport.indiogroup.lan.key
      cert_file: /etc/teleport/certs/teleport.indiogroup.lan.cer
  https_keypairs_reload_interval: 1h
```

> 🛠️ N’oubliez pas de désactiver `acme:` s’il est présent, car vous utilisez un certificat manuellement signé.

---

### 3.3 Redémarrer Teleport

```bash
sudo systemctl restart teleport
sudo systemctl status teleport
```

---

## Étape 4 : Tester l’accès

Depuis un poste client :

```bash
curl -vk https://teleport.indiogroup.lan
```
![5.png](/adcs/5.png)
Ou ouvrez le site dans un navigateur :  
[https://teleport.indiogroup.lan](https://teleport.indiogroup.lan)

> Si votre certificat est auto-signé par AD CS, ajoutez la chaîne d’approbation (Root CA) dans les navigateurs ou GPO.

---

## V. Ajout d'un utilisateur Teleport avec tous les rôles

Une fois Teleport fonctionnel et sécurisé avec TLS, il est temps d'ajouter un utilisateur.

### Objectif

Créer un utilisateur `alexandre` avec **tous les rôles disponibles** et connaître les **commandes utiles** à la gestion des accès via Teleport.

---

### Liste des rôles standards de Teleport

- `access` : accès classique (utilisateur standard)
- `editor` : modification de ressources (mais pas accès complet)
- `auditor` : audit (lecture seule des logs et configs)
- `db`, `kube`, `windows`, `desktop`, `access_request` : rôles plus spécifiques si utilisés

---

### Création de l'utilisateur avec tous les rôles

```bash
sudo /opt/teleport/system/bin/tctl users add alexandre \
  --roles=access,editor,auditor,admin \
  --logins=alexandre
```

Sortie :
```bash
User "alexandre" has been created but requires a password. Share this URL with the user to complete user setup, link is valid for 1h:
https://teleport.indiogroup.lan:443/web/invite/0ac25116badee82180e9e5445c5eeea4

NOTE: Make sure teleport.indiogroup.lan:443 points at a Teleport proxy which users can access.
[teleport@SVR-TELEPORT-01 ~]$
```
![6.png](/adcs/6.png)![7.png](/adcs/7.png)
![8.png](/adcs/8.png)