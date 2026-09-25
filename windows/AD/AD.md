---
title: Configuration de l'AD et d'une forêt
description: Installer windows serveur 2019 et un domaine local
published: 1
date: 2026-09-25T15:52:07.810Z
tags: 2019, windows seveur
editor: markdown
dateCreated: 2023-02-01T09:17:40.399Z
---

# Windows serveur 2019
Voici les étapes pour configurer un Active Directory sur Windows Server 2019 avec une forêt nommée alexandre.local en utilisant
 
## I. Installer windows serveur 2019
### Préparer le support d'installation
1. Téléchargez l'ISO de Windows Server 2019 à partir du site web de Microsoft.
2. Gravez l'ISO sur un DVD ou créez une clé USB bootable avec un utilitaire comme Rufus.

### Configurer le BIOS de l'ordinateur
1. Démarrez l'ordinateur et accédez au BIOS.
2. Modifiez la séquence de démarrage pour démarrer à partir du DVD ou de la clé USB, en fonction de ce que vous avez choisi.
3. Enregistrez les modifications et quittez le BIOS.

### Installer Windows Server 2019
1. Insérez le DVD ou la clé USB dans l'ordinateur et redémarrez l'ordinateur.
2. Suivez les instructions à l'écran pour installer Windows Server 2019.
3. Lorsque vous y êtes invité, sélectionnez l'emplacement où vous souhaitez installer Windows Server 2019.
4. Sélectionnez les fonctionnalités que vous souhaitez installer.
5. Configurez les paramètres de langue et de clavier.
6. Entrez la clé de produit de Windows Server 2019 lorsque vous y êtes invité.
7. Suivez les instructions à l'écran pour terminer l'installation.

Et voilà, vous avez maintenant installé Windows Server 2019 ! N'oubliez pas de prendre les mesures de sécurité nécessaires et de configurer correctement votre serveur.

## II. Installation de l'AD et d'une forêt nomé alexandre.local
1. Installer les rôles de services Active Directory sur le serveur.
2. Lancez le Gestionnaire de serveur et cliquez sur Ajouter des rôles et des fonctionnalités.
3. Sélectionnez le rôle Serveur Active Directory et cliquez sur Suivant.
4. Sélectionnez les services Fonctionnalités de domaine et cliquez sur Suivant.
5. Cliquez sur Installer pour démarrer l'installation.
6. Redémarrez le serveur lorsque l'installation est terminée.
7. Cliquez sur Promouvoir ce serveur en contrôleur de domaine.
8. Sélectionnez la option "Créer une nouvelle forêt" et entrez alexandre.local en tant que nom de domaine racine.
9. Entrez un mot de passe pour le compte Administrateur de la forêt.
10. Suivez les instructions à l'écran pour terminer la promotion en contrôleur de domaine.