---
title: Déploiement de l'agent FusionInventory par GPO
description: 
published: 1
date: 2026-09-25T15:53:19.772Z
tags: fusion inventory, glpi, gpo
editor: markdown
dateCreated: 2023-02-16T21:08:11.588Z
---

# Déploiement de l'agent FusionInventory par GPO
## I. Introduction
Fusion Inventory est une application réputée pour faciliter l'inventaire d'un parc informatique, mais pour remonter les informations liées à vos postes de travail et serveurs, un agent Fusion Inventory doit être installé et configuré. Pour réaliser cette opération, vous pouvez l'inclure dans votre master de déploiement pour vos futurs postes ou mises à jour, mais si vous souhaitez l'installer sur un parc existant, le déploiement par GPO est possible (bien que ce ne soit pas la seule possibilité).
## II. Télécharger fusion inventory
Tout d'abord, on télécharge l'agent FusionInventory correspondant à notre version de GLPI et du plugin.
 
https://github.com/fusioninventory/fusioninventory-agent/releases/tag/2.6

Je prend la version x64, car tous les serveurs/ordinateurs sont en 64 bits.

## III. Placer l'exe dans le bon dossier
On le place dans C:\Share\GPO\GLPI
![dossier.png](/fusioninventory/dossier.png)
Penser à bien partager le dossier
https://wiki.alexandre-faye.fr/fr/windows/serveur/gpo-mappage-réseau

## IV. Création de la GPO
Ouvrez votre console de gestion des stratégies de groupe. Dépliez la partie Domaines, puis le nom de votre domaine jusqu’à voir l’OU sur laquelle sera appliquée la GPO par la suite.
![creer_une_gpo.png](/mappage/creer_une_gpo.png)
Donner un nom à cette nouvelle GPO et cliquez sur OK.
La GPO est désormais liée à l’OU souhaitée. Dans l’arborescence de gauche, cliquez sur votre GPO.
![modifier.png](/fusioninventory/modifier.png)
Dans la partie inférieure droite, on peut voir le filtrage de sécurité, c’est-à-dire précisément à qui ou à quoi va s’appliquer cette GPO. Par défaut, elle sera appliquée au groupe appelé « Utilisateurs authentifiés », c’est-à-dire à TOUT ce qui se trouve dans l’unité d’organisation à laquelle elle est liée uniquement !
Dans mon cas je l'ai mit au niveau du nom de domaine mais vous pouvez l'affecter seulement à une OU si vous ne voulez pas que tous le monde y accède.
A ce stade, la stratégie est encore vide pour le moment, ce qui est normal car nous n’avons encore rien mis dedans… Faites un clic droit sur le nom de votre GPO et cliquez sur « Modifier ».
![nom.png](/fusioninventory/nom.png)
Pour déployer un logiciel sur des machines via une stratégie de groupe, des paramètres sont déjà prévus mais nécessite que le logiciel à déployer soit au format MSI et des fois c’est un peu trop galère pour pas grand chose ce format, surtout quand on a besoin de donner des infos à l’exécutable (ce n’est que mon avis…).

Nous allons donc faire autrement en le lançant mais en tant que script de démarrage d’une machine ! Cela se situe dans Configuration Ordinateur > Stratégies > Paramètres Windows > Scripts (démarrage/arrêt) > Démarrage.
![démarrage.png](/fusioninventory/démarrage.png)
Une fois que vous êtes bien dans cette partie de l’éditeur, double-cliquez sur « Démarrage » dans la partie de droite et vous verrez la fenêtre suivante :
![ajouter.png](/fusioninventory/ajouter.png)
Cliquez sur le bouton « Ajouter ».

Il y a 2 parties à remplir ici :
- Le chemin réseau et le nom du script à exécuter, donc le EXE de l’agent fusion inventory dans notre cas,
- Les paramètres qu’on veut lui donner. Pour nous ça sera surtout des informations sur le mode d’installation et le chemin du serveur GLPI pour que le PC client envoie ses infos au bon endroit.

Dans la 1ère partie, il faut saisir le chemin du partage que vous avez créer précédemment (si vous ne vous en souvenez plus, allez vérifier directement…) suivi du nom du fichier exécutable. Soyez bien attentif à ce que les infos que vous saisissez soient bonnes et à bien mettre l’extension « .exe » du fichier !

> Attention de ne pas utiliser le bouton « Parcourir » qui peut vous induire en erreur si vous déclaré un chemin local de type « E:\Agent-FusionInv » avec qu’il faut un chemin réseau !
{.is-danger}

Dans mon cas, je vais saisir ceci en tant que « nom du script » :
```
\\windowsserveur\Share\GPO\GLPI\fusioninventory-agent-windows-x64_2.6.exe
```
Ensuite on va s’attaquer aux paramètres !
Il ne nous reste qu’à prendre ceux utiles à notre infrastructure. Moi c’est facile je vais juste en garder 5 que voici :

- /acceptlicense : pour accepter les termes et conditions d’utilisation du logiciel
- /runnow : pour lancer l’agent tout de suite après son installation
- /execmode=service : pour définir que l’agent sera exécuté en tant que service sur la machine cliente
- /server=’http://mon-serveur-glpi/plugins/fusioninventory/’ : pour déclarer le lien de mon serveur GLPI et le dossier qui contient le plugin
- /S : pour effectuer l’installation de façon silencieuse, c’est-à-dire sans que rien ne s’affiche à l’écran au cas où un utilisateur est connecté et en train de travailler

Dans la partie « Paramètres de scripts » de ma GPO, je vais donc écrire ceci (sur une seule ligne) :
```
/acceptlicense /runnow /execmode=service /server='http://192.168.150.170/glpi/plugins/fusioninventory/' /S
```
![édition_du_script.png](/fusioninventory/édition_du_script.png)
> Soyez bien attentif à la syntaxe, il y a un espace entre les différentes options mais pas avant ni après un égal, des guillemets simples autour de l’URL du serveur GLPI et bien entendu pensez à adapter cette URL à votre infrastructure (nom ou IP de votre serveur, nom de votre domaine si besoin, emplacement de votre dossier plugins…).
{.is-warning}

Quand c’est bon, cliquez sur OK deux fois et fermez l’éditeur de la stratégie de groupe. Dans la console de gestion des GPO, actualisez la vue, cliquez sur le nom de la GPO qu’on vient de modifier puis sur le haut à droite dans l’onglet Paramètres pour retrouver ce que l’on vient de faire.
![dernière_image.png](/fusioninventory/dernière_image.png)

faire un 
```
gpupdate /force
```
Pour faire une recherche de gpo
![connexion.png](/fusioninventory/connexion.png)

![vierge.png](/fusioninventory/vierge.png)
Il ne reste plus qu’à tester ! Pour cela, je passe sur mon PC windows 10 que je vais tout simplement redémarrer pour voir s’il se passe quelque chose.

Une fois la machine redémarrée, je me connecte dessus avec n’importe quel utilisateur, j’attends une toute petite minute (pour être vraiment large) et je vais voir si l’agent fusion inventory a été installé automatiquement (en allant fouiner dans C:\Programmes, en allant voir la liste des applications installées sur la machine, ou encore en ouvrant la console de gestion des services…).
![bien_installé.png](/fusioninventory/bien_installé.png)
![test_sur_glpi.png](/fusioninventory/test_sur_glpi.png)


























