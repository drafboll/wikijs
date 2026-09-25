---
title: Installer un serveur FOG
description: 
published: 1
date: 2026-09-25T15:48:54.007Z
tags: fog
editor: markdown
dateCreated: 2023-02-09T19:29:43.757Z
---

# Installation d'un serveur FOG
## I. Introduction
Fog Project est un logiciel sous Linux qui va vous permettre d’effectuer vos déploiements pour machines Windows, Linux, et Mac.

Le déploiement sur les clients se fera par le biais d’un serveur PXE fourni par Fog Project. Un client est également disponible pour déployer des applications sur des systèmes en cours d’exécution.
 
## II. Mise à jour des paquets système:
```
sudo apt update
sudo apt upgrade
```
## III. Installation
Le site Internet de Fog Project fournit une version .zip et .tar.gz. Il est possible d’effectuer l’installation depuis git, solution que j’ai utilisée.
```
sudo -i
```
```
cd /opt
apt-get install git
git clone https://github.com/FOGProject/fogproject.git
```
Il faudra ensuite lancer le script d’installation depuis le dossier bin.
```
cd  fogproject/bin
./installfog.sh
```
L’installation se déclenchera :
```
Installing LSB_Release as needed
 * Attempting to get release information.......................Done


   +------------------------------------------+
   |     ..#######:.    ..,#,..     .::##::.  |
   |.:######          .:;####:......;#;..     |
   |...##...        ...##;,;##::::.##...      |
   |   ,#          ...##.....##:::##     ..:: |
   |   ##    .::###,,##.   . ##.::#.:######::.|
   |...##:::###::....#. ..  .#...#. #...#:::. |
   |..:####:..    ..##......##::##  ..  #     |
   |    #  .      ...##:,;##;:::#: ... ##..   |
   |   .#  .       .:;####;::::.##:::;#:..    |
   |    #                     ..:;###..       |
   |                                          |
   +------------------------------------------+
   |      Free Computer Imaging Solution      |
   +------------------------------------------+
   |  Credits: http://fogproject.org/Credits  |
   |       http://fogproject.org/Credits      |
   |       Released under GPL Version 3       |
   +------------------------------------------+


   Version: 1.5.9 Installer/Updater

hostname: No address associated with hostname
```
Une première question vous sera posée concernant la distribution que vous utilisez (trouvée par défaut dans mon cas) :
```
What version of Linux would you like to run the installation for?

          1) Redhat Based Linux (Redhat, CentOS, Mageia)
          2) Debian Based Linux (Debian, Ubuntu, Kubuntu, Edubuntu)
          3) Arch Linux

  Choice: [2]2
```
La question suivante concernera le type d’installation : serveur normal ou serveur de stockage :
```
Starting Debian based Installation


  FOG Server installation modes:
      * Normal Server: (Choice N)
          This is the typical installation type and
          will install all FOG components for you on this
          machine.  Pick this option if you are unsure what to pick.

      * Storage Node: (Choice S)
          This install mode will only install the software required
          to make this server act as a node in a storage group

  More information:
     http://www.fogproject.org/wiki/index.php?title=InstallationModes

  What type of installation would you like to do? [N/s (Normal/Storage)]N
```
Vous seront ensuite proposés les réglages réseaux (à laisser par défaut en cas de présence d’une seule carte réseau) :
```
  We found the following interfaces on your system:
      * enp0s3 - 192.168.1.200/24

  Would you like to change the default network interface from enp0s3?
  If you are not sure, select No. [y/N]N
```
Les prochains réglages concerneront le réseau avec les réglages du serveur DHCP fourni par Fog Project :
```
Would you like to setup a router address for the DHCP server? [Y/n]Y
  What is the IP address to be used for the router on
      the DHCP server? [192.168.1.1]

  Would you like DHCP to handle DNS? [Y/n]n
  What DNS address should DHCP allow? [192.168.1.1]

  Would you like to use the FOG server for DHCP service? [y/N]y
```
> Il semblerait que ne pas activer le serveur DHCP de Fog Project empêche le boot PXE.
{.is-info}

Puis on règle l’internationalisation :
```
 This version of FOG has internationalization support, would
  you like to install the additional language packs? [y/N]y
```
> Pour avoir la possibilité de passer l’interface en français, il faudra répondre « yes »
{.is-info}

Il vous faudra ensuite choisir le support HTTPS ou non :
```
  Using encrypted connections is state of the art on the web and we
  encourage you to enable this for your FOG server. But using HTTPS
  has some implications within FOG, PXE and fog-client and you want
  to read https://wiki.fogproject.org/HTTPS before you decide!
  Would you like to enable secure HTTPS on your FOG server? [y/N]N
```
> Utiliser le mode HTTPS fera que l’installation sera plus lente. Il faut être patient, le script d’installation semblant bloqué. L’option d’accès à l’interface en mode https a été testé, pour les autres options que nous allons voir, celles-ci semblent ne pas fonctionner en mode HTTPS. Il est possible que le problème vienne des certificats autosignés.{.is-info}

Vous sera ensuite demandé le hostname à utiliser :
```
hostname: No address associated with hostname

  Which hostname would you like to use? Currently is:
  Note: This hostname will be in the certificate we generate for your
  FOG webserver. The hostname will only be used for this but won't be
  set as a local hostname on your server!
  Would you like to change it? If you are not sure, select No. [y/N] y
  Which hostname would you like to use? fogsrv
```
Vous aurez enfin une demande de confirmation regroupant vos choix 
Une fois l’installation terminée, il vous faudra faire une première connexion sur l’interface Web de Fog Project pour installer la base de données, vous avez les indications pour vous connecter sur la console :
```
 * You still need to install/update your database schema.
* This can be done 
by opening a web browser and going to:

   http://192.168.1.200/fog/management

* Press [Enter] key when database is updated/installed.
```
Avant d'appuyer sur enter vous devez vous rendre sur la page : http://192.168.1.200/fog/management
et cliquer sur install/Update NOW
![fog_install.png](/fog/fog_install.png)![login.png](/fog/login.png)
Vous pourrez alors appuyer sur la touche retour chariot dans la console d’installation, les services utilisés par Fog Project seront installés :
```
* Setup complete

   You can now login to the FOG Management Portal using
   the information listed below.  The login information
   is only if this is the first install.

   This can be done by opening a web browser and going to:

   http://192.168.1.200/fog/management

   Default User Information
   Username: fog
   Password: password

 * Changed configurations:

   The FOG installer changed configuration files and created the
   following backup files from your origional files:
   * /etc/dhcp/dhcpd.conf <=> /etc/dhcp/dhcpd.conf.1621875467
   * /etc/vsftpd.conf <=> /etc/vsftpd.conf.1621875467
* /etc/exports <=> /etc/exports.1621875467
```
À la fin de la phase de configuration, les informations de connexion, URL, login et mot de passe vous sont indiqués.

## IV. Accès à l'interphace manager
### Premier pas
Lors de l’accès à l’URL de connexion, vous aurez l’écran d’authentification :
![fog_acceuil.png](/fog/fog_acceuil.png)
> À ce stade, choisir la langue française n’aura aucun effet, nous verrons un peu plus tard comment changer la langue.
{.is-info}

![page_acceuil_interface.png](/fog/page_acceuil_interface.png)

Pour modifier les identifiants de connexions 
- aller dans users
- list all users
- cliquer sur l’utilisateur fog permettra de le modifier
- modifier le nom par ce que vous voulez et puis changer le mot de passe

### Changement de la langue
Nous allons commencer par passer l’interface en français.
Après avoir cliqué sur l’icône « Fog configuration », il va falloir aller dans l’onglet « FOG Settings » (en descendant dans la liste à gauche) sur l’écran de configuration 
Il faudra ensuite descendre dans la liste et cliquer sur « General Settings » pour dérouler les réglages. Vous pourrez ensuite sélectionner la langue en changeant la « default locale » 
Après avoir validé en cliquant sur « update » et rafraîchi l’écran en cliquant sur l’icône « dashboard », vous verrez l’interface en français.

> La traduction en français est partielle, certaines parties resteront en anglais.
{.is-info}


Pour les autres aspects, depuis l’écran de configuration, vous pourrez :

    affiner les réglages du serveur PXE ;
    affiner les réglages du client ;
    exporter la base de données pour effectuer une sauvegarde ;
    importer la base de données sauvegardée.

Nous n’entrerons pas dans le détail de ces réglages.

La configuration de base est stockée dans le fichier /opt/fog/.fogsettings, les autres réglages sont stockés dans la base de données MySQL nommée fog.

Vous pourrez faire un import/export de la base de données en sélectionnant « configuration save » dans la liste des choix dans l’onglet configuration.

## V. Configuration si on utilise un dchp ou par exemple ici pfsense
Lors de la configuration du dhcp vous devrez répondre autrement à cette question 
```
Would you like to setup a router address for the DHCP server? [Y/n]n
```
![dhcp_n.png](/fog/dhcp_n.png)
### Pour le DHCP sous windows serveur
Ensuite il faut faire les configuration suivante sur windows serveur ou bien pfsense
![fog_config_dhcp.png](/fog/fog_config_dhcp.png)
![66.png](/fog/66.png)
![67.png](/fog/67.png)

Vous aurez ensuite besoinn de configurer une option pour l'option UEFI (pour acceder à la console pour les windows par exemple)

Faire un clique droit sur IPv4 et cliquer sur Définir les classes des fournisseurs!
![classes_des_fournisseurs.png](/fog/dhcp/classes_des_fournisseurs.png)
Cliquer sur ajouter
![ajouter_1.png](/fog/dhcp/ajouter_1.png)
> NOTE: There are many other UEFI architectures besides just "PXEClient:Arch:00007".
{.is-warning}

- "PXEClient:Arch:00002" and "PXEClient:Arch:00006" both should get "i386-efi/ipxe.efi" as their option 067 boot file.
- "PXEClient:Arch:00008", "PXEClient:Arch:00009", and "PXEClient:Arch:00007" should get "ipxe.efi" as their option 067 boot file.
- "PXEClient:Arch:00007:UNDI:003016" should get "ipxe7156.efi" this file is specific to the Surface Pro 4.

remplir les champs demandé :
- display name : PXEClient (UEFI x64)
- Description : PXEClient:Arch:00007
- Binary : 50 58 45 43 6C 69 65 6E 74 3A 41 72 63 68 3A 30 30 30 30 37
![nouvelle_classe.png](/fog/dhcp/nouvelle_classe.png)

Puis fermer la fenetre 
Ensuite vous devez faire un click droit sur Stratégies et aller dans Nouvelle stratégie
![stratégie.png](/fog/dhcp/stratégie.png)
renseigner 
- Nom de la stratégie : UEFI
- Description : Stratégie pour UEFI based network booting
Puis suivant
Cliquer sur ajouter
et selectionner :
![ajouter_ou_modifier_une_condition.png](/fog/dhcp/ajouter_ou_modifier_une_condition.png)
suivant
![66.png](/fog/dhcp/66.png)
![67.png](/fog/dhcp/67.png)
Vous pouvez ensuite réaliser des enregistrement avec des machine windows.

### Pour le DHCP sous pfsense
ensuite finir la configuration et se rendre sur l'interface web de votre pfsense.
se rendre dans Services  -> serveur DHCP
![fog_dhcp_serveur.png](/fog/fog_dhcp_serveur.png)
Puis se rendre au niveau des configurations comme dans sur photo
![config_pfsense.png](/fog/config_pfsense.png)

## VI. Intégration d'un poste dans FOG
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

> Attention ! Pour vmware workstation lorsque vous voulez faire l'enregistrement d'un emachine windows 10, il faut changer un parametre dans les configuration en le faisant passer de UEFI en BIOS and le menu:
{.is-warning}

setting -> option -> advenced -> et dans frimware type cocher BIOS
![bios.png](/fog/bios.png)
Il faut que la machine soit éteinte pour le faire, et ensuite une fois l'action réalisé le remettre en UEFI
## VII. installation du client FOG sur un poste
### Client Windows
Prérequis pour les versions antérieures à Windows 10 : .net framework 4.5.1

Pour installer le client sur le poste, il faudra aller sur la page Web de connexion à Fog Project, vous trouverez le lien du client en bas de page :
![fog_acceuil.png](/fog/fog_acceuil.png)
ensuite
![client_fog.png](/fog/client_fog.png)
Une fois le client téléchargé, le seul point d’attention nécessaire lors de l’installation sera la page de configuration :
Vous pouvez voir que l’adresse du serveur par défaut est « fogserver » : à remplacer par son adresse IP, à moins que vous ayez créé une entrée pour celui-ci dans vos DNS.
![joindre_le_serveur_fog1.png](/fog/snap/joindre_le_serveur_fog1.png)
Pour le « web root », celui-ci sera à modifier si vous avez effectué des modifications sur le virtualhost créé pour Fog Project.
![agent_bien_installé.png](/fog/snap/agent_bien_installé.png)

### Client Linux

Fog Project dépend de Mono (l’implémentation libre de la plateforme .NET), qu’il faudra commencer par installer :
```
sudo apt-get install mono-complete
```
Vous pourrez ensuite télécharger le client proprement dit :
```
sudo wget http://[serveur fogproject]/fog/client/download.php?smartinstaller
```
Ceci va créer un fichier nommé « download.php?smartinstaller », que l’on renommera en smartinstaller.exe
```
sudo mv download.php?smartinstaller smartinstaller.exe
```

> Mono est l’implémentation libre de l’environnement d’exécution .Net Framework. Celui-ci permet de faire du développement cross-platform, d’où le renommage en fichier .exe
{.is-info}


Nous lancerons ensuite son installation par la commande mono smartinstaller.exe :
```
mono smartinstaller.exe
```
```
=


                     ..#######:.    ..,#,..     .::##::.
                .:######          .:;####:......;#;..
                ...##...        ...##;,;##::::.##...
                   ,#          ...##.....##:::##     ..::
                   ##    .::###,,##.   . ##.::#.:######::.
                ...##:::###::....#. ..  .#...#. #...#:::.
                ..:####:..    ..##......##::##  ..  #
                    #  .      ...##:,;##;:::#: ... ##..
                   .#  .       .:;####;::::.##:::;#:..
                    #                     ..:;###..

                ###########################################
                #     FOG                                 #
                #     Free Computer Imaging Solution      #
                #                                         #
                #     https://www.fogproject.org/         #
                #                                         #
                #     Credits:                            #
                #     https://fogproject.org/Credits      #
                #     GNU GPL Version 3                   #
                ###########################################
                #           FOG Service Installer         #

------------------------------------License-----------------------------------

FOG Service Copyright (C) 2014-2020 FOG Project
This program comes with ABSOLUTELY NO WARRANTY.
This is free software, and you are welcome to redistribute it under certain
conditions. See your FOG server under 'FOG Configuration' -> 'License' for
further information.

----------------------------------Information---------------------------------

Version.................................................................0.12.0
OS.......................................................................Linux
Current Path............................................................./root
Install Location............................................../opt/fog-service
Systemd...................................................................True
Initd.....................................................................True
```
Vous seront ensuite posées les questions suivantes :
```

-----------------------------------Configure----------------------------------

FOG Server address [default: fogserver]: 192.168.150.200
Webroot [default: /fog] :
Enable tray icon? [Y/n] : n
Start FOG Service when done? [Y/n] : Y
```
À la question « enable tray icon », il faudra répondre non, la fonctionnalité n’étant pas disponible sous Linux
Ci-dessous les logs d’installation :
```
----------------------------------Installing----------------------------------

Getting things ready....................................................[Pass]
Installing files........................................................[Pass]
Saving Configuration.................................................... 06/07/2021 08:45:49 Installer Settings successfully saved in /opt/fog-service/settings.json
[Pass]
Applying Configuration..................................................[Pass]
Pinning FOG Project..................................................... 06/07/2021 08:45:49 Installer FOG Project CA successfully installed
[Pass]
Pinning Server.......................................................... 06/07/2021 08:45:49 Data::RSA ERROR: FOG Server CA NOT found in keystore - needs to be installed
 06/07/2021 08:45:49 Middleware::Communication Download: http://192.168.1.200/fog/management/other/ca.cert.der
 06/07/2021 08:45:50 Data::RSA Injecting root CA:
 06/07/2021 08:45:50 Installer Successfully pinned server CA cert to CN=FOG Server CA
[Pass]

Starting FOG Service....................................................[Pass]

-----------------------------------Finished-----------------------------------

See /root/SmartInstaller.log for more information.
```
## VIII. Création d’un snapin
Un snapin correspond à une action de déploiement à effectuer sur les postes auxquels celui-ci sera affecté.
> Le snapin sera applicable à une machine ou à un groupe de machines.
{.is-info}

> Pour fonctionner, le snapin devra être entièrement automatique, et donc n’avoir aucune interaction. Sous Windows, il sera exécuté sous le compte SYSTEM. Ceci limite les possibilités et oblige à travailler ses installations.
{.is-warning}

Les snapins seront par défaut stockés dans /opt/fog/snapins.

Un snapin sera applicable à une machine en production ou exécutable après déploiement.
![snapping.png](/fog/snap/snapping.png)

Le snapin se présentera sous les formes suivantes :

- exécution d’un MSI ;
- script batch (pour machines Windows) ;
- script PowerShell pour Windows ou mono pour Linux ;
- VB script ;
- script Bash (pour machines Linux).

> Qu’est-ce qu’un fichier MSI ? MSI pour Microsoft System Installer. Windows installer est le moteur d’installation, de mise à jour et de désinstallation de logiciels fournis avec Windows.
{.is-info}
![création.png](/fog/snap/création.png)

Beaucoup de logiciels sous Windows sont fournis sous forme de fichier .msi. Un MSI devra être conçu pour vous permettre une installation silencieuse avec fourniture des paramètres d’installation sur la ligne de commande.

Pour modifier les paramètres par défaut d’un fichier .msi, ou si celui-ci ne permet pas de lui passer les paramètres nécessaires, vous pouvez utiliser Orca, fourni avec le SDK de Microsoft pour modifier celui-ci.

> La modification d’un msi peut être interdite ou soumise à condition par l’éditeur d’un produit.
{.is-warning}

### Test avec notpad sous windows
Nous allons tester l’installation de Firefox sur Windows :
Pour cela, il faudra récupérer le fichier .msi de Notpad sur le site de Notpad. Pour une installation simple, il suffira de créer un snapin avec le .msi de Notpad. Par défaut, l’option /quiet permettant l’installation silencieuse est sélectionnée :
![notpad_configuration_snapping.png](/fog/snap/notpad_configuration_snapping.png)
J’ai décoché l’option « redémarrage après installation », inutile pour notpad.
La commande générée sera :
```
msiexec.exe /i Notpad_Setup.msi /quiet
```
Cette commande sera visible dans la page de création du snapin, dans le champ « commande snapin ».
![snap_crée.png](/fog/snap/snap_crée.png)
Une fois le snapin créé, il va falloir l’affecter à une machine ou à un groupe de machines. Pour cela, il faut sélectionner celui-ci depuis la liste 

> Le numéro 1 visible après le nom de la tâche correspond au numéro de snapin, numéro autoincrémenté à chaque création de snapin.
{.is-info}


Une fois la tâche (le snapin) sélectionnée (en cliquant sur son nom), il faudra aller sur l’onglet membership
![ajouter_le_windows_10.png](/fog/snap/ajouter_le_windows_10.png)
Il faudra cocher la case « check to see what hosts can be added » et affecter les machines souhaitées.
![host.png](/fog/snap/host.png)
Dans l’exemple ci-dessous, la machine « win7 » a été intégrée
Restera ensuite à créer la tâche de déploiement depuis l’onglet « basic task » du groupe (ou de la machine), dans la partie avancée
![basic_task.png](/fog/arret-marche/basic_task.png)
il faut ensuite sélectionner « all snapins » pour déployer plusieurs actions ou « single snapin » pour un seul dans la liste des options

Dans la tâche, il faudra sélectionner le ou les snapin et choisir si le déploiement doit se faire immédiatement ou si celui-ci doit être déclenché à un moment précis
![wake_on_lan.png](/fog/snap/wake_on_lan.png)
Vous aurez ensuite confirmation de la création de la tâche

Vous pouvez voir l’état de la tâche en cliquant sur l’icône tâche en haut

En passant sur les icônes dans la colonne « status », vous aurez leur signification :

-  signifiant que la tâche est en file d’attente ;
-  l’icône signifiant que le traitement est en cours ;
-  l’icône indique qu’il s’agit d’un snapin.

> En cas de choix d’installation immédiate, il y aura quand même un petit délai avant exécution de la tâche sur la machine cliente.
{.is-warning}

Au moment du déclenchement de l’installation, sur un client Windows, vous aurez une notification :
Une nouvelle notification sera affichée une fois le snapin installé. Vous pouvez constater l’ajout d’une icône
![notepad_installé.png](/fog/snap/notepad_installé.png)
Dans l’interface Fog Project, vous pouvez voir la liste des snapins installés dans l’onglet « snapin history »

> Rappel : il sera impératif que l’installation soit silencieuse. Certains installeurs le permettent grâce à des paramètres de ligne de commande. Vous devrez rechercher pour chaque logiciel à installer les options pour un déploiement silencieux. En cas d’absence de celui-ci, vous ne pourrez pas le déployer sans utiliser des outils comme Msiexec pour les fichiers .msi ou sans repackager l’application pour que celle-ci soit silencieuse lors de l’installation.
{.is-warning}

### Installation de firefox sous linux

Le fonctionnement est similaire, quel que soit le gestionnaire de paquets.

Exemple pour l’installation de Firefox sur une Debian :
```
apt-get install firefox-esr -y
```
L’option -y servant à l’autoconfirmation
Firefox ESR est la version avec support à long terme (ESR=Extended Support Release)
sur base Redhat :
```
yum install firefox -y
```

## IX. Capture d'image
Prérequis
Installer un pc propre et fair el'enregistrement
![ubuntu_neuf.png](/fog/image/ubuntu_neuf.png)
![image_ubuntu.png](/fog/image/image_ubuntu.png)
Le prochain usage de Fog Project que nous allons étudier consistera à créer une image d’une machine, il s’agira d’une archive contenant la capture en l’état de la machine sélectionnée (machine éteinte, redémarrée en PXE).
Les images seront par défaut stockées dans /images/[nom de l'image]. Pour changer le chemin, il faudra aller dans fog configuration → fog settings → FOG Utils.
La première chose à faire sera de créer un conteneur nommé image dans la nomenclature Fog Project.
Pour cela, il faudra accéder à la partie « Image Management »
![image_create.png](/fog/image/image_create.png)
Entrer un nom pour l’image et la version de l’OS (si non détecté) suffira (au niveau des flèches bleues).
Il faudra ensuite aller sur l’hôte à capturer depuis la liste des hôtes 
![configuration_dans_create_image.png](/fog/image/configuration_dans_create_image.png)
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

### Personnalisation d'image windows 
Windows fournit un outil adapté à cela : sysprep.

Une fois la machine préparée à votre convenance (applications installées par exemple), il faudra lancer sysprep :
```
sysprep /generalize /oobe
```
> Il ne faudra pas redémarrer sur le master, mais en PXE pour effectuer la capture de l’image.
{.is-info}

L’outil sysprep peut également utiliser un fichier de réponse sous forme de XML, permettant la personnalisation de l’installation comme :
- le nom de l’ordinateur ;
- la clef d’installation Windows ;
- la timezone ;
- des commandes à lancer lors du premier boot ;
- …
Commande pour appeler sysprep avec un fichier de réponse :
```
C:\Windows\system32\sysprep\sysprep.exe /generalize /oobe /unattend:c:\sysprep.xml
```
exemple de fichier de réponse  :
```
<unattend xmlns="urn:schemas-microsoft-com:unattend" xmlns:wcm="http://schemas.microsoft.com/WMIConfig/2002/State">
 <settings pass="specialize">
  <component name="Microsoft-Windows-Shell-Setup" publicKeyToken="31bf3856ad364e35" 
     language="neutral" versionScope="nonSxS" processorArchitecture="x86">
   <ComputerName>JohnWayne</ComputerName>
   <ProductKey>AAAAA-AAAAA-AAAAA-AAAAA-AAAAA</ProductKey>
   <TimeZone>Central European Standard Time</TimeZone>
  </component>
 </settings>
 <settings pass="oobeSystem">
  <component name="Microsoft-Windows-Shell-Setup" processorArchitecture="x86" publicKeyToken="31bf3856ad364e35" 
     language="neutral" versionScope="nonSxS" processorArchitecture="x86">
   <FirstLogonCommands>
    <SynchronousCommand wcm:action="add">
     <Order>1</Order>
     <CommandLine>C:\runonce.cmd</CommandLine>
     <Description>RunOnce Command</Description>
    </SynchronousCommand>
   </FirstLogonCommands>
  </component>
 </settings>
</unattend>
```
Source : wikipédia.

### Personnalisation image linux

## X. Restauration d'image
Si vous souhaitez restaurer une image sur une machine existante, vous pouvez directement procéder à la restauration.
![enregistrement_ubuntu_restau.png](/fog/image/enregistrement_ubuntu_restau.png)
ensuite cliquer sur Deploy image
entrer les mot de passe et identifiant 
```
fog 
password 
```
> attention le clavier est en qwerty
{.is-warning}

![mdp.png](/fog/image/mdp.png)
Ensuite choissir l'image
![selectionner_l'image.png](/fog/image/selectionner_l'image.png)
![partclone.png](/fog/image/partclone.png)
![clone.png](/fog/image/clone.png)
La restauration c'est bien fini
![l'image_c'est_bien_cloné.png](/fog/image/l'image_c'est_bien_cloné.png)
La restauration d’une image personnalisée pourra être déployée sur plusieurs machines, ce qui correspond à une opération de déploiement.

## XI. Redémarrage, arrêt, ou réveil de poste à distance
Pour accéder à cette fonctionnalité, il vous faudra aller sur l’hôte concerné dans l’interface de Fog Project. Une fois celui-ci sélectionné, la fonctionnalité d’arrêt ou de redémarrage sera accessible depuis l’onglet « Power management ».

Vous pourrez choisir l’action :

- redémarrage
- arrêt
- wake-on-lan
![wake_on_lan_restart.png](/fog/arret-marche/wake_on_lan_restart.png)
Vous pourrez ensuite planifier ou lancer immédiatement l’opération :
![wake_on_lan_restart.png](/fog/arret-marche/wake_on_lan_restart.png)![redemarrer_maintenant.png](/fog/arret-marche/redemarrer_maintenant.png)
Sur les hôtes Windows, vous aurez une notification sur le poste client, avec possibilité de décaler celle-ci, ou de l’annuler
Sur un poste Linux, le redémarrage, une fois pris en compte par Fog Project, sera immédiat sans notification.
Le redémarrage ou arrêt est également applicable à un groupe.

> L’option sera effective depuis un compte sans droits administrateur.
{.is-info}

## XII. Suppression de mot de passe
La suppression de mot de passe sera accessible depuis l’écran « basic task » de la machine concernée :
![basic_task.png](/fog/arret-marche/basic_task.png)
![password_reset.png](/fog/arret-marche/password_reset.png)
Il faudra alors saisir le nom du compte ou le mot de passe doit être réinitialisé :
![confirmation.png](/fog/arret-marche/confirmation.png)
Si la machine est en route, cela déclenchera son redémarrage (avec la boite de dialogue permettant le report sous Windows)
tâche en cours 

> Le test de cette fonctionnalité n’a pas abouti. Un message d’erreur « enable to locate SAM file » s’affichant depuis le boot PXE (le fichier SAM est spécifique à Windows), que ce soit avec un client Windows ou un client Linux.
{.is-warning}

Le problème mentionné ci-dessus est contournable en créant un snapin réinitialisant le mot de passe par un script batch ou PowerShell, mais rendant du coup cette fonctionnalité inutile.

source de ce tuto : https://chrtophe.developpez.com/tutoriels/deploiement-fogproject/#L12-1



















