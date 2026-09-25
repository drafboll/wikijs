---
title: Fiche E5
description: 
published: 1
date: 2026-09-25T15:46:11.450Z
tags: 
editor: markdown
dateCreated: 2023-02-24T07:56:46.053Z
---

# Présentation pour la fiche E5

## Sujet  FOG
 
## 1.1 Intégration d'un poste dans FOG
Nous démarrons un poste client sous Windows grâce à un boot réseau PXE :
![pxe.png](/fog/pxe.png)
Nous aurons le menu suivant :
![interphace_fog_apres_pxe.png](/fog/interphace_fog_apres_pxe.png)
Il y a deux options d’enregistrement :

Quick registration and inventory ;
Perform Full Host Registration and Inventory.
Nous utiliserons dans un premier temps la première option, les différences seront expliquées un peu plus bas.

Une fois « quick registration and inventory » (que l’on peut traduire par enregistrement et inventaire rapide) sélectionné, la machine démarrera sur le système fourni par FOG.

Dans le cadre d’un enregistrement avec l’option « Perform Full Host Registration and Inventory », il vous sera demandé :

- l’« image id » lié à la machine ;
- le choix d’affectation à un groupe (Y/N) ;
- l’association à un snapin (y/N) ;
- l’association d’une product key (y/N) ;
- la jonction à un domaine (y/N) ;
- le nom de l’utilisateur principal sur l’ordinateur ;
- le choix de déploiement d’une image.
Image du boot en cours :
![enregistration.png](/fog/enregistration.png)
Une fois l’enregistrement effectué, la machine apparaîtra dans la liste des hôtes avec comme nom son adresse MAC ou le nom que vous lui avez donné :
![liste_des_hosts.png](/fog/liste_des_hosts.png)


## 1.2 création d'une image
Les images seront par défaut stockées dans /images/[nom de l'image].

La première chose à faire sera de créer un conteneur nommé image dans la nomenclature Fog Project.
Pour cela, il faudra accéder à la partie « Image Management »
![image_create.png](/fog/image/image_create.png)
Entrer un nom pour l’image et la version de l’OS (si non détecté) suffira (au niveau des flèches bleues).
![création_de_l'image.png](/e5/création_de_l'image.png)

## 1.3 choisir le type d'image
selcetion l'os correspondant
![opérating_system.png](/e5/opérating_system.png)
Puis sur quel disque
![single_disk_partition.png](/e5/single_disk_partition.png)
Ensuite la partition
![partition.png](/e5/partition.png)

## 1.4 associer l'image
Il faudra ensuite aller sur l’hôte à capturer depuis la liste des hôtes 
Cliquez sur l’icône |-> pour créer la tâche. Ceci déclenchera l’ouverture des paramètres de la machine, où il vous faudra sélectionner l’image (le conteneur) créée précédemment
!![fleche_orange.png](/fog/image/fleche_orange.png)
Une fois l’image créée, le nom de celle-ci apparaîtra à côté de l’icône permettant de déclencher la capture
![image_config.png](/fog/image/image_config.png)
![image_assigné.png](/fog/image/image_assigné.png)
Lors d’une prochaine capture, vous aurez l’écran suivant. Il faudra relancer la procédure pour arriver sur l’écran de création de la tâche dans le cas où vous venez de créer l’image comme vu ci-dessus.
![fleche_orange.png](/fog/image/fleche_orange.png)
![wake_on_lan.png](/fog/image/wake_on_lan.png)
Nous créons la tâche immédiatement, vous aurez un message comme quoi la tâche a été créée

vous pourrez ensuite gérer et suivre l’état depuis la liste des tâches :
![regarder_l'avancement.png](/fog/image/regarder_l'avancement.png)
Au démarrage de la machine à sauvegarder, celle-ci démarre en réseau sur l’image disque Fog Project
![pxe.png](/fog/pxe.png)
ci-dessous, sauvegarde en cours avec partclone, utilisé par Fog Project :
![partclone.png](/fog/image/partclone.png)
état de capture en cours depuis l’interface :
![avancement_en_cours.png](/fog/image/avancement_en_cours.png)
La barre de progression indique l’état d’avancement de la capture.
![clone.png](/fog/image/clone.png)
ici nous pouvons voir que fog est entrain de cloner ubuntu client

Une fois la capture effectuée, vous pourrez voir les informations afférentes (comme la taille, la date de capture) depuis la liste des images :
![image_situation.png](/fog/image/image_situation.png)
> La prise d’image pourra vous servir de backup, mais qui ne sera pas incrémental. Utiliser une image en tant que backup n’est pas le but premier de Fog Project.
{.is-warning}
![image_bine_déployé.png](/e5/image_bine_déployé.png)
## 1.5
Pour changer le chemin il faut aller dans 
```
nano /opt/fog/.fogsettings
```
![image_directory.png](/e5/image_directory.png)
ensuite refaire la meme procédure qu'avant sur une nouvelle image 
![affectation_image_à_ubuntu_e5.png](/e5/affectation_image_à_ubuntu_e5.png)
et voir si il c'est bien copier dans le bon dossier
![nouveau_dossier.png](/e5/nouveau_dossier.png)
```
sudo systemctl restart FOGImageReplicator.service FOGSnap inReplicator.service FOGImageSize.service FOGMulticastManager.service
```


## 1.6
D'après Wikipedia :
« Le multicast est une forme de diffusion d'un émetteur vers un groupe. Il est plus efficace que l'unicast pour diffuser des contenus vers une large audience. En unicast, on enverrait l'information autant de fois qu'il y a de connexions d'où un gaspillage de temps et de ressource du serveur. En multicast, chaque paquet n'est émis qu'une seule fois et routé vers toutes les machines du groupe de diffusion sans que le contenu ne soit dupliqué sur une quelconque ligne physique ; c'est donc le réseau qui se charge de reproduire les données…»

Pour se faire enregistrer tous les postes voulu
puis crée un groupe 
![nouveau_groupe.png](/e5/nouveau_groupe.png)
![groupe_manager.png](/e5/groupe_manager.png)

sélection l'image au groupe
![associer_l'image.png](/e5/associer_l'image.png)
Aller dans task basic et multi cast
![multicast.png](/e5/multicast.png)
selectionné pour que le multicast se lancer dans host 
![multicast_2.png](/e5/multicast_2.png)

![multicast_3.png](/e5/multicast_3.png)
ensuite lancé la tacche 
![multicast_4.png](/e5/multicast_4.png)
il ne reste plus qu'a regarder comment faire le status 










