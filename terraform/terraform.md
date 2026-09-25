---
title: Guide d'Installation de Terraform
description: 
published: 1
date: 2026-09-25T15:51:18.043Z
tags: 
editor: markdown
dateCreated: 2026-02-20T15:10:15.693Z
---

# Guide d'Installation de Terraform

Ce guide détaille les étapes nécessaires pour installer Terraform sur différents systèmes d'exploitation afin de pouvoir exécuter notre infrastructure as code.

---

## 1. Installation sur Ubuntu / Debian

HashiCorp fournit un dépôt officiel (repository) pour les distributions basées sur Debian. C'est la méthode recommandée pour faciliter les futures mises à jour.

Ouvrez votre terminal et exécutez les commandes suivantes :

```bash
# 1. Mettre à jour le système et installer les paquets prérequis
sudo apt-get update && sudo apt-get install -y gnupg software-properties-common curl

# 2. Télécharger et ajouter la clé GPG officielle de HashiCorp
curl -fsSL [https://apt.releases.hashicorp.com/gpg](https://apt.releases.hashicorp.com/gpg) | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg

# 3. Ajouter le dépôt HashiCorp à la liste des sources de votre système
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] [https://apt.releases.hashicorp.com](https://apt.releases.hashicorp.com) $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list

# 4. Mettre à jour le cache APT et installer Terraform
sudo apt-get update && sudo apt-get install terraform -y
```

---

## 2. Installation sur Windows

Pour Windows, la méthode la plus simple et automatisée consiste à utiliser un gestionnaire de paquets en ligne de commande.

**Option A : Avec Winget (inclus nativement dans Windows 10 et 11)**
Ouvrez PowerShell en tant qu'administrateur et tapez :
```powershell
winget install Hashicorp.Terraform
```

**Option B : Avec Chocolatey (si vous l'utilisez déjà)**
Ouvrez PowerShell en tant qu'administrateur et tapez :
```powershell
choco install terraform -y
```

*(Note : Redémarrez votre terminal après l'installation pour que la commande soit reconnue).*

---

## 3. Installation sur Rocky Linux (et autres distributions RHEL/CentOS)

Rocky Linux utilisant le gestionnaire de paquets `dnf` (ou `yum`), la logique est similaire à Debian mais avec les outils Red Hat.

Ouvrez votre terminal et exécutez :

```bash
# 1. Installer l'utilitaire de gestion des dépôts
sudo dnf install -y dnf-plugins-core

# 2. Ajouter le dépôt officiel de HashiCorp pour RHEL
sudo dnf config-manager --add-repo [https://rpm.releases.hashicorp.com/RHEL/hashicorp.repo](https://rpm.releases.hashicorp.com/RHEL/hashicorp.repo)

# 3. Installer Terraform
sudo dnf -y install terraform
```

---

## Vérification de l'installation (Tous systèmes)

Une fois l'installation terminée, peu importe votre système d'exploitation, ouvrez un nouveau terminal et tapez la commande suivante pour vérifier que Terraform est bien installé :

```bash
terraform -version
```

Si l'installation a réussi, vous devriez voir s'afficher la version actuelle (ex: `Terraform v1.x.x`).