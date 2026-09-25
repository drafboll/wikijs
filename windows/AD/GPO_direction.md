---
title: Création d'un partage réseau par direction
description: 
published: 1
date: 2026-09-25T15:53:05.332Z
tags: 
editor: markdown
dateCreated: 2025-07-04T12:15:37.886Z
---

# Tutoriel complet : Création d'un partage réseau par service avec Active Directory

## Objectif
 
Mettre en place un partage réseau `\\SVR-FIC-01\Repertoire_Services` avec une arborescence par service (COM, DSI, RH), sécurisé avec des groupes AD, et mappé automatiquement sur le lecteur `K:` grâce à une GPO.

---

## Création des groupes dans Active Directory

Ouvrir **Utilisateurs et ordinateurs Active Directory**, dans l’UO :
`alpha.lan > Ressources > Fichiers > K`.

### Groupes globaux (GG_)

| Nom                 | Portée  | Rôle                      |
|---------------------|---------|---------------------------|
| GG_K_Lecture_COM    | Global  | Lecture pour COM          |
| GG_K_Ecriture_COM   | Global  | Écriture pour COM         |
| GG_K_Lecture_DSI    | Global  | Lecture pour DSI          |
| GG_K_Ecriture_DSI   | Global  | Écriture pour DSI         |
| GG_K_Lecture_RH     | Global  | Lecture pour RH           |
| GG_K_Ecriture_RH    | Global  | Écriture pour RH          |

### Groupe local de domaine (GDL_)

| Nom             | Portée        | Rôle                                |
|------------------|---------------|-------------------------------------|
| GDL_K_Acces      | Domaine local | Point d’entrée pour la GPO réseau  |

 **Ajouter tous les groupes GG_ dans GDL_K_Acces.**
![capture_d'écran_2025-07-04_141608.png](/service_k/capture_d'écran_2025-07-04_141608.png)
![capture_d'écran_2025-07-04_141650.png](/service_k/capture_d'écran_2025-07-04_141650.png)

---

## Création de la structure de dossiers sur le serveur

Sur le serveur `SVR-FIC-01`, créer la structure :

D:\Repertoire_Services
├── COM
├── DSI
└── RH

![capture_d'écran_2025-07-04_142130.png](/service_k/capture_d'écran_2025-07-04_142130.png)



## Configuration du partage réseau

### Partage du dossier `Repertoire_Services`

1. Clic droit > **Propriétés** > onglet **Partage**
2. Cliquer sur **Partage avancé**
3. Cocher **Partager ce dossier**
4. Nom du partage : `Repertoire_Services`
5. Cliquer sur **Autorisations** et ajouter :

| Groupe                 | Autorisation |
|------------------------|--------------|
| GDL_K_Acces            | Lecture      |
| GG_T1_ADM_SERVEUR      | Contrôle total |
| Admins du domaine      | Contrôle total |

 **Attention** : Ces droits sont de base. La vraie sécurité se fera avec les permissions NTFS.

![capture_d'écran_2025-07-04_142235.png](/service_k/capture_d'écran_2025-07-04_142235.png)

## Configuration des droits NTFS

Pour chaque dossier de service (`COM`, `DSI`, `RH`) :

1. Clic droit > **Propriétés** > **Sécurité** > **Avancé**
2. Cliquer sur **Désactiver l’héritage**, puis **Supprimer les entrées héritées**
3. Ajouter les autorisations suivantes (exemple pour `COM`) :

| Groupe                 | Accès                  | Appliquer à                       |
|------------------------|------------------------|-----------------------------------|
| GG_K_Lecture_COM       | Lecture/Exécution      | Ce dossier, sous-dossiers, fichiers |
| GG_K_Ecriture_COM      | Écriture personnalisée | Ce dossier, sous-dossiers, fichiers |
| CREATOR OWNER          | Contrôle total         | Ce dossier, sous-dossiers, fichiers |
| Administrateurs        | Contrôle total         | Ce dossier, sous-dossiers, fichiers |
| SYSTEM                 | Contrôle total         | Ce dossier, sous-dossiers, fichiers |

![capture_d'écran_2025-07-04_142412.png](/service_k/capture_d'écran_2025-07-04_142412.png)

### Détails pour les droits du groupe `GG_K_Ecriture_COM`

Autoriser :
- Création fichiers/dossiers
- Lecture des données
- Écriture d’attributs
- Suppression de sous-dossiers (mais pas des fichiers des autres)

Ne pas autoriser :
- Suppression
- Contrôle total
![capture_d'écran_2025-07-04_142425.png](/service_k/capture_d'écran_2025-07-04_142425.png)

 Répéter ces étapes pour `DSI` et `RH` avec les groupes correspondants.

---

## Mise en place de la GPO pour mapper le lecteur K:

### Créer la GPO

1. Ouvrir **GPMC** sur un DC
2. Créer une GPO nommée : `K_Connecter_Lecteur_Reseau`
3. Lier cette GPO à l’UO des utilisateurs ciblés

### Configurer le mappage réseau

Dans la GPO :

Configuration utilisateur > Préférences > Paramètres Windows > Mappage de lecteurs

![capture_d'écran_2025-07-04_142559.png](/service_k/capture_d'écran_2025-07-04_142559.png)
- **Action** : Mettre à jour
- **Lettre** : K
- **Chemin** : `\\SVR-FIC-01\Repertoire_Services`
- **Cocher** : Utiliser le premier disponible
- Ne pas masquer le lecteur

![capture_d'écran_2025-07-04_142623.png](/service_k/capture_d'écran_2025-07-04_142623.png)

### Ciblage de sécurité

Dans l’onglet **Ciblage** :
- Ajouter une condition de groupe de sécurité
- Nom du groupe : `GDL_K_Acces`
![capture_d'écran_2025-07-04_142651.png](/service_k/capture_d'écran_2025-07-04_142651.png)
---

## Test utilisateur

1. Connectez-vous avec un utilisateur membre de `GG_K_Ecriture_DSI` (ex: `alexandre.faye`)
2. Lancer la commande suivante :
   ```bash
   gpupdate /force

![capture_d'écran_2025-07-04_142749.png](/service_k/capture_d'écran_2025-07-04_142749.png)
