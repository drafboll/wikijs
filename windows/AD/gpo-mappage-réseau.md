---
title: Mappage lecteur réseau : GPO et Script
description: 
published: 1
date: 2026-09-25T15:53:46.704Z
tags: gpo, lecteur réseau, mappage, mappage réseau, windows serveur
editor: markdown
dateCreated: 2023-02-16T21:06:03.141Z
---

# Mappage lecteur réseau : GPO
Dans ce tutoriel, je vais vous présenter deux façons de mapper un lecteur réseau, par GPO et à l’aide d’un script qui doit être exécuté à l’ouverture de session, donc à l’aide d’une stratégie de groupe également.
 
À travers ce tutoriel, nous allons voir que le résultat est identique
## I. Prérequis
Être dans un environnement Active Directory.
Avoir un dossier partagé accessible aux utilisateurs à qui celui-ci va être mappé:
Partager un dossier par l’explorateur de fichier

Cette solution est souvent utilisée et marche pour l’ensemble des versions de Windows serveur et bureau.
Ouvrir l’explorateur et aller à l’emplacement du fichier à partager.
Faire un clic droit sur le dossier (ici Share)  et cliquer sur Propriétés
![créer_un_dossier.png](/mappage/créer_un_dossier.png)
![propriété.png](/mappage/propriété.png)
Aller sur l’onglet Partage puis cliquer sur Partage avancé
![partage_avancé.png](/mappage/partage_avancé.png)
Cocher la case Partager ce dossier et cliquer sur Autorisations
![partager_ce_dossier.png](/mappage/partager_ce_dossier.png)
Configurer les autorisations de partage en fonction des besoins puis cliquer sur Appliquer et OK
Cliquer de nouveau sur Appliquer et OK pour fermer la fenêtre Partage avancé
![tout_le_monde.png](/mappage/tout_le_monde.png)
Tester le partage directement depuis le serveur entrant l’adresse ci-dessus dans l’explorateur Windows
```
\\192.168.150.150\
```
## II. GPO – Stratégie de groupe
Ouvrir l’éditeur de stratégie de groupe sur un contrôleur de domaine

Créer une nouvelle stratégie, faire un clic droit sur le nom domaine ou sur une unité d’organisation et cliquer sur Créer un objet GPO dans ce domaine, et lier ici
![creer_une_gpo.png](/mappage/creer_une_gpo.png)
Donner un nom avec stratégie et cliquer sur OK
![lecteur_reseau.png](/mappage/lecteur_reseau.png)
 Faire un clic droit sur la stratégie et cliquer sur Modifier pour ouvrir l’éditeur.
![modifier.png](/mappage/modifier.png)
Aller sur Configuration utilisateur / Préférences / Paramètres Windows et double cliquer sur Mappages de lecteurs
![chemin_mappage.png](/mappage/chemin_mappage.png)
Faire un clic droit  Nouveau / Lecteur mappé
![nouveau_lecteur_mappé.png](/mappage/nouveau_lecteur_mappé.png)
Remplir le formulaire :

- Saisir l’emplacement du partage réseau
- Indiquer la lettre utilisée
- Appliquer
- OK
- pour libeller le lecteur réseau
![propriété_lecteur.png](/mappage/propriété_lecteur.png)
Le lecteur doit être visible dans Mappages de lecteurs.
![mappage_de_lecteur.png](/mappage/mappage_de_lecteur.png)






















