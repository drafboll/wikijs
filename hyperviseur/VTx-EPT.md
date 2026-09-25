---
title: Résoudre les conflits de Virtualisation VMware sur Windows 10/11 VTx-EPT
description: 
published: 1
date: 2026-09-25T15:42:26.822Z
tags: 
editor: markdown
dateCreated: 2026-04-29T08:02:51.244Z
---

# Guide Complet : Résoudre les conflits de Virtualisation VMware sur Windows 10/11
 
Ce guide regroupe toutes les solutions pour corriger les erreurs de type :
* *"Virtualized Intel VT-x/EPT is not supported on this platform"*
* *"VMware Workstation does not support nested virtualization"*
* *"Module 'HV' power on failed"*

---

## 1. La Solution "Compatibilité" (La plus rapide)
Si vous n'avez pas besoin de faire de la "virtualisation imbriquée" (installer Docker ou une autre VM *dans* votre VM), faites ceci :

1.  Fermez VMware.
2.  Allez dans les **Settings** (Paramètres) de votre VM.
3.  Onglet **Hardware** > **Processors**.
4.  **DÉCOCHEZ** la case : `Virtualize Intel VT-x/EPT or AMD-V/RVI`.
5.  **DÉCOCHEZ** également : `Virtualize CPU performance counters`.
6.  Validez et lancez la VM.

---

## 2. Désactivation des fonctionnalités Windows
Windows utilise un hyperviseur invisible pour sa propre sécurité qui "vole" l'accès au processeur.

1.  Appuyez sur `Win + R`, tapez `optionalfeatures.exe` et validez.
2.  **Décochez** absolument tout ce qui suit :
    * [ ] Hyper-V
    * [ ] Plateforme de l'hyperviseur Windows
    * [ ] Plateforme de machine virtuelle
    * [ ] Bac à sable Windows (Windows Sandbox)
    * [ ] Protection des applications Microsoft Defender (Application Guard)
3.  Cliquez sur OK et **Redémarrez**.

---

## 3. Désactiver la Sécurité Basée sur la Virtualisation (VBS)
Même si Hyper-V est décoché, Windows 11 laisse souvent l'isolation du noyau active.

1.  Allez dans **Paramètres** > **Confidentialité et sécurité** > **Sécurité Windows**.
2.  Cliquez sur **Sécurité de l'appareil** > **Détails de l'isolation du noyau**.
3.  Désactivez **Intégrité de la mémoire**.
4.  Redémarrez votre PC.

---

## 4. Commande de "Force" (Bcdedit)
Cette commande interdit à Windows de lancer son hyperviseur au boot.

1.  Faites un clic droit sur le bouton **Démarrer** > **Terminal (Admin)** ou **Invite de commandes (Admin)**.
2.  Tapez : `bcdedit /set hypervisorlaunchtype off`
3.  Appuyez sur Entrée.
4.  **Redémarrez complètement l'ordinateur.**

---

## 5. Nettoyage via le Registre (Credential Guard)
Sur les PC pro (Dell, HP, Lenovo), une sécurité appelée Credential Guard bloque souvent le VT-x.

1.  `Win + R`, tapez `regedit`.
2.  Allez à : `HKEY_LOCAL_MACHINE\System\CurrentControlSet\Control\Lsa`
3.  Créez une valeur **DWORD 32 bits** nommée `LsaCfgFlags` et mettez-la à `0`.
4.  Allez à : `HKEY_LOCAL_MACHINE\System\CurrentControlSet\Control\DeviceGuard`
5.  Créez une valeur **DWORD 32 bits** nommée `EnableVirtualizationBasedSecurity` et mettez-la à `0`.

---

## 6. Vérification du BIOS (Matériel)
Si rien ne fonctionne, le verrou est matériel.

1.  Entrez dans votre BIOS (F2 ou F12 sur Dell au démarrage).
2.  Cherchez l'onglet **Virtualization Support**.
3.  Vérifiez que ces options sont sur **ON** ou **Enabled** :
    * `Intel Virtualization Technology`
    * `VT for Direct I/O`
4.  Cherchez aussi **"SMM Security Mitigation"** dans la partie Security et essayez de la désactiver si le problème persiste.

---

## 7. Conflit Antivirus
Certains antivirus (Avast, AVG, Kaspersky) utilisent leur propre moteur de virtualisation.
1.  Allez dans les réglages de votre antivirus.
2.  Cherchez "Dépannage" ou "Défense matérielle".
3.  Décochez **"Utiliser la virtualisation matérielle pour renforcer la sécurité"**.

---

### Comment savoir si c'est réglé ?
Ouvrez l'application **Informations système** (msinfo32) :
* Regardez la toute dernière ligne : **Sécurité basée sur la virtualisation**.
* Si elle affiche **"Non activée"**, VMware fonctionnera à 100% de ses capacités avec toutes les options cochées.