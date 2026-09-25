---
title: Installation complète de Rocky Linux + Docker + Docker Swarm (2 nœuds HA)
description: docker Swarm
published: 1
date: 2026-09-25T15:43:10.862Z
tags: 
editor: markdown
dateCreated: 2025-11-10T22:30:31.751Z
---

# Installation complète Rocky Linux + Docker + Docker Swarm (2 nœuds)
 
## Configuration réseau

| Machine | Hostname | IP | Rôle |
|----------|-----------|----|------|
| VM1 | SVR-DOCKER-01 | 192.168.40.30 | Manager |
| VM2 | SVR-DOCKER-02 | 192.168.40.40 | Worker |

---

## 1. Préparation de Rocky Linux

### Mise à jour et outils de base
```bash
sudo dnf update -y
sudo hostnamectl set-hostname SVR-DOCKER-01   ## ou SVR-DOCKER-02 selon la VM
sudo timedatectl set-timezone Europe/Paris
sudo dnf install -y vim curl wget git net-tools lsof unzip tar
Configuration réseau (adapter l’adresse selon la VM)
```

### Pour le manager :
```bash
sudo nmcli con mod "System eth0" ipv4.addresses 192.168.40.30/24
sudo nmcli con mod "System eth0" ipv4.gateway 192.168.40.1
sudo nmcli con mod "System eth0" ipv4.dns "192.168.20.10;192.168.20.20"
sudo nmcli con mod "System eth0" ipv4.method manual
sudo nmcli con up "System eth0"
```

### Pour le worker :
```bash
sudo nmcli con mod "System eth0" ipv4.addresses 192.168.40.40/24
sudo nmcli con mod "System eth0" ipv4.gateway 192.168.40.1
sudo nmcli con mod "System eth0" ipv4.dns "192.168.20.10;192.168.20.20"
sudo nmcli con mod "System eth0" ipv4.method manual
sudo nmcli con up "System eth0"
```

## 2. Installation de Docker et Docker Compose
Dépôt et installation
```bash
sudo dnf install -y dnf-utils device-mapper-persistent-data lvm2
sudo dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo dnf install -y docker-ce docker-ce-cli containerd.io
sudo systemctl enable --now docker
```

## Création d’un utilisateur dédié

```bash
sudo useradd -m -s /bin/bash docker
echo "docker:Respons11+" | sudo chpasswd
sudo usermod -aG wheel docker
sudo usermod -aG docker docker
```

## Installation de Docker Compose
```bash
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
docker-compose version
```
Test de Docker
```
sudo docker run hello-world
```

## 3. Création du cluster Docker Swarm

Initialisation sur le manager
```
sudo docker swarm init --advertise-addr 192.168.40.30
```
Copie la ligne affichée du type :

```
docker swarm join --token SWMTKN-1-xxxxxx 192.168.40.30:2377
```
Ajout du worker
Sur SVR-DOCKER-02 :

```
sudo docker swarm join --token SWMTKN-1-xxxxxx 192.168.40.30:2377
```
Vérification sur le manager
```
sudo docker node ls
```

Résultat attendu :

ID                            HOSTNAME         STATUS  AVAILABILITY  MANAGER STATUS
xxxxxx *                      SVR-DOCKER-01    Ready   Active        Leader
yyyyyy                        SVR-DOCKER-02    Ready   Active


## 4. Déploiement d’un service de test

Création d’un service NGINX répliqué
```
sudo docker service create --name web --replicas 2 -p 8080:80 nginx
sudo docker service ps web
```
Vérification de l’accès
Depuis ton navigateur :

http://192.168.40.30:8080
http://192.168.40.40:8080

## 5. Commandes utiles
```
## Lister les nœuds du cluster
docker node ls

## Lister les services
docker service ls

## Lister les conteneurs d’un service
docker service ps web

## Changer le nombre de réplicas
docker service scale web=4

## Supprimer un service
docker service rm web
```

## 6. Test de tolérance aux pannes
```
## Éteindre le worker
sudo shutdown now
```
Sur le manager :
```
docker service ps web
```
Les conteneurs du worker sont redéployés automatiquement sur le manager.

## 7. Ouverture des ports nécessaires (firewalld)
```
sudo systemctl enable --now firewalld
sudo firewall-cmd --add-port=22/tcp --permanent
sudo firewall-cmd --add-port=2377/tcp --permanent
sudo firewall-cmd --add-port=7946/tcp --permanent
sudo firewall-cmd --add-port=7946/udp --permanent
sudo firewall-cmd --add-port=4789/udp --permanent
sudo firewall-cmd --reload
```