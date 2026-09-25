---
title: Configuration d'un ADCS
description: 
published: 1
date: 2026-09-25T15:54:19.881Z
tags: 
editor: markdown
dateCreated: 2025-06-25T14:54:24.396Z
---

# Installation d'une Autorité de Certification Racine ADCS sur Windows Server

## Objectif
 
Mettre en place une autorité de certification d’entreprise racine (intégrée à l’Active Directory) sur un seul serveur.


## Étape 1 : Création du fichier CAPolicy.inf (préconfiguration)

vous pouvez créer un fichier nommé "CAPolicy.inf" (dans "C:\Windows") sur votre serveur ADCS. Vous pouvez éditer le fichier avec le Bloc-notes. Ce fichier sert à préconfigurer certaines options de l'autorité de certification que nous allons créer.

Emplacement : C:\Windows\CAPolicy.inf

Contenu à copier dans le fichier :
```
[Version]
Signature="$Windows NT$"
[Certsrv_Server]
RenewalKeyLength=4096
RenewalValidityPeriod=Years
RenewalValidityPeriodUnits=10
AlternateSignatureAlgorithm=0
CRLPeriod=Years
CRLPeriodUnits=1
```

💡 Ce fichier permet de configurer la durée de validité de la CA, la taille de clé, etc.

------------------------------------------------------------

## Étape 2 : Installation du rôle ADCS

→ Via l’interface graphique :
1. Ouvrir le Gestionnaire de serveur
2. Cliquer sur “Ajouter des rôles et fonctionnalités”
3. Sélectionner :
   - Services de certificats Active Directory
4. À l’étape des services de rôle, cocher uniquement :
   - Autorité de certification
5. Lancer l’installation, puis cliquer sur “Fermer”

------------------------------------------------------------

## Étape 3 : Configuration post-déploiement

1. Cliquer sur “Configurer les services de certificats Active Directory”
2. Assistant :
   - Rôle : Autorité de certification
   - Type : Autorité de certification d’entreprise
   - Niveau : Autorité de certification racine
   - Clé privée : Créer une nouvelle clé privée
   - Algorithme : SHA512
   - Taille de clé : 4096 bits
   - Nom de la CA : alpha-SVR-DC-01-CA
   - Durée de validité : 10 ans
   - Chemins de base : laisser par défaut
3. Valider et attendre la fin de la configuration

------------------------------------------------------------

## Étape 4 : Vérification

Ouvrir PowerShell et taper :

certutil -CAinfo
certsrv.msc

------------------------------------------------------------

## Étape 5 : Publication manuelle du certificat racine dans AD (optionnel)

certutil -dspublish -f "C:\Windows\System32\CertSrv\CertEnroll\alpha-SVR-DC-01-CA.crt" RootCA

------------------------------------------------------------

## Étape 6 : Vérifications dans Active Directory (ADSI Edit)

1. Lancer : adsiedit.msc
2. Se connecter au contexte “Configuration”
3. Naviguer ici :

Configuration > Services > Public Key Services > Certification Authorities

4. Vérifier que la CA est bien visible

------------------------------------------------------------

## Étape 7 : Nettoyage (optionnel)

- Le serveur SVR-DC-01 a été ajouté automatiquement au groupe “Cert Publishers”
  - Il peut être retiré de ce groupe après l’installation
- Supprimer le fichier CAPolicy.inf après l’installation pour éviter toute réutilisation accidentelle

------------------------------------------------------------

## Remarques de bonnes pratiques

- Pour plus de sécurité, il est recommandé de :
  - Créer une CA racine **autonome** (serveur hors domaine, éteint hors maintenance)
  - Déployer une CA **intermédiaire d’entreprise** (connectée au domaine) pour émettre les certificats
- Toujours effectuer des sauvegardes régulières de la base de la CA (`certutil -backup`)
- Documenter chaque étape pour les audits ou les plans de reprise


