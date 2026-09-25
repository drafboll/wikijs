---
title: Installation zabbix rocky
description: 
published: 1
date: 2026-09-25T15:47:58.495Z
tags: zabbix
editor: markdown
dateCreated: 2025-05-20T15:19:45.264Z
---

# Guide d'installation de Zabbix 7.2 sur Rocky Linux 9

Ce document rassemble en un seul bloc‑note les étapes de préparation du système et d'installation de Zabbix 7.2, avec des explications sur le **pourquoi** de chaque composant et la procédure d'accès à l'interface web.

---
 
## 1. Pré-requis

* **Système** : Rocky Linux 9 à jour (x86\_64)
* **Privilèges** : accès root ou via `sudo`
* **Matériel** : minimum 2 cœurs CPU, 2 Go RAM, 10 Go disque
* **Ports** : 80 (HTTP), 443 (HTTPS), 10051 (Zabbix Server), 10050 (Zabbix Agent)

> **Pourquoi ?** Ces ressources et accès sont nécessaires pour assurer performance, stockage de données et communication entre les composants.

---

## 2. Mise à jour du système et installation d'EPEL

```bash
sudo dnf update -y
sudo dnf install -y epel-release wget
```

**Pourquoi ?**

* `dnf update` : garantir les derniers correctifs de sécurité et fonctionnalités
* `epel-release` : fournir des paquets supplémentaires (PHP 8.0 enforcement, outils utiles)

---

## 3. Installation et configuration de MariaDB (MySQL)

1. **Installer MariaDB et lancer le service**

   ```bash
   sudo dnf install -y mariadb-server
   sudo systemctl enable --now mariadb
   ```
2. **Sécuriser MariaDB**

   ```bash
   sudo mysql_secure_installation
   ```

   * Définir un mot de passe fort pour `root`
   * Supprimer les utilisateurs anonymes
   * Désactiver la base de test
3. **Créer la base et l'utilisateur Zabbix**

```sql
mysql -uroot -p
password

create database zabbix character set utf8mb4 collate utf8mb4_bin;
create user zabbix@localhost identified by 'password';
grant all privileges on zabbix.* to zabbix@localhost;
quit;
   ```

> **Pourquoi ?** Zabbix utilise une base de données pour stocker la configuration, les métriques et l'historique.

---

## 4. Installation d'Apache HTTPD

```bash
sudo dnf install -y httpd
sudo systemctl enable --now httpd
```

**Pourquoi ?** Serveur web pour héberger l'interface frontend de Zabbix.

---

## 5. Installation de PHP 8.0 et extensions

1. **Basculer vers PHP 8.0** (Rocky 9 propose PHP 8.2 par défaut)

   ```bash
   sudo dnf module reset php -y
   sudo dnf module enable php:8.2 -y
   ```
2. **Installer PHP‑FPM et extensions nécessaires**

   ```bash
   sudo dnf install -y \
     php-fpm \
     php-mysqlnd \
     php-gd \
     php-xml \
     php-bcmath \
     php-mbstring \
     php-json \
     php-opcache
   ```
3. **Démarrer PHP‑FPM**

   ```bash
   sudo systemctl enable --now php-fpm
   ```
4. **Configurer la timezone**
   Ajoutez dans `/etc/php-fpm.d/zabbix.conf` :

   ```ini
   [zabbix]
   php_value[date.timezone] = 'Europe/Paris'
   ```

> **Pourquoi ?** L'interface Zabbix requiert PHP et ces extensions pour fonctionner correctement.

---

## 6. Configuration SELinux et pare-feu

1. **Mode SELinux (optionnel)**

   ```bash
   sudo setenforce 0
   # Pour permanent, éditez /etc/selinux/config et mettez SELINUX=permissive
   ```
2. **Ouvrir les ports via firewalld**

   ```bash
   sudo firewall-cmd --permanent --add-service=http
   sudo firewall-cmd --permanent --add-service=https
   sudo firewall-cmd --permanent --add-port=10051/tcp
   sudo firewall-cmd --permanent --add-port=10050/tcp
   sudo firewall-cmd --reload
   ```

> **Pourquoi ?** Assurer que les services peuvent communiquer et éviter les blocages liés à SELinux.

---

## 7. Vérification préliminaire

```bash
systemctl status mariadb httpd php-fpm
```

Tous les services doivent être en état **active (running)**.

---

## 8. Ajout du dépôt Zabbix et installation des paquets

1. **Importer la clé GPG et ajouter le dépôt**

   ```bash
   sudo rpm --import https://repo.zabbix.com/RPM-GPG-KEY-ZABBIX
   sudo rpm -Uvh \
     https://repo.zabbix.com/zabbix/7.2/release/rhel/9/noarch/\
     zabbix-release-latest.el9.noarch.rpm
   sudo dnf clean all
   ```
2. **Installer Zabbix Server, Frontend, scripts SQL et Agent2**

   ```bash
   sudo dnf install -y \
     zabbix-server-mysql \
     zabbix-web-mysql \
     zabbix-apache-conf \
     zabbix-sql-scripts \
     zabbix-selinux-policy \
     zabbix-agent2
   ```

> **Pourquoi installer ces paquets ?**
>
> * `zabbix-server-mysql` : le cœur du serveur Zabbix avec support MySQL
> * `zabbix-web-mysql` + `zabbix-apache-conf` : interface web et configuration Apache
> * `zabbix-sql-scripts` : schémas SQL initiaux
> * `zabbix-selinux-policy` : politiques SELinux pour Zabbix
> * `zabbix-agent2` : agent performant et extensible pour la collecte de métriques

---

## 9. Import du schéma initial et configuration serveur

1. **Importer le schéma SQL**

   ```bash
   zcat /usr/share/doc/zabbix-sql-scripts/mysql/*.sql.gz \
     | mysql -uzabbix -p zabbix
   ```

   ```bash
   mysql -uroot -p
	 password
	 grant all privileges on zabbix.* to zabbix@localhost;
	 quit;
   ```
2. **Configurer `/etc/zabbix/zabbix_server.conf`**

   ```ini
   DBHost=localhost
   DBName=zabbix
   DBUser=zabbix
   DBPassword=VotreMotDePasseZabbix
   ```

> **Pourquoi ?** Relier Zabbix Server à la base de données que vous avez préparée.

---

## 10. Démarrage des services Zabbix

```bash
sudo systemctl enable --now \
  zabbix-server \
  zabbix-agent2 \
  httpd \
  php-fpm
```

> **Pourquoi ?** Lancer le serveur, l'agent, le serveur web et PHP-FPM pour la plateforme de monitoring.

---

## 11. Accès à l'interface web Zabbix

* **URL** : `http://<IP_de_votre_serveur>/zabbix`
* **Identifiants par défaut** :

  * **Utilisateur :** `Admin`
  * **Mot de passe :** `zabbix`
![zabbix.png](/zabbix/zabbix.png)
Suivez l’assistant d’installation web pour vérifier les prérequis, configurer le frontend et définir votre mot de passe administrateur.
![zabbix_acceuil.png](/zabbix/zabbix_acceuil.png)
---

### Prochaines étapes

* Ajouter vos hôtes et templates
* Configurer alertes et actions
* Installer des plugins ou modules pour étendre les capacités de `zabbix-agent2`

*Fin du bloc‑note d'installation*
