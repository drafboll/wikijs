---
title: CCNA S1
description: 
published: 1
date: 2026-09-25T15:41:54.447Z
tags: ccna, cisco, commande
editor: markdown
dateCreated: 2024-02-25T11:37:23.861Z
---

# Documentation de l'Infrastructure Réseau

## I. Topologie réseau :
 
Voici la topologie réseau choisi pour cette infrastructure ainsi que la segmentation réseau choisi:

L'entreprise a optimisé son plan d'adressage réseau en utilisant deux classes C pour répondre de manière efficace à ses besoins. La première classe, **192.168.0.0/24**, est subdivisée en deux sous-réseaux pour la ville de Paris, couvrant les plages d'adresses de 1 à 126 et de 129 à 254, avec un total de 200 utilisateurs. La deuxième classe, **192.168.1.0/24**, est utilisée pour les villes de Brest, Orléans et Mulhouse, avec des sous-réseaux adéquatement dimensionnés pour 100, 60, 25, 12 et 8 utilisateurs, respectivement, couvrant des plages d'adresses spécifiques pour chaque segment.

| Segment réseau | Plage d'adresses | Ville | Nombre d'utilisateurs |
| --- | --- | --- | --- |
| 192.168.0.0/25 | 1-126 | Paris | 100 |
| 192.168.0.128/25 | 129-254 | Paris | 100 |
| 192.168.1.0/25 | 1-126 | Brest | 100 |
| 192.168.1.128/26 | 129-190 | Brest | 60  |
| 192.168.1.192/27 | 193-222 | Orléans | 25  |
| 192.168.1.224/28 | 225-238 | Orléans | 12  |
| 192.168.1.240/28 | 241-254 | Mulhouse | 8   |

Ce choix, est justifié car nous répondons au plan d’adressage au plus proche du besoin de l’entreprise en utilisant deux classes C. 

Pour Paris :

- Sous-réseau 1 : 192.168.0.0/25 avec une plage d'adresses de 1 à 126 pour 100 utilisateurs.
- Sous-réseau 2 : 192.168.0.128/25 avec une plage d'adresses de 129 à 254 pour 100 autres utilisateurs.

Pour Brest :

- Sous-réseau 3 : 192.168.1.0/25 avec une plage d'adresses de 1 à 126 pour 100 utilisateurs.
- Sous-réseau 4 : 192.168.1.128/26 avec une plage d'adresses de 129 à 190 pour 60 utilisateurs.

Pour Orléans :

- Sous-réseau 5 : 192.168.1.192/27 avec une plage d'adresses de 193 à 222 pour 25 utilisateurs.
- Sous-réseau 6 : 192.168.1.224/28 avec une plage d'adresses de 225 à 238 pour 12 utilisateurs.

Pour Mulhouse :

- Sous-réseau 7 : 192.168.1.240/28 avec une plage d'adresses de 241 à 254 pour 8 utilisateurs.

Chaque sous-réseau est conçu pour être aussi proche que possible du nombre d'utilisateurs réels, minimisant ainsi le gaspillage d'adresses IP. Les deux réseaux de classe C (192.168.0.0/24 et 192.168.1.0/24) sont utilisés pour créer ces sous-réseaux, chacun adapté à la taille requise pour le nombre d'utilisateurs dans chaque ville.
    

## II. Configuration des routeurs

## A. Configuration du routeur Paris
Configuration hostname et de la page d'introduction du CLI


```
conf t
hostname RTR-PARIS
banner motd #
 _____           _
 |  __ \         (_)
 | |__) |_ _ __ _ ___
 |  ___/ _` | '__| / __|
 | |  | (_| | |  | \__ \
 |_|   \__,_|_|  |_|___/
#
```
Ensuite nous mettons en place les mots de passe
```
line console 0
password Respons11
login
exit
line vty 0 4
enable secret Respons11
```
Pour les interfaces fast ethernet
```
int fa0/1
ip address 192.168.0.126 255.255.255.128
description Connexion vers LAN Paris 100 users
no shutdown
exit
int fa0/0
ip address 192.168.0.254 255.255.255.128
description Connexion vers LAN Paris 100 users
no shutdown
```
Configuration interfaces Serial entre sites Routeur Paris
```
int s0/2/0
ip addr 1.1.1.22 255.255.255.252
description Connexion WAN to MULHOUSE

int s0/0/0
ip addr 1.1.1.14 255.255.255.252
description Connexion WAN to BREST

int s0/2/1
ip addr 1.1.1.9 255.255.255.252
description Connexion WAN to BREST

int s0/0/1
ip addr 1.1.1.6 255.255.255.252
description Connexion WAN to ORLEANS
```

Maintenant dans notre infrastructure réseau, nous déployons le protocole EIGRP, un protocole de routage dynamique avancé, pour assurer une convergence rapide et une gestion efficace des itinéraires au sein de notre environnement multi-sites, optimisant ainsi la performance et la fiabilité de nos communications inter routeur. Pour la configuration de ce routeur :
```
router eigrp 100
no auto-summary
netw 192.168.0.0 0.0.0.127
netw 192.168.0.128 0.0.0.127
netw 1.1.1.20 0.0.0.3
netw 1.1.1.12 0.0.0.3
netw 1.1.1.8 0.0.0.3
netw 1.1.1.4 0.0.0.3
do wr
```


## B. Configuration du routeur Orléans

Configuration hostname et de la page d'introduction du CLI


```
conf t
hostname RTR-ORLEANS
banner motd #
   ____       _
  / __ \     | |
 | |  | |_ __| | ___  __ _ _ __  ___
 | |  | | '__| |/ _ \/ _` | '_ \/ __|
 | |__| | |  | |  __/ (_| | | | \__ \
  \____/|_|  |_|\___|\__,_|_| |_|___/
#
```
Ensuite nous mettons en place les mots de passe
```
line console 0
password Respons11
login
exit
line vty 0 4
enable secret Respons11
```
Configuration interfaces fast ethernet entre sites Routeur Orleans
```
int fa0/0
ip address 192.168.1.222 255.255.255.224
description Connexion vers LAN Orleans 25 users
no shutdown
exit
int fa0/1
ip address 192.168.1.238 255.255.255.224
description Connexion vers LAN Orleans 12 users
no shutdown
```
Configuration interfaces Serial entre sites Routeur Orleans
```
int s0/0/1
ip addr 1.1.1.5 255.255.255.252
description Connexion WAN to PARIS

int s0/0/0
ip addr 1.1.1.2 255.255.255.252
description Connexion WAN to BREST
```
Pour la configuration du EIGRP :

```
router eigrp 100
no auto-summary
netw 1.1.1.0 0.0.0.3
netw 1.1.1.4 0.0.0.3
netw 192.168.1.192 0.0.0.31
netw 192.168.1.224 0.0.0.31
do wr
```
## C. Configuration du routeur Brest

Configuration hostname et de la page d'introduction du CLI


```
conf t
hostname RTR-BREST
banner motd #
  ____                _
 |  _ \              | |
 | |_) |_ __ ___  ___| |_
 |  _ <| '__/ _ \/ __| __|
 | |_) | | |  __/\__ \ |_
 |____/|_|  \___||___/\__|
#
```
Ensuite nous mettons en place les mots de passe
```
line console 0
password Respons11
login
exit
line vty 0 4
enable secret Respons11
```
Configuration interfaces fast ethernet entre sites Routeur Brest
```
int fa0/0
ip address 192.168.1.126 255.255.255.128
description Connexion vers LAN Brest 100 users
no shutdown
exit
int fa0/1
ip address 192.168.1.190 255.255.255.192
description Connexion vers LAN Brest 60 users
no shutdown
```
Configuration interfaces Serial entre sites Routeur Brest
```
int s0/0/1
ip addr 1.1.1.1 255.255.255.252
description Connexion WAN to ORLEANS

int s0/2/1
ip addr 1.1.1.10 255.255.255.252
description Connexion WAN to PARIS

int s0/0/0
ip addr 1.1.1.13 255.255.255.252
description Connexion WAN to PARIS

int s0/2/0
ip addr 1.1.1.17 255.255.255.252
description Connexion WAN to MULHOUSE
```
Pour la configuration du EIGRP :

```
router eigrp 100
no auto-summary
netw 1.1.1.16 0.0.0.3
netw 1.1.1.6 0.0.0.3
netw 1.1.1.12 0.0.0.3
netw 1.1.1.8 0.0.0.3
netw 1.1.1.0 0.0.0.3
netw 192.168.1.128 0.0.0.63
netw 192.168.1.0 0.0.0.128
do wr
```
## D. Configuration du routeur Mulhouse

Configuration hostname et de la page d'introduction du CLI


```
conf t
hostname RTR-MULHOUSE
banner motd #
  __  __       _ _
 |  \/  |     | | |
 | \  / |_   _| | |__   ___  _   _ ___  ___
 | |\/| | | | | | '_ \ / _ \| | | / __|/ _ \
 | |  | | |_| | | | | | (_) | |_| \__ \  __/
 |_|  |_|\__,_|_|_| |_|\___/ \__,_|___/\___|
#
```
Ensuite nous mettons en place les mots de passe
```
line console 0
password Respons11
login
exit
line vty 0 4
enable secret Respons11
```
Configuration interfaces fast ethernet entre sites Routeur Orleans
```
ip address 192.168.1.254 255.255.255.240
description Connexion vers LAN Mulhouse 8 users
no shutdown
```
Configuration interfaces Serial entre sites Routeur Orleans
```
int s0/0/0
ip addr 1.1.1.18 255.255.255.252
description Connexion WAN to BREST

int s0/0/1
ip addr 1.1.1.21 255.255.255.252
description Connexion WAN to PARIS
```
Pour ce routeur nous allons configurer un DHCP 
```
ip dhcp pool mulhouse
network 192.168.1.240 255.255.255.240
default-router 192.168.1.254
ip dhcp excluded-address 192.168.1.231
```

Pour la configuration du EIGRP :

```
router eigrp 100
no auto-summary
netw 1.1.1.0 0.0.0.3
netw 1.1.1.4 0.0.0.3
netw 192.168.1.192 0.0.0.31
netw 192.168.1.224 0.0.0.31
do wr
```

## III. Configuration du serveur
### A. Serveur DNS
Nous avons mis en place un serveur dns pointant vers l'adresse rtr-brest.lumina-tech.lan vers la patte du routeur en 192.168.1.190
![dns.png](/dns.png)
Il suffit juste de configurer les dhcp en mettant les lignes dans les pools adresses
```
RTR-MULHOUSE(dhcp-config)#dns 192.168.1.150
```
### Le serveur web
Un serveur web est hebergé sur le serveur 192.168.1.150, ce serveur est enregistré sur le serveur dns avec un record A lumina-tech.lan vers l'addresse 192.168.1.150.
![capture_d’écran_2024-02-23_131237.png](/capture_d’écran_2024-02-23_131237.png)
Ce serveur est donc joignable par l'ensemble des machines sur le réseau par son FQDN.
![http.png](/http.png)
### Le DHCP relay
Un serveur DHCP est activé sur le serveur 192.168.1.150, il distribue des adresses IP sur l'ensemble des réseaux de PARIS, BREST et Orleans en suivant le plan d'adressage de chacun d'entre eux.
![dhcp.png](/dhcp.png)
Il faut que toutes les interfaces sur lequelles les packets DHPCP transitent soient configurées avec un dhcp relay qui pointent vers l'adresse 192.168.1.150
```
conf t
int Fa0/0
ip helper-address 192.168.1.150
```

## IV. Le protocole EIGRP 
### A. Introduction
EIGRP (Enhanced Interior Gateway Routing Protocol) est un protocole de routage avancé développé par Cisco Systems. Bien qu'il ait été initialement propriétaire, Cisco a partiellement ouvert EIGRP en 2013, permettant une certaine interopérabilité avec des équipements d'autres fabricants. EIGRP est un protocole de routage dynamique utilisé dans les réseaux informatiques pour automatiser le routage des décisions et la configuration. Il est considéré comme un protocole de routage hybride car il possède des caractéristiques des protocoles de routage à vecteur de distance et à état de lien. Voici quelques aspects clé d'EIGRP :

1. Mécanisme de mise à jour : EIGRP utilise un mécanisme de mise à jour incrémentielle, où les modifications sont propagées uniquement lorsque le réseau change, plutôt que d'envoyer périodiquement des mises à jour complètes. Cela réduit la bande passante nécessaire pour les mises à jour de routage.

2. Tables utilisées par EIGRP :
- Table de voisinage : Contient des informations sur les routeurs voisins directement connectés.
- Table de topologie : Contient toutes les routes apprises de tous les voisins, y compris les chemins de secours.
- Table de routage : Contient les meilleures routes sélectionnées à partir de la table de topologie pour être utilisées dans le routage du trafic.

3. Métrique : EIGRP utilise une métrique composite basée sur plusieurs facteurs, tels que la bande passante, le délai, la charge et la fiabilité. La formule de la métrique peut être ajustée en fonction des besoins du réseau.

1. Équilibrage de charge : EIGRP est capable de réaliser un équilibrage de charge sur des chemins de coûts inégaux, une caractéristique unique parmi les protocoles de routage.

1. Convergence rapide : EIGRP offre une convergence rapide grâce à l'utilisation des successeurs et des successeurs faisables. Ces mécanismes permettent à EIGRP de trouver rapidement des routes de remplacement en cas de défaillance d'un lien.

1. Support du VLSM/CIDR : EIGRP prend en charge le Variable Length Subnet Masking (VLSM) et Classless Inter-Domain Routing (CIDR), ce qui le rend adapté aux environnements de réseaux modernes.

1. Authentification : EIGRP peut configurer l'authentification pour sécuriser les échanges d'informations de routage entre les routeurs.

1. Multicast et Unicast : EIGRP utilise le multicast pour envoyer la majorité de ses messages (adresse IP multicast 224.0.0.10), mais peut aussi utiliser unicast dans certaines situations.

1. Interopérabilité : Bien que EIGRP ait été rendu partiellement ouvert, son adoption en dehors des équipements Cisco reste limitée.

### B. Configuration sur les routeurs de notre maquette

#### Sur tout les routeurs

Nous avons configurer une variance 2 sur tous les routeurs :
Celle ci est utilisée pour implémenter ce qu'on appelle l'équilibrage de charge inégal. Par défaut, EIGRP effectue un équilibrage de charge sur les chemins qui ont exactement la même métrique (coût).

```
RTE-PARIS(config-router)# variance 2
```

#### Agrégation permettant de regrouper plusieurs routes

1. Réduction de la taille de la table de routage : En regroupant plusieurs routes en une seule entrée, l'agrégation diminue le nombre d'entrées dans les tables de routage des routeurs. Cela réduit la consommation de mémoire et améliore les performances du routeur.

2. Diminution du nombre de mises à jour de routage : L'agrégation peut réduire le nombre de mises à jour de routage nécessaires lorsque des changements se produisent dans le réseau. Cela est particulièrement bénéfique dans un protocole comme EIGRP, qui utilise des mises à jour diffusées pour informer les autres routeurs des changements de routage.

3. Amélioration de la stabilité du réseau : En masquant les changements mineurs dans la topologie du réseau (comme l'activation/désactivation des interfaces sur les routeurs individuels), l'agrégation peut rendre le réseau globalement plus stable et prévisible.

4. Simplification de la politique de routage : L'agrégation permet de simplifier les politiques de routage en réduisant le nombre de routes à gérer et à maintenir.

Nous avons donc configurer les routeurs Paris et Brest pour que les interfaces les reliant soient les prioritaires dans leur échange à l'aide d'une agrégation.

Pour BREST
```
int s0/0/0 (BREST) ip summary-addr ei 100 1.1.1.12 255.255.255.252
int s0/2/1 (BREST) ip summary-addr ei 100 1.1.1.8 255.255.255.252
```

Pour PARIS
```
int s0/0/0 (PARIS) ip summary-addr ei 100 1.1.1.12 255.255.255.252
int s0/2/1 (PARIS) ip summary-addr ei 100 1.1.1.8 255.255.255.252
```