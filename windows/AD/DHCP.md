---
title: Configurer le serveur DHCP sous Windows Server 2019
description: 
published: 1
date: 2026-09-25T15:52:17.124Z
tags: dhcp, windows seveur, étendue
editor: markdown
dateCreated: 2023-02-15T19:24:24.091Z
---

# Configurer le serveur DHCP sous Windows Server 2019
Pour la configuration, je vous propose de voir la création d'une étendue DHCP et d'une réservation d'adresse IP. Je vous invite à ouvrir la console "DHCP" qui se trouve dans les Outils d'administration de votre serveur.
## I. Créer une étendue DHCP
Une étendue DHCP va permettre de déclarer une plage d'adresses IP que le serveur DHCP peut distribuer aux postes clients qui se connecteront au réseau.

> Dans cet exemple, je vous rappelle que le serveur a l'adresse IP "192.168.136.0". Nous allons créer une étendue pour distribuer les adresses IP de 192.168.136.50 à 100.
{.is-info}
 

Il est important que la plage IP à distribuer soit sur le même segment réseau que le serveur pour notre test. Bien sûr, un serveur DHCP peut contenir plusieurs étendues et distribuer des adresses IP sur des réseaux différents du sien, mais ceci implique l'utilisation de la fonctionnalité relais DHCP. Cette fonctionnalité sera à configurer sur l'équipement qui effectue le routage entre vos réseaux (vos VLANs, par exemple), car les trames DHCP ne peuvent pas passer les routeurs par défaut.

Dans la console DHCP, effectuez un clic droit sur "IPv4" puis sur "Nouvelle étendue".
![nouvelle_etendu.png](/dhcp/nouvelle_etendu.png)
Nommez l'étendue, par exemple "LAN_Virtuel". Ce nom sera affiché dans la console DHCP. Poursuivez.
Désormais, il faut définir la plage d'adresses IP que l'on veut distribuer aux clients DHCP. Pour ma part, je vais définir une plage de 10 adresses sur le même réseau que celui sur lequel est connecté mon serveur DHCP. Il faut également spécifier le masque de sous-réseau adéquat.
![dhcp_addr_debut_fin.png](/dhcp/dhcp_addr_debut_fin.png)
> Pour la partie "Ajout d'exclusions et de retard", on peut l'utiliser pour exclure certaines adresses IP de la plage définie précédemment.
{.is-warning}


Imaginons qu'au sein de cette plage de 50 adresses, qui va de ".50" à ".100" on souhaite exclure l'adresse IP ".205". On peut imaginer que cette adresse IP en plein milieu de la plage est déjà utilisée par une imprimante, par exemple. Dans ce cas, il faudrait indiquer "192.168.1.205" comme adresse IP de début et "192.168.136.56" comme adresse IP de fin, puis cliquer sur "Ajouter". Ainsi de cette façon, l'adresse IP ne sera pas distribuée aux clients bien qu'elle soit dans la plage de notre étendue.

> La durée du bail correspond à la durée pendant laquelle le client pourra bénéficier de l'adresse IP fournie par le serveur DHCP.
{.is-info}


Lorsqu'il s'agit d'une étendue qui sera utilisée par les postes de votre établissement, vous pouvez utiliser une durée sur plusieurs jours, par exemple 8 jours. Pourquoi ? Simplement, car les postes de votre établissement seront amenés à se connecter régulièrement, voire même tous les jours, donc autant leur attribuer une adresse IP sur plusieurs jours pour éviter qu'il effectue une demande trop fréquemment auprès du serveur DHCP.

À l'inverse, si vous utilisez une étendue à destination d'un réseau Wi-Fi de type "Hotspot" où se connecteront des visiteurs, une durée de quelques heures ou une journée pour le bail DHCP est une bonne idée. En effet, un visiteur qui est là aujourd'hui ne le sera peut-être pas demain, alors pourquoi maintenir un bail DHCP sur cette adresse IP pendant plusieurs jours alors que l'appareil ne sera plus là ? Autant libérer l'adresse IP plus rapidement de manière à pouvoir la réattribuer à un autre appareil. Sinon, tout le temps que le bail est en cours, l'adresse IP sera bloquée même si l'appareil n'est plus connecté au réseau, à moins de supprimer soi-même le bail dans la base de données du serveur DHCP, mais ce n'est pas le but.

![durée_du_bail.png](/dhcp/durée_du_bail.png)
A l'étape suivante, sélectionnez "Oui, je veux configurer ces options maintenant" et poursuivez. Cela va permettre de définir des paramètres supplémentaires comme l'attribution d'une passerelle et d'un DNS.

Commençons par le routeur à attribuer aux clients DHCP de cette étendue, autrement dit la passerelle par défaut de votre réseau. Indiquez l'adresse IP et cliquez sur "Ajouter".

![passerelle_par_défault.png](/dhcp/passerelle_par_défault.png)
De la même façon pour l'étape "Nom de domaine et serveurs DNS", vous pouvez spécifier le nom de domaine Active Directory dans la zone "Domaine parent" s'il s'agit d'une étendue à destination des postes de votre entreprise. Ensuite, indiquez le(s) serveur(s) DHCP à distribuer à vos clients : là encore s'il s'agit d'une étendue pour vos postes intégrés au domaine, indiquez vos contrôleurs de domaine.
![dsn.png](/dhcp/dsn.png)
La résolution WINS étant obsolète, il n'est pas nécessaire de renseigner un serveur. Laissez vide et poursuivez.
Pour finir, cliquez sur "Oui, je veux activer cette étendue maintenant" et continuez jusqu'à la fin.

Notre étendue DHCP "LAN_Virtuel" apparaît bien dans la console DHCP et elle est active : à partir de ce moment-là, les postes clients peuvent obtenir une adresse IP à partir de notre serveur. Dans la section "Options d'étendue", on retrouve bien les options définies précédemment : le routeur, les DNS et le nom de domaine.

Il est à noter qu'il y a aussi la section "Options de serveur" : si une option DHCP est définie dans cette partie, elle sera héritée par l'ensemble des étendues automatiquement. Par contre, si une option DHCP est définie au niveau de l’étendue directement, dans "Options d'étendue", elle sera prioritaire vis-à-vis de la valeur dans "Options de serveurs".

Dans cette même console, la section "Baux d'adresses" permet de visualiser tous les baux d'adresses distribués à vos clients avec plusieurs informations : nom de l'hôte, fin du bail et l'adresse MAC de l'appareil. Pour information, un bail = des baux.

## II. Créer une réservation d'adresse IP
Il y a une autre section à connaître dans la console DHCP, c'est celle qui est nommée "Réservations". Elle va permettre de réserver l'utilisation d'une adresse IP à un appareil spécifique.

Prenons un exemple, vous avez une imprimante connectée au réseau et configurée en DHCP, pour faciliter le partage de cette imprimante avec les postes clients, ce serait mieux que son adresse IP soit toujours la même. Dans ce cas, il n'est pas obligatoire de configurer l'imprimante en IP fixe. On va choisir une adresse IP, récupérer l'adresse MAC de l'imprimante, dans le but de dire au serveur DHCP que cette IP est réservée à l'appareil qui a cette adresse MAC. Cela revient à faire ce que l'on appelle une réservation d'adresse IP par adresse MAC.

Une réservation se crée facilement, il suffit de faire un clic droit sur "Réservations" et de cliquer sur "Nouvelle réservation".


Ensuite, on donne un nom à la réservation, l'adresse IP à réserver et l'adresse MAC associée. Pour le format de l'adresse MAC, retirait tous les séparateurs habituels, notamment ":" et "-". Pour la prise en charge, choisissez "Les deux" ou "DHCP", la partie "BOOTP" étant un protocole d'amorçage pour faire démarrer des machines sur le réseau notamment pour le déploiement d'une image.


Cliquez sur "Ajouter" puis sur "Fermer" : le tour est joué. Maintenant, il est temps de tester notre serveur DHCP.

## III. Tester le serveur DHCP
Basculons sur la machine Windows 10 qui fait office de client DHCP pour valider le bon fonctionnement de notre serveur DHCP. Si l'on regarde la configuration réseau active sur la machine, on peut constater qu'elle a déjà reçu une adresse IP dans le cas où elle est configurée en DHCP (adressage dynamique).

L'adresse IP qu'elle a reçue correspond à la première adresse IP de notre étendue "LAN_Virtuel" : 192.168.136.50. Il y a une autre information très intéressante, au niveau de la valeur "Serveur DHCP IPv4" : le serveur DHCP qui a distribué l'adresse IP à notre machine est bien celui que l'on vient de configurer ! ?

Il y a deux commandes à connaître sous Windows pour gérer un bail DHCP. La première commande va permettre de libérer le bail DHCP au niveau du serveur DHCP, ce qui implique que le PC va perdre son adresse IP :
```
ipconfig /release
```
Ensuite, on peut effectuer une nouvelle demande d'adresse IP auprès du serveur DHCP grâce à cette seconde commande :
```
ipconfig /renew
```
Normalement, vous devez récupérer une adresse IP, en l'occurrence l'adresse IP "192.168.1.200".

Pour finir ce tutoriel, jetons un oeil aux logs générés par le serveur DHCP sous Windows.









