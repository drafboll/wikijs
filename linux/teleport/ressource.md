---
title: Ajouter des ressources dans Teleport
description: 
published: 1
date: 2026-09-25T15:44:44.883Z
tags: 
editor: markdown
dateCreated: 2025-06-28T14:03:54.706Z
---


# Ajouter des ressources dans Teleport

## Comment ajouter des ressources à Teleport ?
 
Pour ajouter une nouvelle ressource dans Teleport :

1. Connecte-toi à l’interface Web de Teleport (`https://teleport.alpha.lan/`)
![1.png](/ressource/1.png)
2. Clique sur le bouton **"Enroll New Resource"** en haut à droite


## Se connecter à un serveur Linux (SSH)

### Étape 1 – Choix de l’OS
Pour ajouter une nouvelle ressource dans Teleport, nous devons cliquer sur le bouton "Enroll New Resource" en haut à droite.

Comme le montre l'image ci-dessous, Teleport prend en charge des ressources diverses et variées : Linux, macOS, Windows, SQL Server, MySQL, Kubernetes, etc...

Prenons l’exemple d’une machine Debian nommée `SVR-ZABBIX-01`.  
Depuis le menu d’ajout, clique sur **“RHEL 8+/CentOS Stream 9+”**.  
Un **assistant interactif** va s’ouvrir.

![2.png](/ressource/2.png)

---
### Étape préliminaire – Configuration de la machine Linux

Avant d’installer Teleport sur une machine Linux, il faut :

- S’assurer que le fichier `/etc/hosts` contient une résolution correcte pour `teleport.alpha.lan`
```bash
192.168.40.10   SVR-K8S-MASTER.alpha.lan
192.168.40.20   SVR-K8S-WORKER-01.alpha.lan
192.168.40.30   SVR-K8S-WORKER-02.alpha.lan
192.168.40.40   SVR-K8S-WORKER-03.alpha.lan
192.168.30.10   SVR-ZABBIX-01.alpha.lan
192.168.30.30   SVR-GRAYLOG-01.alpha.lan
192.168.60.10   SVR-CADDY-01.alpha.lan
192.168.70.10   SVR-TELEPORT-01.alpha.lan
192.168.70.10   TELEPORT.alpha.lan
```

- Importer le certificat de l’autorité interne utilisée par Teleport :
[rootca.cer](/ressource/rootca.cer)

```bash
sudo cp rootCA.cer /etc/pki/ca-trust/source/anchors/teleport_rootCA.crt
sudo update-ca-trust extract
openssl s_client -connect teleport.alpha.lan:443
```

### Étape 2 – Déploiement du service Teleport

Pour commencer, nous allons ajouter une machine sous Debian (nommée SVR-ZABBIX-01) : la connexion vers cet hôte sera établie via SSH. Dans ce cas, nous allons cliquer sur "RHEL 8+/CentOS Stream 9+" dans la liste. Un assistant sera exécuté.

Le service Teleport doit être déployé sur la machine Linux. Là encore, Teleport facilite l'opération en mettant à notre disposition un script d'installation prêt à l'emploi. Il suffit de copier la ligne de commande qui s'affiche dans l'assistant :

Cliquer sur skip a Label 
![3.png](/ressource/3.png)

![4.png](/ressource/4.png)
```bash
sudo bash -c "$(curl -fsSL https://teleport.alpha.lan/scripts/8e68473a9d226b6797c51c2028268b88/install-node.sh)"
```
![5.png](/ressource/5.png)
Et exécute-la en **SSH sur le serveur Linux cible (`SVR-ZABBIX-0`)**.

Suite à l'installation du service Teleport, il y a bien une communication établie avec le serveur Teleport. D'ailleurs, l'assistant Teleport nous donne l'information : "Successfully detected your new Teleport instance". Nous pouvons cliquer sur "Next".

![6.png](/ressource/6.png)



### Étape 3 – Définir les utilisateurs Linux autorisés

À l’étape “Set Up Access”, sélectionne ou indique les utilisateurs existants sur la machine, par exemple :

![7.png](/ressource/7.png)

Puis clique sur **Next**.

---

### Étape 4 – Tester la connexion (optionnel)

Tu peux cliquer sur **Test Connection** pour valider immédiatement l’accès.  
Cette étape est facultative mais conseillée.
![8.png](/ressource/8.png)

---
