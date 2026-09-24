---
title: Installation d'un cluster K3S
description: 
published: 1
date: 2026-09-24T18:26:08.149Z
tags: 
editor: markdown
dateCreated: 2025-12-02T09:57:35.767Z
---

# Procédure d'installation Cluster Kubernetes K3s (Rocky Linux)

Ce document décrit la procédure pour installer un cluster K3s sur 3 serveurs Rocky Linux.
 
## Inventaire des machines
| Rôle | Hostname | IP | OS |
| :--- | :--- | :--- | :--- |
| **Master** | K3S-Master | 192.168.50.210 | Rocky Linux |
| **Worker** | K3S-Worker1 | 192.168.50.211 | Rocky Linux |
| **Worker** | K3S-Worker2 | 192.168.50.212 | Rocky Linux |
---

## Prérequis (Sur TOUS les nœuds)
*À exécuter sur le Master et les Workers avant l'installation.*

### Mise à jour et outils
```
sudo dnf update -y
sudo dnf install -y curl
```
Configuration du Pare-feu
Pour un environnement interne (Lab), nous désactivons le pare-feu pour faciliter la communication entre les pods (Flannel/VXLAN).


sudo systemctl disable --now firewalld
(Note pour la production : Si le pare-feu est requis, ouvrir les ports TCP 6443, UDP 8472 et TCP 10250).

# Installation du Master
Cible : K3S-Master (192.168.50.210)

A. Installation du service
```
sudo curl -sfL [https://get.k3s.io](https://get.k3s.io) | sh -
```
B. Configuration de l'accès utilisateur (kubectl)
Par défaut, le fichier de configuration appartient à root. Pour utiliser kubectl avec l'utilisateur rocky :


## 1. Création du dossier de config
```
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown rocky:rocky ~/.kube/config
chmod 600 ~/.kube/config
```


```
sudo cat /var/lib/rancher/k3s/server/node-token
```
> Copiez la chaîne de caractères (ex: K1049...::server:xxx).

# Installation des Workers
Cible : K3S-Worker1 & K3S-Worker2

Remplacez <VOTRE_TOKEN> ci-dessous par celui récupéré à l'étape précédente.


## Définition des variables
MASTER_IP="192.168.50.210"
NODE_TOKEN="<VOTRE_TOKEN>"

## Installation en mode Agent (Worker)
```
sudo curl -sfL [https://get.k3s.io](https://get.k3s.io) | K3S_URL=https://${MASTER_IP}:6443 K3S_TOKEN=${NODE_TOKEN} sh -
```

# Validation du Cluster
Retournez sur le Master pour vérifier que les nœuds sont bien enregistrés.

```
kubectl get nodes
```
Résultat attendu :

Plaintext
```
NAME          STATUS   ROLES                  AGE     VERSION
k3s-master    Ready    control-plane,master   10m     v1.2x...
k3s-worker1   Ready    <none>                 2m      v1.2x...
k3s-worker2   Ready    <none>                 2m      v1.2x...
```
## Maintenance & Astuces

Activer l'autocomplétion 
Pour ne pas avoir à taper les commandes entières :

```
echo 'source <(kubectl completion )' >> ~/.rc
source ~/.rc
```
Redémarrer K3s
Master :
```
sudo systemctl restart k3s
```

Worker : 
```
sudo systemctl restart k3s-agent
```

Désinstallation propre
Master : /usr/local/bin/k3s-uninstall.sh

Worker : /usr/local/bin/k3s-agent-uninstall.sh