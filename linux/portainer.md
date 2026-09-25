---
title: Installation de portainer
description: 
published: 1
date: 2026-09-25T15:47:47.742Z
tags: docker, docker hub, dockerhub, portainer
editor: markdown
dateCreated: 2023-02-16T09:14:26.535Z
---

# Installer et configurer portainer sur ubuntu 20.04
## I. installer Docker
### Introduction
Docker est une application qui simplifie le processus de gestion des processus d’application dans les conteneurs. Les conteneurs vous permettent d’exécuter vos applications dans des processus isolés des ressources. Ils sont similaires aux machines virtuelles, mais les conteneurs sont plus portables, plus respectueux des ressources et plus dépendants du système d’exploitation hôte.
 
Pour une introduction détaillée aux différents composants d’un conteneur Docker, consultez l’Écosystème Docker : Une introduction aux composants communs.

Dans ce tutoriel, vous allez installer et utiliser Docker Community Edition (CE) sur Ubuntu 20.04. Vous allez installer Docker lui-même, travailler avec des conteneurs et des images, et pousser une image vers un référentiel Docker.

### Installation de Docker
Le package d’installation Docker disponible dans le référentiel officiel Ubuntu peut ne pas être la dernière version. Pour être sûr de disposer de la dernière version, nous allons installer Docker à partir du référentiel officiel Docker. Pour ce faire, nous allons ajouter une nouvelle source de paquets, ajouter la clé GPG de Docker pour nous assurer que les téléchargements sont valables, puis nous installerons le paquet.

Tout d’abord, mettez à jour votre liste de packages existante :
```
sudo apt update
```
Ensuite, installez quelques paquets pré-requis qui permettent à apt d’utiliser les paquets sur HTTPS :
```
sudo apt install apt-transport-https ca-certificates curl software-properties-common
```
Ensuite, ajoutez la clé GPG du dépôt officiel de Docker à votre système :
```
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
```
Ajoutez le référentiel Docker aux sources APT :
```
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu focal stable"
```
Ensuite, mettez à jour la base de données des paquets avec les paquets Docker à partir du référentiel qui vient d’être ajouté :
```
sudo apt update
```
Assurez-vous que vous êtes sur le point d’installer à partir du dépôt Docker et non du dépôt Ubuntu par défaut :
```
apt-cache policy docker-ce
```
Vous verrez un résultat comme celui-ci, bien que le numéro de version du Docker puisse être différent :
```
docker-ce:
  Installed: (none)
  Candidate: 5:19.03.9~3-0~ubuntu-focal
  Version table:
     5:19.03.9~3-0~ubuntu-focal 500
        500 https://download.docker.com/linux/ubuntu focal/stable amd64 Packages
Notez que le docker-ce n’est pas installé, mais que le candidat à l’installation provient du dépôt Docker pour Ubuntu 20.04 (focal).
```
Enfin, installez Docker :
```
sudo apt install docker-ce
```
Le Docker devrait maintenant être installé, le démon démarré, et le processus autorisé à démarrer au boot. Vérifiez qu’il tourne :
```
sudo systemctl status docker
```
La sortie devrait être similaire à ce qui suit, montrant que le service est actif et en cours d’exécution :
```
Output
● docker.service - Docker Application Container Engine
     Loaded: loaded (/lib/systemd/system/docker.service; enabled; vendor preset: enabled)
     Active: active (running) since Tue 2020-05-19 17:00:41 UTC; 17s ago
TriggeredBy: ● docker.socket
       Docs: https://docs.docker.com
   Main PID: 24321 (dockerd)
      Tasks: 8
     Memory: 46.4M
     CGroup: /system.slice/docker.service
             └─24321 /usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock
```
L’installation de Docker vous donne maintenant non seulement le service Docker (démon) mais aussi l’utilitaire en ligne de commande docker, ou le client Docker. Nous allons voir comment utiliser la commande docker plus loin dans ce tutoriel.

> Pour information vous pouvez trouver des docker déja construit sur : https://hub.docker.com/
{.is-info}

## II. installer Docker Compose
Télécharger Docker Compose
Commencez par confirmer la version la plus récente de Docker Compose disponible dans leur page de versions (https://github.com/docker/compose/releases). Au moment de la rédaction du présent document, la version stable la plus récente est 1.26.0.

Exécutez la commande suivante pour télécharger Docker Compose et rendre ce logiciel globalement accessiblesur votre système en tant que docker-compose :
```
sudo curl -L "https://github.com/docker/compose/releases/download/1.26.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
```
### Configurer les autorisations exécutables
Ensuite, définissez les autorisations correctes pour vous assurer que la commande docker-compose est exécutable :
```
sudo chmod +x /usr/local/bin/docker-compose
```
Pour vérifier que l’installation a réussi, exécutez :
```
docker-compose --version
```
Vous verrez une sortie semblable à celle-ci :
```
Output
docker-compose version 1.26.0, build 8a1c60f6
Docker Compose est maintenant installé avec succès sur votre système.
```

## III. Installer portainer
Exécutez la commande suivante pour démarrer portainer dans un conteneur :
```
docker run -d -p 9000:9000 --name=portainer --restart=always -v /var/run/docker.sock:/var/run/docker.sock -v portainer_data:/data portainer/portainer-ce
```
> Voici le détail de chaque option :
{.is-info}


docker run est la commande de base pour démarrer un conteneur
- -d signifie detach. Il exécute le conteneur en arrière-plan.
- -p exposer l'interface utilisateur sur le port 9000, l'interface web de portainer sera à l'URL hostname:9000.
- --name=portainer nomme le conteneur portainer
- --restart=always redémarre toujours Portainer (après un crash ou au redémarrage de la machine ou du deamon)
- -v /var/run/docker.sock:/var/run/docker.sock monte le volume /var/run/docker.sock de la machine hôte vers le conteneur Portainer. /var/run/docker.sock est en fait le socket du démon Docker. Portainer a besoin de cette socket pour obtenir le statut de Docker.
- -v portainer_data:/data monte le volume créé précédemment (vous pouvez le mettre dans le volume que vous voulez)
- portainer/portainer-ce est le nom de l'image [Portainer Community Edition] (https://hub.docker.com/r/portainer/portainer-ce).



















