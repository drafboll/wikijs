---
title: Procédure de Restauration - WordPress (alexandre-faye.fr)
description: 
published: 1
date: 2026-11-12T00:56:22.953Z
tags: 
editor: markdown
dateCreated: 2026-02-15T13:25:15.010Z
---

## PROCÉDURE DE RESTAURATION - WordPress alexandre-faye.fr

Sauvegardes : /home/alexandre/Partage/wordpress/alexandre-faye.fr/backup/  (copie : /media/alexandre/sauvegarde-wordpress/alexandre-faye.fr/)
Chaque zip contient le dossier du site ET le dump SQL du même jour.

### 1. Vérifier les accès
```bash
sudo grep -E "DB_NAME|DB_USER" /var/www/alexandre-faye.fr/wp-config.php
sudo mysql -u root -p -e "SHOW GRANTS FOR 'alexandre'@'localhost';"
```

### 2. Extraire la sauvegarde dans un dossier dédié (remplacer AAAA-MM-JJ)
```bash
sudo rm -rf /tmp/restauration
sudo install -d -m 700 /tmp/restauration
sudo unzip -q /home/alexandre/Partage/wordpress/alexandre-faye.fr/backup/alexandre-faye-AAAA-MM-JJ.zip -d /tmp/restauration
sudo ls /tmp/restauration/alexandre-faye.fr /tmp/restauration/home/alexandre/Partage/wordpress/alexandre-faye.fr/mysql
```
On voit le site (wp-config.php, wp-content…) et le fichier newalexandre.sql-AAAA-MM-JJ.

### 3. Remplacer les fichiers du site (fichiers cachés compris, ex. .htaccess)
```bash
sudo find /var/www/alexandre-faye.fr -mindepth 1 -delete
sudo cp -a /tmp/restauration/alexandre-faye.fr/. /var/www/alexandre-faye.fr/
```
Chemin complet : ne vide que ce dossier, d'où qu'on lance la commande.
« /. » copie tout le contenu, fichiers cachés inclus.

### 4. Restaurer la base (dump pris dans le zip)
```bash
sudo mysql -u alexandre -p newalexandre < /tmp/restauration/home/alexandre/Partage/wordpress/alexandre-faye.fr/mysql/newalexandre.sql-AAAA-MM-JJ
```

### 5. Droits
```bash
sudo chown -R www-data:www-data /var/www/alexandre-faye.fr
sudo find /var/www/alexandre-faye.fr -type d -exec chmod 755 {} +
sudo find /var/www/alexandre-faye.fr -type f -exec chmod 644 {} +
sudo chmod 640 /var/www/alexandre-faye.fr/wp-config.php
```

### 6. Nettoyer (le dump et wp-config.php contiennent des secrets)
```bash
sudo rm -rf /tmp/restauration
```

### 7. Vérifier
Ouvrir le site, se connecter à /wp-admin, puis purger le cache Cloudflare.