---
title: Installer glpi sur ubuntu 20.04
description: 
published: 1
date: 2026-09-25T15:48:38.684Z
tags: glpi, lamp
editor: markdown
dateCreated: 2023-02-14T14:44:09.932Z
---

# Installer GLPI sur ubuntu 20.04
## I. Mise à jour des paquets système:
```
sudo apt update
sudo apt upgrade
```
## II. Installation de LAMP
> https://wiki.alexandre-faye.fr/fr/linux/Instalation-de-LAMP-sur-ubuntu-20
{.is-info}
 
## III. Téléchargez et installez GLPI
Téléchargez la dernière version stable de GLPI. Il suit un schéma de version sémantique, sur 3
chiffres, où le premier est la version majeure , le second la version mineure et le troisième la
version du correctif .
Recherchez la dernière version stable sur la page Téléchargements . Au
moment d'écrire ces lignes, c'est 9.4.5.
```
sudo apt-get -y install wget
export VER="9.4.5"
wget https://github.com/glpi-project/glpi/releases/download/$VER/g
lpi-$VER.tgz
```
Décompressez l'archive téléchargée:
```
tar xvf glpi-$VER.tgz
```
Déplacez le glpi dossier créé dans le /var/www/html
```
sudo mv glpi /var/www/html/
```
Donnez à l'utilisateur Apache la propriété du répertoire:
```
sudo chown -R www-data:www-data /var/www/html/
```
Installer des dépendance php
``` 
sudo apt -y install php php-{curl,gd,imagick,intl,apcu,memcache,imap,mysql,cas,ldap,tidy,pear,xmlrpc,pspell,mbstring,json,iconv,xml,gd,xsl}
``` 
Redemarer apache2
``` 
sudo systemctl restart apache2
``` 


## IV. Terminez l'installation de GLPI
Visitez l'adresse IP de votre serveur ou l'URL de votre nom d'hôte sur /glpi.S'il s'agit de votre
machine locale, vous pouvez utiliser: http://127.0.0.1/glpi/install/install.php
Sur la première page, sélectionnez votre langue.
![glpi_setup.png](/glpi/glpi_setup.png)
Acceptez les termes de la licence et cliquez sur « Continuer »
![licence_glpi.png](/glpi/licence_glpi.png)
Choisissez « Installer » pour une toute nouvelle installation de GLPI
![installation.png](/glpi/installation.png)
Confirmez que les vérifications de la compatibilité de votre environnement avec
l'exécution de GLPI sont réussies.
![test_de_compatibilité.png](/glpi/test_de_compatibilité.png)
Configurez la connexion à la base de données, avec le nom et la base créé
![base_de_donnée_glpi.png](/glpi/base_de_donnée_glpi.png)
Lors de la première connexion, vous êtes invité à modifier le mot de passe. Veuillez définir le nouveau mot de passe avant de configurer GLPI. Cela se fait dans Administration > Utilisateurs .
![page_d'acceuil.png](/glpi/page_d'acceuil.png)
Les identifiants et mots de passe par défaut sont :

    Utilisateur : glpi / Mot de passe : glpi pour le compte administrateur

    tech/tech pour le compte technicien

    normal/normal pour le compte normal

    post-only/postonly pour le compte postonly

Vous pouvez supprimer ou modifier ces comptes ainsi que les données initiales.



















## V. Créer la liaison LDAP
Depuis le menu de navigation aller sur Configuration > Authentifications
Annuaire LDAP
![laison_ldap.png](/glpi/laison_ldap.png)
- Nom : du pc de l’AD (WIN-25P6AB0M624)
- Server par défaut : oui et actif : oui
- Serveur : AD 192.168.136.120
- Port : 389
- Filtre de connexion : (&(objectClass=user)(objectCategory=person)(!(userAccountControl:1.2.840.113556.1.4.803:=2)))
- BASE DN : OU=GLPI,DC=ALEXANDRE,DC=LOCAL

![dns.png](/glpi/dns.png)
DN du compte : compte de l’OU (glpi)
Mdp : mdp de l’utilisateur
Champs de l’identifiant : samaccountname
Champs de synchronisation : objectguid
Sauvegarder 

![sauvegarde_ldap.png](/glpi/sauvegarde_ldap.png)
Puis retourner dans la configuration et aller tester le serveur

## VI. Créer des utilisateurs et les importer dans glpi
Maintenant nous créons des utilisateurs dans l’AD et dans l’OU glpi
![ou.png](/glpi/ou.png)

Nous allons les rapatrier dans glpi
Aller dans utilisateurs > annuaire LDAP > Importation de nouveaux utilisateurs > cliquer sur mode expert 
![ldap_annuaire.png](/glpi/ldap_annuaire.png)
Maintenant vous pouvez vous connecter à votre utlisateur sur glpi
![ou.png](/glpi/ou.png)

## VII. Configurer l’agent fusion inventory
Pour ce faire installer le plugin agent inventorie pour le server glpi

> Vérifier la version de glpi
{.is-info}

 ![version_glpi.png](/glpi/version_glpi.png)
Puis aller sur : https://github.com/fusioninventory/fusioninventory-for-glpi/releases?q=Version+9.4&expanded=true
Pour trouver le fichier compresser de votre version de glpi
Décompresser la 2 fois et aller l’insérer dans le fichier : /var/www/html/glpi/plugins

ou
```
apt install fusioninventory-agent
nano /etc/fusioninventory/agent.cfg
```
on decommente la ligne
```
server = http://192.168.122.55/glpi/plugins/fusioninventory/
```
et on lance dans un terminal

fusioninventory-agent
Ensuite retourner sur le site dans configuration > Plugins vous trouverez votre plugin et installer là. 
![installer_fusion_inventory.png](/glpi/installer_fusion_inventory.png)
Et penser à l'activer en cochant le bouton vert.

## Configuration de fusion inventory 
Administration > fusion inventory
 
Nous allons ensuite dans général > configuration générale
Mettre la fréquence d’inventaire sur 12h
Pour configurer les éléments remontés par l’agent il suffit d’aller dans inventaire ordinateur

Pour enlever le Cron de GLPI ne fonctionne pas , voir documentation
On peut voir que cron ne fonctionne pas. Cron permet les tâches planifiées automatiques. Afin de résoudre le problème, on rentre dans le Shell de linux via PuTTY. On rentre cette commande :
```
sudo crontab -u www-data -e
```
On tape 1. On descend tout en bas avec les flèches directionnelles. On écrit :
```
*/1 * * * * /usr/bin/php5 /var/www/html/glpi/front/cron.php &>/dev/null
```
On fait ctrl+O puis entrée (cela va sauvegarder). On quitte avec CTRL+X.

On relance cron :
```
/etc/init.d/cron restart
```
On se rend dans le navigateur web, sur GLPI puis dans la rubrique Configuration > Actions Automatiques.

On clique sur taskscheduler puis exécuter.
![task.png](/glpi/task.png)
Cron est de nouveau fonctionnel !
Tout est prêt pour inventorier notre parc !



















