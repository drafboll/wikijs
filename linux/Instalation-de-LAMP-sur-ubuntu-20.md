---
title: Installer LAMP sur ubuntu 20.04
description: Installation LAMP et GLPI sur ubuntu 20.04
published: 1
date: 2026-09-25T15:48:28.445Z
tags: apache, lamp, mariadb, php
editor: markdown
dateCreated: 2023-02-01T10:39:22.970Z
---

# Installation de LAMP sur Ubuntu 20.04
## I. Mise à jour des paquets système:
```
sudo apt update
sudo apt upgrade
``` 
## Installation du serveur Apache
On passe maintenant à l’installation du serveur Web Apache. Lancer la commande suivante pour procéder à l’installation d’Apache : 
```
sudo apt install apache2
```
Une fois installé, Apache devrait être démarré automatiquement. Vérifiez son état avec la commande systemctl.
```
systemctl status apache2
```
Voici ce que vous devriez avoir :
```
● apache2.service - The Apache HTTP Server
     Loaded: loaded (/lib/systemd/system/apache2.service; enabled; vendor preset: enabled)
     Active: active (running) since Thu 2020-07-16 21:13:21 UTC; 55s ago
       Docs: https://httpd.apache.org/docs/2.4/
   Main PID: 45918 (apache2)
      Tasks: 55 (limit: 4621)
     Memory: 4.9M
     CGroup: /system.slice/apache2.service
             ├─45918 /usr/sbin/apache2 -k start
             ├─45920 /usr/sbin/apache2 -k start
             └─45921 /usr/sbin/apache2 -k start
```
Une fois que vous avez l’adresse IP de votre serveur, saisissez-la dans la barre d’adresse de votre navigateur :
```
http://your_server_ip
```
Vous devriez voir la page web par défaut d’Apache Ubuntu 20.04 :
![apache.png](/apache.png)

Si la connexion est refusée ou bloquée, jeter un coup d’oeil au niveau du pare-feu de votre serveur (iptable ou UFW par exemple).

Maintenant que nous savons que notre serveur Apache est correctement installé, nous allons passer à la phase d’installation.
Configuration Apache

Maintenant, nous devons définir  www-data (utilisateur Apache) en tant que propriétaire de la racine du document (autrement appelé racine Web). Par défaut, il appartient à l’utilisateur root.
```
sudo chown www-data:www-data /var/www/html/ -R
```
Par défaut, Apache utilise le nom d’hôte du système comme son global ServerName. Si le nom d’hôte du système ne peut pas être résolu via DNS, alors vous risquez d’avoir une erreur lors de l’utilisation de la commande suivante :
```
sudo apache2ctl -t
```

## Installation de MySQL
### installez le paquet mysql-server :
```
sudo apt install mysql-server
```
Cela installera MySQL, mais ne vous demandera pas de définir un mot de passe ou de faire d’autres changements de configuration. Comme cela rend votre installation de MySQL non sécurisée, nous allons aborder ce point.
> Avertissement : à partir de juillet 2022, une erreur se produira lorsque vous exécuterez le script `mysql_secure_installation` sans configuration supplémentaire. La raison en est que ce script tentera de définir un mot de passe pour le compte MySQL root de l'installation mais, par défaut sur les installations Ubuntu, ce compte n'est pas configuré pour se connecter à l'aide d'un mot de passe.
{.is-danger}



Attention juillet 2022, ce script échouait silencieusement après avoir tenté de définir le mot de passe du compte root et continuait avec le reste des invites. Cependant, au moment d'écrire ces lignes, le script renverra l'erreur suivante après avoir entré et confirmé un mot de passe :



```
output
 ... Failed! Error: SET PASSWORD has no significance for user 'root'@'localhost' as the authentication method used doesn't store authentication data in the MySQL server. Please consider using ALTER USER instead if you want to change authentication parameters.

New password:
```

Cela entraînera le script dans une boucle récursive dont vous ne pourrez sortir qu'en fermant la fenêtre de votre terminal.

Étant donné que le script mysql_secure_installation effectue un certain nombre d'autres actions utiles pour sécuriser votre installation MySQL, il est toujours recommandé de l'exécuter avant de commencer à utiliser MySQL pour gérer vos données. Pour éviter d'entrer dans cette boucle récursive, cependant, vous devrez d'abord ajuster la façon dont votre utilisateur root MySQL s'authentifie.

Tout d'abord, ouvrez l'invite MySQL :
```
sudo mysql
```
Exécutez ensuite la commande ALTER USER suivante pour remplacer la méthode d'authentification de l'utilisateur root par une méthode utilisant un mot de passe. L'exemple suivant change la méthode d'authentification en mysql_native_password :
```
ALTER USER 'root'@'localhost' IDENTIFIED mysql_native_password BY 'mettre son mdp' ;
```
Après avoir effectué cette modification, quittez l'invite MySQL :
```
exit
```
Ensuite, vous pouvez exécuter le script mysql_secure_installation sans problème.

Une fois le script de sécurité terminé, vous pouvez alors rouvrir MySQL et rétablir la méthode d'authentification de l'utilisateur root par défaut, auth_socket. Pour vous authentifier en tant qu'utilisateur MySQL root à l'aide d'un mot de passe, exécutez cette commande :
```
mysql -u root -p
```
Revenez ensuite à la méthode d'authentification par défaut à l'aide de cette commande :
```
ALTER USER 'root'@'localhost' IDENTIFIED WITH auth_socket ;
```
Cela signifie que vous pourrez à nouveau vous connecter à MySQL en tant qu'utilisateur root à l'aide de la commande sudo mysql.
Vous serez alors guidé à travers une série d’invites où vous pourrez apporter quelques modifications aux options de sécurité de votre installation MySQL. La première invite vous demandera si vous souhaitez configurer le plugin Validate Password, que vous pouvez utiliser pour tester la solidité de votre mot de passe MySQL.

Pour les nouvelles installations de MySQL, vous devrez exécuter le script de sécurité inclus dans le SGBD. Ce script modifie certaines des options par défaut les moins sûres pour des choses comme les connexions root distantes et les sample users.
```
sudo mysql_secure_installation
```
### creer des utilisateurs et une base de donnée
Certains peuvent aussi trouver qu’il est plus adapté à leur travail de se connecter à MySQL avec un utilisateur dédié. Pour créer un tel utilisateur, ouvrez à nouveau le shell MySQL :
```
sudo mysql
```
> Remarque : si vous avez activé l’authentification par mot de passe pour root, comme décrit dans les paragraphes précédents, vous devrez utiliser une commande différente pour accéder au shell MySQL. Ce qui suit permettra d’exécuter votre client MySQL avec les privilèges d’utilisateur habituels, et vous n’obtiendrez les privilèges d’administrateur au sein de la base de données qu’en vous authentifiant :
{.is-success}
```
mysql -u root -p
```
De là, créez un nouvel utilisateur et attribuez-lui un mot de passe fort :
```
CREATE USER 'sammy'@'localhost' IDENTIFIED BY 'password';
```
Si tu veux creer une database :
Créez votre base de données. Dans la ligne de commande MySQL, entrez la commande : 
```
CREATE DATABASE NOMDEVOTREBASE;
```
Remplacez NOMDEVOTREBASE par le nom de votre base de données sans espace aucun.

Affichez la liste de vos bases de données. Entrez la commande :
```
SHOW DATABASES;
```
qui liste les bases de données disponibles sur le serveur MySQL. En parallèle, vous verrez apparaitre une base de données mysql (qui gère les accès et les privilèges) et une base test (qui sert aux utilisateurs pour y effectuer leurs tests). Pour l'instant, vous pouvez ignorer ces remarques. 

La simple création de ce nouvel utilisateur ne suffit pas. Vous devez lui accorder des privilèges. Pour accorder à l’utilisateur récemment créé tous les privilèges pour la base de données, exécutez la commande suivante :
```
GRANT ALL PRIVILEGES ON NOMDELADATABASECREE . * TO 'utilisateur crée'@'localhost';
```
ou bien pour qu'il ai accès à toutes les databases
```
GRANT ALL PRIVILEGES ON *.* TO 'sammy'@'localhost' WITH GRANT OPTION;
```
puis sortir :
```
exit
```
## Installation de PHP
Au moment où j’écris cet article, c’est PHP 7.4 la dernière version stable de PHP. Vous pouvez bien évidemment adapter les commandes en fonction de la version dont vous avez besoin. Entrez la commande suivante pour installer PHP 7.4 et quelques modules PHP qui pourraient vous être utile.
```
sudo apt install php7.4 libapache2-mod-php7.4 php7.4-mysql php-common php7.4-cli php7.4-common php7.4-json php7.4-opcache php7.4-readline
```
Activer le module Apache PHP puis redémarrer Apache avec la commande suivante : 
```
sudo a2enmod php7.4
sudo systemctl restart apache2
```
Vous pouvez vérifier la version de PHP avec la commande suivante :
```
php --version
```
#### Activer PHP FPM
Dans certains cas, vous aurez besoin de  PHP-FPM pour obtenir de meilleures performances pour votre serveur web, nous allons alors activer PHP-FPM sur notre serveur Apache.

Désactiver le module PHP 7.4
```
sudo a2dismod php7.4
```
Installer le module PHP-FPM
```
sudo apt install php7.4-fpm
```
Activer les modules proxy_fcgi et setenvif
```
sudo a2enmod proxy_fcgi setenvif
```
Activer le fichier de configuration /etc/apache2/conf-available/php7.4-fpm.conf
```
sudo a2enconf php7.4-fpm
```
Puis redémarrer Apache :
```
sudo systemctl restart apache2
```
















