---
title: Procédure de Restauration - WordPress (alexandre-faye.fr)
description: 
published: 1
date: 2026-09-25T15:49:17.039Z
tags: 
editor: markdown
dateCreated: 2026-02-15T13:25:15.010Z
---

# PROCEDURE DE RESTAURATION RAPIDE - ALEXANDRE-FAYE.FR

## 1. Verification des acces (Database & User)
Verifier le nom de la BDD et l'user dans la config WordPress
```bash
grep -E "DB_NAME|DB_USER" /var/www/alexandre-faye.fr/wp-config.php
```
Verifier les privileges de l'utilisateur alexandre sur la BDD newalexandre
```
sudo mysql -u root -p -e "SHOW GRANTS FOR 'alexandre'@'localhost';"
```
 
## 2. Nettoyage et Restauration des fichiers
Suppression du contenu actuel (nettoyage post-import Astra)
```
sudo rm -rf /var/www/alexandre-faye.fr/*
```

Extraction de la sauvegarde ZIP (Remplacer [DATE] par ex: 2026-02-01)
Source : /home/alexandre/Partage/wordpress/alexandre-faye.fr/backup/
```
sudo unzip /home/alexandre/Partage/wordpress/alexandre-faye.fr/backup/alexandre-faye-[DATE].zip -d /tmp/
sudo mv /tmp/alexandre-faye.fr/* /var/www/alexandre-faye.fr/
sudo rm -rf /tmp/restoration/
```

## 3. Restauration de la Base de Donnees
Import du dump SQL dans la base newalexandre

Source : /home/alexandre/Partage/wordpress/alexandre-faye.fr/mysql/
```
sudo mysql -u alexandre -p newalexandre < /home/alexandre/Partage/wordpress/alexandre-faye.fr/mysql/newalexandre.sql-[DATE]
```

## 4. Finalisation des permissions

Remise en place de l'appartenance à www-data et des droits 755
```

sudo chown -R www-data:www-data /var/www/alexandre-faye.fr/
sudo find /var/www/alexandre-faye.fr/ -type d -exec chmod 755 {} \;
sudo find /var/www/alexandre-faye.fr/ -type f -exec chmod 644 {} \;
```