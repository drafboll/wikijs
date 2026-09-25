---
title: Installation d'un truenas
description: 
published: 1
date: 2026-09-25T15:47:15.222Z
tags: 
editor: markdown
dateCreated: 2025-06-27T12:39:08.048Z
---

### Objectif

Configurer un environnement de centralisation de logs via **Graylog**, avec un **stockage distant iSCSI** hébergé sur **TrueNAS**, monté sur un hôte **ESXi02**.

---
 
### 1. Installation de TrueNAS SCALE

- Téléchargement de l’ISO depuis https://www.truenas.com/download-truenas-scale
- Création d’une VM dans **ESXi** ou installation physique
- Affectation d’une IP statique : `192.168.20.50`
- Ajout d’un disque **de 500 Go** pour les données iSCSI
- Nom de la machine : `SVR-NAS-01`

---

### 2. Création du stockage iSCSI (TrueNAS)

### ➔ 2.1 Créer un zvol de 500 Go

- Accéder à : **Stockage > Pools > Ajouter Zvol**
- Nom : `graylog_iscsi`
- Taille : `500 GiB`
- Compression activée ✅

### ➔ 2.2 Créer la cible iSCSI

- Aller dans : **Services > iSCSI > Cibles**
- Ajouter : `target-graylog`
- IP d’écoute : `0.0.0.0`
- Port : `3260`

### ➔ 2.3 Ajouter un extent de type Device

- Type : `Device`
- Disque : `/dev/zvol/tank/graylog_iscsi`
- Nom : `extent-graylog`

### ➔ 2.4 Associer l’extent à la cible

- `Services > iSCSI > Associated Targets`
- Lier `target-graylog` ↔ `extent-graylog`

### ➔ 2.5 Activer le service iSCSI

```bash
Services > iSCSI > Activer au démarrage

```

---

### 3. Connexion iSCSI sur le serveur Graylog (VM)

### ➔ 3.1 Installer les utilitaires iSCSI

```bash
sudo dnf install iscsi-initiator-utils -y

```

### ➔ 3.2 Découverte et connexion à la cible

```bash
sudo systemctl enable --now iscsid
sudo iscsiadm -m discovery -t sendtargets -p 192.168.20.50
sudo iscsiadm -m node -T iqn.2005-10.org.freenas.ctl:target-graylog -p 192.168.20.50 --login

```

---

### 4. Formatage et montage du disque iSCSI

```bash
sudo mkfs.ext4 /dev/sdX
sudo mkdir -p /mnt/graylog-data
sudo echo "/dev/sdX /mnt/graylog-data ext4 defaults 0 2" | sudo tee -a /etc/fstab
sudo mount -a

```

---

### 5. Schéma d’infrastructure

- ESXi02 : `192.168.10.111`
- TrueNAS : `192.168.20.50`
- Datastore iSCSI 500 Go : zvol
- VM Graylog connectée au disque iSCSI

---