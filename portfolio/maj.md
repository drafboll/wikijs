---
title: Mise à jour du site
description: 
published: 1
date: 2026-11-11T11:10:30.738Z
tags: 
editor: markdown
dateCreated: 2026-11-11T11:09:57.190Z
---

# METTRE À JOUR LE SITE ALEXANDRE-FAYE.FR

Comment envoyer sur le serveur MINI une modification faite sur le PC, revenir en arrière si
besoin, mettre à jour le formulaire ou la configuration Apache, et entretenir le serveur.

> Avant cette page : la modification est faite et vérifiée sur le PC avec `mettre-a-jour.bat`
> (voir *Modifier le contenu du site*).
{.is-info}

---

## 1. Vue d'ensemble

| Étape | Où | Durée |
|---|---|---|
| 1. Préparer l'archive | PC | 1 min |
| 2. L'envoyer sur MINI | PC | 1 min |
| 3. Sauvegarder la version en ligne | MINI | quelques secondes |
| 4. Installer la nouvelle version | MINI | quelques secondes |
| 5. Vider le cache Cloudflare | navigateur | 1 min |
| 6. Vérifier | navigateur | 1 min |

---

## 2. Préparer l'archive (PC)

Double-clic sur **`preparer-mise-en-ligne.bat`**, dans `D:\Documents\Claude\alexandre-faye-3d\`.

Il fait deux choses :

1. il régénère le site dans `site/` (comme `mettre-a-jour.bat`) ;
2. il emballe uniquement les fichiers utiles dans **`mise-en-ligne\alexandre-faye-3d.tar.gz`**,
   et copie les fichiers serveur (`contact.php`, `contact.ini`, `regles.conf`, script de
   sauvegarde) dans `mise-en-ligne\serveur\`.

À la fin, la fenêtre affiche **« Pret »**. Si elle affiche **« ERREUR »**, rien n'a été préparé :
lis le message, corrige (souvent une virgule dans un `.json`) et relance.

---

## 3. Envoyer l'archive sur MINI (PC)

Ouvre PowerShell **dans le dossier du projet** : dans l'Explorateur Windows, ouvre
`D:\Documents\Claude\alexandre-faye-3d`, clique dans la barre d'adresse, tape `powershell` et
**Entrée**.

```bash
scp mise-en-ligne\alexandre-faye-3d.tar.gz alexandre@MINI:~/
```

`scp` copie le fichier par SSH dans ton dossier personnel sur MINI (`~/`, c'est-à-dire
`/home/alexandre/`). Il demande ton mot de passe SSH. L'ancienne archive du même nom est remplacée.

---

## 4. Sauvegarder la version en ligne (MINI)

Avant de remplacer le site, on garde une copie de la version actuelle. Le plus simple : lancer le
script de sauvegarde à la main.

```bash
sudo /bin/bash /scripts/alexandre-faye-3d-sauvegarde.sh
```

Il crée `/home/alexandre/Partage/alexandre-faye-3d/backup/alexandre-faye-3d-<date du jour>.tar.gz`,
le copie sur le disque de sauvegarde et t'envoie l'e-mail « Sauvegarde site 3D : OK ».

> C'est ton bouton « annuler » : si la nouvelle version pose problème, la section 7 explique
> comment revenir à celle-ci en deux commandes.
{.is-info}

---

## 5. Installer la nouvelle version (MINI)

Six commandes, toujours les mêmes, à taper dans l'ordre :

```bash
sudo rm -rf /var/www/alexandre-faye-3d
sudo mkdir /var/www/alexandre-faye-3d
sudo tar -xzf ~/alexandre-faye-3d.tar.gz -C /var/www/alexandre-faye-3d --no-same-owner --no-same-permissions
sudo chown -R root:root /var/www/alexandre-faye-3d
sudo find /var/www/alexandre-faye-3d -type d -exec chmod 755 {} +
sudo find /var/www/alexandre-faye-3d -type f -exec chmod 644 {} +
```

| Commande | Ce qu'elle fait |
|---|---|
| `rm -rf …` | supprime l'ancienne version, pour ne pas laisser traîner de fichiers supprimés entre-temps |
| `mkdir …` | recrée le dossier vide |
| `tar -xzf …` | extrait l'archive dedans ; `--no-same-owner --no-same-permissions` ignore les droits venus de Windows |
| `chown -R root:root …` | tout appartient à root : Apache peut lire mais pas modifier |
| `find … -type d … 755` | dossiers : lisibles et traversables par tous, modifiables par root seul |
| `find … -type f … 644` | fichiers : lisibles par tous, modifiables par root seul |

Le site est en ligne dès la commande `tar`. Le formulaire de contact n'est **pas** touché : il
est dans un autre dossier (`/var/www/alexandre-faye-3d-api`).

> Si la première commande est mal tapée (par exemple un espace en trop dans le chemin), `rm -rf`
> peut supprimer autre chose. **Copie-colle** les commandes plutôt que de les retaper.
{.is-danger}

---

## 6. Vider le cache Cloudflare et vérifier

### a. Purge Cloudflare

1. Va sur **dash.cloudflare.com** → le domaine **alexandre-faye.fr** ;
2. menu **Caching** → **Configuration** ;
3. bouton **Purge Everything** → confirmer.

Cloudflare garde une copie des fichiers (photos, CSS, JS) pour les servir plus vite. La purge
l'oblige à reprendre la nouvelle version sur MINI.

### b. Vérifier

1. Ouvre **https://alexandre-faye.fr** et fais **Ctrl+F5** (rechargement complet) ;
2. va sur la page modifiée et vérifie la modification ;
3. vérifie que la **3D s'affiche** sur l'accueil, que les **photos** s'ouvrent, et que le
   **formulaire** répond (un message de test de temps en temps).

> Si la modification ne s'affiche toujours pas : essaie dans une fenêtre de navigation privée
> (Ctrl+Maj+N). Si elle y apparaît, c'est le cache de ton navigateur.
{.is-info}

---

## 7. Revenir à la version précédente

Si la nouvelle version pose problème, on remet celle sauvegardée à l'étape 4. Repère d'abord le
nom de la sauvegarde :

```bash
sudo ls -lh /home/alexandre/Partage/alexandre-faye-3d/backup/
```

Puis, en remplaçant `[DATE]` par la date de la sauvegarde (ex. `2026-12-06`) :

```bash
sudo rm -rf /var/www/alexandre-faye-3d
sudo tar -xzf /home/alexandre/Partage/alexandre-faye-3d/backup/alexandre-faye-3d-[DATE].tar.gz -C / var/www/alexandre-faye-3d
```

- `-C /` : extrait depuis la racine du serveur, à l'emplacement d'origine ;
- `var/www/alexandre-faye-3d` à la fin : **n'extrait que le site**, sans toucher au formulaire ni
  à la configuration Apache ;
- les droits d'origine (root:root, 755/644) sont dans l'archive et sont remis en place.

Puis purge Cloudflare et Ctrl+F5 (section 6).

**Tout restaurer** (site, formulaire, réglages, configuration Apache), par exemple après une
réinstallation du serveur :

```bash
sudo tar -xzf /home/alexandre/Partage/alexandre-faye-3d/backup/alexandre-faye-3d-[DATE].tar.gz -C /
sudo apache2ctl configtest
sudo systemctl reload apache2
systemctl is-active apache2
```

Si le disque du serveur est perdu, la même archive se trouve sur le disque de sauvegarde, dans
`/media/alexandre/sauvegarde-site/alexandre-faye-3d/`.

---

## 8. Mettre à jour le formulaire ou la configuration Apache

Ces fichiers ne sont **pas** dans l'archive du site : ils sont installés une fois, à part. On ne
les met à jour que s'ils ont changé sur le PC (dans `deploiement\`), par exemple après une
modification faite avec Claude.

### a. Envoyer les fichiers serveur (PC)

Après `preparer-mise-en-ligne.bat` :

```bash
scp -r mise-en-ligne\serveur alexandre@MINI:~/
```

### b. Le programme du formulaire : `contact.php`

```bash
sudo install -m 644 -o root -g root ~/serveur/contact.php /var/www/alexandre-faye-3d-api/contact.php
```

Pris en compte immédiatement, sans recharger Apache.

### c. Les réglages du formulaire : `contact.ini`

> **Attention** : sur le serveur, `contact.ini` peut contenir la clé secrète Turnstile, qui n'est
> pas dans la version du PC. Ne l'écrase pas : modifie-le directement sur MINI.
{.is-warning}

```bash
sudo nano /etc/alexandre-faye-3d/contact.ini
```

Modifie, puis **Ctrl+O**, **Entrée**, **Ctrl+X**. Pris en compte au message suivant.

### d. Les règles Apache : `regles.conf`

C'est le fichier le plus sensible : une erreur peut couper le site. On compare **toujours** avant
de remplacer.

1. **Copie de secours** :

```bash
sudo cp /etc/apache2/alexandre-faye-3d/regles.conf /etc/apache2/alexandre-faye-3d/regles.conf.avant-maj
```

2. **Comparer** l'actuel et le nouveau (`<` = actuel, `>` = nouveau) :

```bash
diff /etc/apache2/alexandre-faye-3d/regles.conf ~/serveur/regles.conf
```

   Vérifie qu'aucune ligne importante ne disparaît (une ligne `<` sans équivalent `>`). C'est
   comme ça qu'on a évité de perdre l'en-tête HSTS le 10/11/2026.

3. **Installer** :

```bash
sudo install -m 644 -o root -g root ~/serveur/regles.conf /etc/apache2/alexandre-faye-3d/regles.conf
```

4. **Vérifier la syntaxe** ; il faut lire `Syntax OK` (l'avertissement `DocumentRoot [/var/www/html]`
   est connu et sans gravité) :

```bash
sudo apache2ctl configtest
```

5. **Seulement si `Syntax OK`**, recharger puis vérifier qu'Apache tourne (`active`) :

```bash
sudo systemctl reload apache2
systemctl is-active apache2
```

6. **En cas de problème**, remettre l'ancien fichier et recharger :

```bash
sudo cp /etc/apache2/alexandre-faye-3d/regles.conf.avant-maj /etc/apache2/alexandre-faye-3d/regles.conf
sudo systemctl reload apache2
```

### e. Le script de sauvegarde

```bash
bash -n ~/serveur/alexandre-faye-3d-sauvegarde.sh
sudo install -m 700 -o root -g root ~/serveur/alexandre-faye-3d-sauvegarde.sh /scripts/alexandre-faye-3d-sauvegarde.sh
sudo /bin/bash /scripts/alexandre-faye-3d-sauvegarde.sh
```

`bash -n` vérifie la syntaxe sans rien lancer (aucun affichage = c'est bon), puis on installe et
on lance une fois à la main pour recevoir l'e-mail de contrôle.

---

## 9. Entretenir le serveur (chaque mois)

Le site lui-même n'a **rien à mettre à jour** : ce sont des fichiers statiques, et la 3D tourne
dans le navigateur des visiteurs. Ce qu'il faut entretenir, c'est **MINI**.

### a. Mises à jour du système

```bash
sudo apt update
sudo apt full-upgrade
```

`update` récupère la liste des nouvelles versions, `full-upgrade` les installe (Apache, PHP,
Postfix, correctifs de sécurité…). Lis ce qu'il propose avant de valider avec `O`.

Ensuite, vérifier qu'Apache tourne toujours :

```bash
systemctl is-active apache2
```

Si `full-upgrade` annonce un nouveau noyau, MINI redémarre de toute façon tous les deux jours à
5 h (crontab).

### b. Certificat HTTPS

Certbot renouvelle le certificat tout seul (crontab, tous les 3 jours à 3 h). On vérifie que le
renouvellement fonctionnerait :

```bash
sudo certbot renew --dry-run
```

Il faut lire que tous les renouvellements simulés ont réussi (`succeeded`).

> **Indispensable avec HSTS** : les navigateurs refusent le site en HTTP pendant un an. Si le
> certificat expire, le site devient **inaccessible**, sans bouton « continuer quand même ».
{.is-danger}

### c. Les sauvegardes

- Vérifie que tu reçois bien l'e-mail **« Sauvegarde site 3D : OK »** les 1, 11, 21 et 31 du mois.
- Un e-mail **« ECHEC »** donne la cause sur la première ligne (le plus souvent : disque de
  sauvegarde non monté).
- Pas d'e-mail du tout : vérifie la crontab (`sudo crontab -l`) et le journal des e-mails
  (`sudo tail -n 30 /var/log/mail.log`).

### d. Le formulaire

De temps en temps, envoie-toi un message depuis le site. S'il n'arrive pas :

```bash
sudo tail -n 30 /var/log/mail.log
sudo tail -n 30 /var/log/apache2/error.log
```

Dans `mail.log`, chaque envoi doit finir par `status=sent`.

---

## 10. Aide-mémoire

```bash
# --- PC (PowerShell, dans D:\Documents\Claude\alexandre-faye-3d) ---
# double-clic sur preparer-mise-en-ligne.bat, puis :
scp mise-en-ligne\alexandre-faye-3d.tar.gz alexandre@MINI:~/

# --- MINI ---
sudo /bin/bash /scripts/alexandre-faye-3d-sauvegarde.sh
sudo rm -rf /var/www/alexandre-faye-3d
sudo mkdir /var/www/alexandre-faye-3d
sudo tar -xzf ~/alexandre-faye-3d.tar.gz -C /var/www/alexandre-faye-3d --no-same-owner --no-same-permissions
sudo chown -R root:root /var/www/alexandre-faye-3d
sudo find /var/www/alexandre-faye-3d -type d -exec chmod 755 {} +
sudo find /var/www/alexandre-faye-3d -type f -exec chmod 644 {} +

# --- Navigateur ---
# Cloudflare : Caching > Configuration > Purge Everything
# https://alexandre-faye.fr : Ctrl+F5
```
