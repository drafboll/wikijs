---
title: Infrastructure KVM, HA ET CEPH
description: 
published: 1
date: 2026-09-25T15:47:03.488Z
tags: 
editor: markdown
dateCreated: 2026-05-27T14:40:45.429Z
---

# Tutoriel Complet : Infrastructure KVM, HA et Ceph

Ce guide rassemble toutes les étapes de configuration des TP0, TP1, TP2 et TP3.
 
---

## TP0 : Préparation de l'infrastructure et du réseau

### 1. Installation des paquets requis (Sur la VM Template avant clonage)
Installez les dépôts et tous les outils nécessaires :
```bash
sudo dnf install epel-release centos-release-ceph-squid.noarch -y
```

Activez les modules de haute disponibilité et CRB :
```bash
sudo dnf config-manager --set-enabled highavailability crb
```

Installez l'ensemble des composants de virtualisation, de cluster et de stockage :
```bash
sudo dnf install wget vim bash-completion chrony libvirt libvirt-daemon libvirt-daemon-driver-storage-rbd libvirt-daemon-driver-network libvirt-daemon-config-network qemu-kvm virt-install bridge-utils pcs pacemaker ceph -y
```
*(Une fois ces commandes passées, éteignez votre VM Template et clonez-la 3 fois pour créer node1, node2 et node3).*

### 2. Configuration réseau (Exemple pour le Node 1 - 192.168.50.20)
*À faire depuis la console VMware.* Définissez le nom d'hôte de la première machine :
```bash
sudo hostnamectl set-hostname node1
```

Créez l'interface de type pont (bridge) nommée `br0` :
```bash
sudo nmcli connection add type bridge ifname br0 con-name br0
```

Attribuez l'adresse IP statique, la passerelle du NAT VMware et le DNS au bridge :
```bash
sudo nmcli connection modify br0 ipv4.addresses 192.168.50.20/24 ipv4.gateway 192.168.50.2 ipv4.dns 8.8.8.8 ipv4.method manual
```

Asservissez votre carte réseau physique (ex: `ens160`) au bridge `br0` :
```bash
sudo nmcli connection add type ethernet ifname ens160 master br0 con-name br0-slave
```

Supprimez l'ancienne configuration de votre carte réseau pour éviter les conflits d'IP :
```bash
sudo nmcli connection delete ens160
```

Activez le bridge pour appliquer toute la configuration réseau :
```bash
sudo nmcli connection up br0
```
*(Pour le Node 2 et le Node 3, répétez ces étapes en adaptant le hostname et en remplaçant l'IP par `192.168.50.30` et `192.168.50.40`).*

---

## TP1 : Création d'une machine virtuelle Debian 13 (Console Série)

### 1. Démarrage de l'hyperviseur
Activez et démarrez le service de virtualisation Libvirt :
```bash
sudo systemctl enable --now libvirtd
```

### 2. Gestion du pool de stockage pour les ISOs
Créez le dossier physique qui va recevoir vos fichiers ISO :
```bash
sudo mkdir -p /usr/share/libvirt/iso
```

Déclarez le pool de stockage nommé `pool_iso` dans Libvirt :
```bash
sudo virsh pool-define-as --type dir --target /usr/share/libvirt/iso --name pool_iso
```

Construisez et préparez le pool de stockage :
```bash
sudo virsh pool-build pool_iso
```

Démarrez le pool pour le rendre actif :
```bash
sudo virsh pool-start pool_iso
```

Configurez le pool pour qu'il s'active automatiquement à chaque démarrage du serveur :
```bash
sudo virsh pool-autostart pool_iso
```

### 3. Téléchargement de l'image d'installation
Placez-vous dans le dossier du pool :
```bash
cd /usr/share/libvirt/iso
```

Téléchargez l'image ISO de Debian 13 (Testing) via `wget` :
```bash
sudo wget -O debian13-netinst.iso [https://cdimage.debian.org/cdimage/weekly-builds/amd64/iso-cd/debian-testing-amd64-netinst.iso](https://cdimage.debian.org/cdimage/weekly-builds/amd64/iso-cd/debian-testing-amd64-netinst.iso)
```

Forcez Libvirt à rafraîchir sa liste pour qu'il détecte votre nouveau fichier ISO :
```bash
sudo virsh pool-refresh pool_iso
```

### 4. Déploiement de la machine virtuelle
Lancez la création de la VM configurée pour une installation via la console série (vitesse 115200) :
```bash
sudo virt-install \
  --virt-type kvm \
  --name debian13 \
  --vcpus 1 \
  --memory 1024 \
  --disk size=20,format=qcow2 \
  --network bridge=br0 \
  --location /usr/share/libvirt/iso/debian13-netinst.iso \
  --os-variant debian12 \
  --boot hd,cdrom,menu=on \
  --console pty,target_type=serial \
  --extra-args "console=ttyS0,115200n8"
```
*(Pour quitter la console série sans éteindre la VM, utilisez le raccourci clavier `Ctrl` + `]`)*.

---

## TP2 : Déploiement d'une VM Debian avec serveur graphique VNC

### 1. Configuration du pare-feu
Autorisez les ports VNC (5900 et suivants) dans le pare-feu de votre machine Rocky Linux :
```bash
sudo firewall-cmd --add-port=5900-5910/tcp --permanent
```

Rechargez le pare-feu pour appliquer la règle :
```bash
sudo firewall-cmd --reload
```

### 2. Déploiement de la machine virtuelle
Lancez la création de la VM en mode graphique VNC (sans console automatique au terminal) :
```bash
sudo virt-install \
  --virt-type kvm \
  --name debian-vnc \
  --vcpus 1 \
  --memory 1024 \
  --disk size=20,format=qcow2 \
  --network bridge=br0 \
  --cdrom /usr/share/libvirt/iso/debian13-netinst.iso \
  --os-variant debian12 \
  --boot hd,cdrom \
  --graphics vnc,listen=0.0.0.0 \
  --noautoconsole
```

### 3. Identification du port d'écoute VNC
Vérifiez quel numéro d'affichage Libvirt a attribué à votre VM (ex: `:0` signifie port `5900`, `:1` signifie port `5901`) :
```bash
sudo virsh vncdisplay debian-vnc
```
*(Ouvrez ensuite votre logiciel VNC Viewer sur votre machine physique et connectez-vous à `192.168.50.20:5900`).*

---

## TP3 : Configuration de libvirt en mode Rootless complet

### 1. Attribution des groupes à l'utilisateur
Ajoutez l'utilisateur `rocky` aux groupes système nécessaires pour piloter KVM :
```bash
sudo usermod -aG kvm,libvirt,qemu rocky
```

### 2. Modification des fichiers de configuration globale
Éditez le premier fichier de configuration avec `vi` :
```bash
sudo vi /etc/libvirt/libvirt.conf
```
*(Dans `vi`, appuyez sur `i`, cherchez ou ajoutez la ligne `uri_default = "qemu:///session"`, puis faites `Échap`, tapez `:wq` et `Entrée`).*

Éditez le second fichier de configuration avec `vi` :
```bash
sudo vi /etc/libvirt/libvirt-admin.conf
```
*(Dans `vi`, appuyez sur `i`, ajoutez la ligne `uri_default = "qemu:///session"`, puis faites `Échap`, tapez `:wq` et `Entrée`).*

### 3. Configuration des accès au pool ISO (ACL)
Installez l'utilitaire de gestion des listes de contrôle d'accès :
```bash
sudo dnf install acl -y
```

Donnez les permissions de lecture et d'exécution à l'utilisateur `rocky` sur le dossier des ISOs :
```bash
sudo setfacl -Rm u:rocky:rx /usr/share/libvirt/iso
```

### 4. Autorisation du réseau Bridge pour l'utilisateur
Créez le dossier de configuration s'il n'existe pas :
```bash
sudo mkdir -p /etc/qemu-kvm
```

Ouvrez le fichier d'autorisation du bridge avec `vi` :
```bash
sudo vi /etc/qemu-kvm/bridge.conf
```
*(Dans `vi`, appuyez sur `i`, tapez la ligne `allow br0`, puis faites `Échap`, tapez `:wq` et `Entrée`).*

### 5. Validation de la configuration Rootless
**IMPORTANT :** Déconnectez-vous complètement de votre terminal (fermez la session SSH) et reconnectez-vous pour appliquer les changements de groupe.

Vérifiez ensuite que votre URI par défaut est bien passée en mode session sans utiliser `sudo` :
```bash
virsh uri
```

---

## Mémento : Commandes utiles de gestion des VMs

Pour lister l'intégralité de vos machines virtuelles (allumées comme éteintes) :
```bash
sudo virsh list --all
```

Pour forcer l'arrêt immédiat d'une machine virtuelle (équivalent d'un débranchement électrique) :
```bash
sudo virsh destroy debian-vnc
```

Pour supprimer définitivement une machine virtuelle et effacer son disque dur afin de libérer l'espace :
```bash
sudo virsh undefine debian-vnc --remove-all-storage
```