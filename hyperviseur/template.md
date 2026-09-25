---
title: Créer des templates de VM avec VMware vCenter Server Appliance
description: 
published: 1
date: 2026-09-25T15:42:07.567Z
tags: 
editor: markdown
dateCreated: 2025-06-18T08:55:08.979Z
---

# Créer des templates de VM avec VMware vCenter Server Appliance (Windows Server 2022)


## I. Présentation
 
Ce guide présente la création d’un template (modèle) de VM sous **VMware vCenter** à partir d’une installation de **Windows Server 2022**, pour automatiser et standardiser les déploiements de serveurs dans un environnement vSphere.

Un template est une image figée d’une VM. Elle ne peut pas être démarrée, ce qui évite les modifications accidentelles. Il est possible d’y associer une **spécification de personnalisation** pour configurer automatiquement le nom de machine, l’adresse IP, l’utilisateur administrateur, etc.

---

## II. Modèle VS clone VS OVA/OVF

### A. Clone vs Modèle

* **Clone** : copie exacte d’une VM à un instant T
* **Modèle** : image figée, non modifiable sans conversion

### B. Modèle vs OVA/OVF

* **OVF** : format ouvert, multi-fichier
* **OVA** : archive unique compressée

Les templates sont gérés uniquement via vCenter et ne sont pas destinés à l’export vers d'autres organisations.

---

## III. Environnement de mise en place

* **vCenter Server Appliance (VCSA)** avec cluster ESXi
* **VM Windows Server 2022** propre, avec VMware Tools installés
* Configuration réseau statique, Windows à jour, nommage conforme
* (Optionnel) Préinstallation de logiciels selon les besoins

---

## IV. Création du modèle

1. Sélectionner la VM Windows Server 2022 prête à être utilisée comme référence
2. Clic droit > **Clone** > **Cloner vers un modèle**
3. Donner un nom clair au template, ex : `WinSrv-2022-Template`
4. Sélectionner le cluster/hôte et le datastore cible
5. Confirmer et lancer le clonage

> 🎯 Le modèle est désormais figé et stocké dans l'inventaire Templates de vCenter.

---

## V. Création d'une spécification de personnalisation d'invité VM

1. Aller dans **Menu > Stratégies et profils > Spécifications de personnalisation**
2. Cliquer sur **Nouveau**
![startégie.png](/template/startégie.png)

### Étapes principales :

* **OS cible** : Windows
* **Utilisation de Sysprep** : non nécessaire
* **Nom du propriétaire** : ex. IT-Admin
* **Nom du compte** : Admin local de la VM
* **Nom d’hôte** : Entrer un nom dans l'assistance de clonage/deployement
* **Clé de licence Windows** : facultatif
* **Mot de passe administrateur** : définir et confirmer
* **Fuseau horaire** : `Romance Standard Time` (Europe/Paris)
* **Script post-install** : non utilisé chez nous
* **Réseau** : DHCP ou IP manuelle lors du déploiement
* **Domaine/Workgroup** : Workgroup ou intégration AD lors du déploiement
![workgroup.png](/template/workgroup.png)

Valider la spécification. Elle sera réutilisable pour tous les futurs déploiements de ce modèle.

---

## VI. Déploiement d'une machine virtuelle à partir du modèle

1. Aller dans **Menu > VMs and Templates**
2. Clic droit sur le template > **Déployer une VM depuis ce modèle**
3. Donner un nom à la nouvelle VM, ex : `SVR-ADCS-01`
4. Sélectionner hôte et datastore
5. Activer les options suivantes :
![personalisertmplt.png](/template/personalisertmplt.png)
   * ✅ Personnaliser l’OS avec le fichier créé
   * ✅ Personnaliser le matériel (CPU, RAM, stockage...)
   * ✅ Allumer la VM après déploiement
![templateselect.png](/template/templateselect.png)
6. Vérifier et confirmer le déploiement

> ⏳ Le processus prend quelques minutes selon les ressources

### Vérification post-déploiement :

* Le nom d’hôte est bien appliqué
* Le réseau fonctionne
* Le compte administrateur est opérationnel
* Le fuseau horaire est correct

---

## VII. Conclusion

L’utilisation de **templates + spécifications de personnalisation** avec Windows Server 2022 permet :

* Gain de temps
* Standardisation
* Réduction des erreurs humaines
* Automatisation des déploiements à grande échelle

> 🔁 Pour modifier le modèle : convertir en VM > modifier > reconvertir en modèle.

---

**📌 Pense à capturer chaque étape dans vSphere pour illustrer ton projet.**
