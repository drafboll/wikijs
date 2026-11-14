---
title: Site alexandre-faye.fr
description: 
published: 1
date: 2026-11-14T19:39:38.835Z
tags: 
editor: markdown
dateCreated: 2026-11-11T09:00:25.593Z
---

# CRÉATION DU SITE ALEXANDRE-FAYE.FR

Comment mon nouveau site a été construit, avec quel outil et quel skill, ce qui a été
modifié par rapport au modèle de départ, puis comment le formulaire de contact et le script
de sauvegarde ont été mis en place sur le serveur MINI.

![capture_d'écran_2026-11-13_183120.png](/portolio/capture_d'écran_2026-11-13_183120.png)
> **Pages liées** : *Modifier le contenu du site* (ajouter un récit, des photos, un sport…)
> et *Mettre à jour le site en ligne*.
{.is-info}

---

## 1. Le point de départ

L'ancien site `alexandre-faye.fr` était un **WordPress** (thème Astra, base MySQL
`newalexandre`), hébergé sur MINI (Ubuntu, Apache, derrière Cloudflare).

Objectif : le remplacer par un **site statique** plus moderne et plus sûr :

- plus de PHP pour les pages, plus de base de données, plus d'extensions à mettre à jour ;
- un rendu « haut de gamme » : un objet en verre 3D qui éclate puis se recompose au scroll ;
- garder le contenu existant : photos, vidéos, parcours, compétences, pages de sport.

---

## 2. Les outils utilisés

| Outil | Rôle |
|---|---|
| **Claude** (application de bureau, onglet *Code*, Claude Code) | Écrit et modifie les fichiers du projet sur le PC, lance les scripts, guide les commandes serveur |
| **Skill `premium-3d-glass`** | Le modèle de départ du site (voir section 3) |
| **Python** (`py`) sur le PC | Génère le site à partir des sources (`build_site.py`) et prépare l'archive (`package_site.py`) |
| **Three.js r160** | Le moteur 3D, exécuté dans le navigateur du visiteur (rien ne tourne sur le serveur) |
| **Apache + PHP 8.4** sur MINI | Sert les fichiers du site ; PHP ne sert **que** au formulaire de contact |
| **Postfix + relais OVH** | Envoie les e-mails du formulaire et des sauvegardes |
| **Cloudflare** | DNS, cache, HTTPS côté visiteurs, masque l'IP réelle du serveur |

### Comment on travaille avec Claude

1. On ouvre l'application Claude, onglet **Code**, sur le dossier `D:\Documents\Claude`.
2. On décrit ce qu'on veut en français (« ajoute une page sport », « le formulaire ne marche pas »…).
3. Claude lit les fichiers, les modifie, régénère le site et le vérifie dans son navigateur intégré.
4. Pour le **serveur**, Claude ne se connecte pas lui-même : il donne les commandes **une par une,
   expliquées**, on les tape sur MINI et on lui colle le résultat. C'est volontaire : rien ne
   s'exécute sur le serveur sans être compris.
5. Le fichier `RECAP.md` du projet résume tout l'état du projet : on le donne à Claude au début
   d'une nouvelle conversation pour qu'il reprenne là où on s'était arrêtés.

---

## 3. Le skill `premium-3d-glass`

Un **skill** est un paquet d'instructions et de fichiers qu'on donne à Claude pour une tâche
précise. Celui-ci a été fourni sous forme de zip (`premium-3d-glass-skill.zip`), au début de
la première conversation, avec l'adresse de l'ancien site WordPress.

### Ce que contient le skill

| Élément | Rôle |
|---|---|
| `SKILL.md` | La méthode : interviewer (entreprise, activité, e-mail, logo ou forme), choisir la forme, écrire le contenu, construire, vérifier |
| `template/js/app.js` | Le moteur 3D déjà écrit et testé : verre avec réfraction et dispersion, découpage en 4 fragments, recomposition au scroll |
| `template/css/styles.css` | La mise en page : header, chapitres, cartes en verre, formulaire |
| `template/tools/make_shape.py` | Une bibliothèque de formes 3D (lettres, raquette, serveur, nuage, montagne…) et un vérificateur de logos SVG |
| `template/tools/build_site.py` | Fabrique le site à partir d'un fichier de réglages `.json` |
| `references/` | La documentation du moteur et des formes |

### Ce que le skill produit tel quel

Une **seule page** : un objet 3D, cinq chapitres (Accueil, À propos, Services, Résultats,
Contact) et un formulaire qui ouvre simplement la messagerie du visiteur (`mailto:`).

---

## 4. Ce qui a été modifié par rapport au modèle

Le moteur 3D n'a pas été réécrit : on l'a **rempli** et **étendu**.

### a. Plusieurs pages au lieu d'une

| Page | Fichier de réglages | Objet 3D |
|---|---|---|
| Accueil | `site.json` | la lettre **A** (`letter-a`) |
| Compétences | `competences.json` | un serveur en tour (`server-rack`) |
| Sport | `sport.json` | une raquette fine et cordée (`racket`, épaisseur `0.1`) |
| Tennis, Course, Vélo, Natation, Randonnée | listées dans `sport.json` → `"subpages"` | pas de 3D : pages simples avec récits et galeries |
| Version anglaise | `site-en.json`, `competences-en.json` | mêmes objets |

`build_site.py` a été étendu pour générer toutes ces pages en une fois, avec le même header
(**Accueil · Compétences · Sport · Contact**, suivi du chapitre en cours, bouton **FR | EN**).

### b. Les parties longues

Sous la partie animée, chaque page a une partie « longue » écrite en HTML simple :

- `template/sections/parcours.html` : expérience et formation (accueil) ;
- `template/sections/competences.html` : les 7 catégories de compétences ;
- `template/sections/sport.html` et `template/sections/sport/*.html` : le journal de sport ;
- `template/sections/en/` : les mêmes en anglais.

### c. Le contenu repris de WordPress

- Le dossier `wp-content/uploads` de WordPress a été copié dans `upload/`, puis **trié et
  renommé** dans `template/media/` : `profil`, `cv`, `logo`, `randonnee` (20 photos),
  `velo` (9), `natation` (2), `tennis` (2 vidéos MP4), `course` (vide).
- Chaque photo existe en **deux tailles** : la grande (`bordeaux.jpg`, environ 1500 px) et une
  miniature pour les galeries (`bordeaux-mini.jpg`, 300 px), pour que les pages se chargent vite.
- Les anciennes adresses WordPress (`/competences/`, `/tennis/`, `/contact/`…) redirigent vers
  les nouvelles pages (redirections 301 dans `regles.conf`), pour ne pas perdre Google.

### d. Les réglages d'affichage

- Partie animée sur **4 écrans** de défilement (`.scroll-stage { height: 400vh }`).
- Réticule (curseur personnalisé) **retiré** : souris normale.
- Écran de chargement **retiré** : la 3D apparaît en fondu.
- Accueil : grande photo de profil et 4 bulles **LinkedIn · Wiki · CV · Sport**.

### e. Les outils ajoutés sur le PC

| Fichier | Rôle |
|---|---|
| `mettre-a-jour.bat` | Régénère le site dans `site/` et l'ouvre sur http://127.0.0.1:5180 |
| `preparer-mise-en-ligne.bat` | Régénère et emballe `mise-en-ligne\alexandre-faye-3d.tar.gz` + les fichiers serveur |
| `template/tools/package_site.py` | Crée l'archive avec uniquement les fichiers utiles |
| `TUTO.md` | Aide-mémoire pour modifier le contenu |
| `RECAP.md` | L'état complet du projet (fait foi) |

Chaque page charge ses fichiers avec une **étiquette de version** `?v=<empreinte>` calculée sur
leur contenu : dès qu'un fichier change, Cloudflare et les navigateurs rechargent la nouvelle
version au lieu de garder l'ancienne en cache.

---

## 5. La mise en ligne sur MINI

| Élément | Emplacement sur MINI |
|---|---|
| Le site (lecture seule, root:root, 755/644) | `/var/www/alexandre-faye-3d` |
| Règles du site (sécurité, redirections, cache, formulaire) | `/etc/apache2/alexandre-faye-3d/regles.conf` |
| Site HTTPS actif | `/etc/apache2/sites-available/alexandre-faye-3d-le-ssl.conf` |
| WordPress (intact, désactivé) | `/var/www/alexandre-faye.fr` + `alexandre-faye-le-ssl.conf` |

Retour à WordPress si besoin :

```bash
sudo a2dissite alexandre-faye-3d-le-ssl
sudo a2ensite alexandre-faye-le-ssl
sudo systemctl reload apache2
```

### Les protections en place

- Le dossier du site n'accepte que les lectures (`Require method GET HEAD`) et refuse de servir
  tout script (`.php`, `.sh`, `.py`…) ou fichier caché.
- En-têtes de sécurité : CSP, HSTS (HTTPS obligatoire pendant un an), `X-Frame-Options`,
  `nosniff`, `Referrer-Policy`, `Permissions-Policy`.
- Plus aucun accès au dossier de WordPress.

> **Leçons des incidents** : un `Listen 127.0.0.1:8081` a fait tomber Apache (port déjà pris,
> et `configtest` ne le voit pas) ; un fichier de règles activé comme site sans `<VirtualHost>`
> s'applique à **tous** les sous-domaines. D'où la règle : après toute modification Apache,
> `configtest` → `reload` → `systemctl is-active apache2`.
{.is-warning}

---

## 6. Le formulaire de contact (10/11/2026)

### Le principe

Le modèle ouvrait la messagerie du visiteur (`mailto:`). Il a été remplacé par un vrai envoi :

1. le site envoie le formulaire en **POST** vers `/api/contact` ;
2. Apache lance `contact.php` ;
3. `contact.php` envoie **deux e-mails** par Postfix et le relais OVH : le message pour
   `pro@alexandre-faye.fr`, et un accusé de réception avec la copie du message pour le visiteur ;
4. le site affiche « Merci ! Votre message est bien parti ».

Protections : champ piège invisible (`website`) contre les robots, **3 messages par heure et par
visiteur**, **40 par jour** au total, longueurs bornées, en-têtes d'e-mail nettoyés, et
Cloudflare Turnstile prévu (optionnel).

### Où sont les fichiers, et pourquoi là

| Fichier | Emplacement | Droits | Pourquoi |
|---|---|---|---|
| Programme | `/var/www/alexandre-faye-3d-api/contact.php` | root:root 644 | Hors du site : pas effacé à chaque mise à jour, et le site reste « rien d'exécutable » |
| Réglages | `/etc/alexandre-faye-3d/contact.ini` | root:www-data 640 | `/etc` = réglages ; contiendra la clé secrète Turnstile, jamais téléchargeable |
| Compteurs anti-abus | `/var/lib/alexandre-faye-3d-contact/` | www-data 700 | `/var/lib` = données écrites par un programme ; seul endroit où Apache écrit |

Dans `regles.conf`, un seul point d'entrée, en POST uniquement :

```apache
Alias /api/contact /var/www/alexandre-faye-3d-api/contact.php
<Directory /var/www/alexandre-faye-3d-api>
    Options None
    AllowOverride None
    Require method POST
</Directory>
```

### Les étapes suivies

1. Installer l'archive à jour du site.
2. Copier `contact.php`, `contact.ini`, créer le dossier des compteurs (commandes `install`
   avec les droits ci-dessus).
3. Vérifier que l'adresse d'expéditeur (`expediteur` dans `contact.ini`) est bien celle du
   compte OVH de Postfix, sinon OVH refuse l'envoi :

```bash
sudo cut -d' ' -f2 /etc/postfix/sasl/sasl_passwd | cut -d: -f1
```

4. **Comparer** l'ancien et le nouveau `regles.conf` avant de remplacer :

```bash
sudo cp /etc/apache2/alexandre-faye-3d/regles.conf /etc/apache2/alexandre-faye-3d/regles.conf.avant-formulaire
diff /etc/apache2/alexandre-faye-3d/regles.conf ~/serveur/regles.conf
```

> Le `diff` a montré que le nouveau fichier **supprimait l'en-tête HSTS**
> (`Strict-Transport-Security`), présent sur le serveur mais absent du fichier du PC. Il a été
> rajouté dans `deploiement/regles.conf` avant l'installation. Toujours comparer avant
> d'écraser une configuration !
{.is-warning}

5. Installer `regles.conf`, puis `configtest` → `reload` → `is-active`.
6. Test depuis le vrai site : message « Merci ! », et dans `/var/log/mail.log` deux lignes
   `status=sent` via `ssl0.ovh.net`, envoyées par `uid=33` (www-data, donc bien le formulaire).

---

## 7. Le script de sauvegarde (11/11/2026)

### Le besoin

Faire comme la sauvegarde WordPress existante (`/scripts/alexandre-faye-script.sh`, lancée par
cron) : une archive datée, copiée sur le disque de sauvegarde, gardée 52 jours, avec un e-mail
de compte rendu.

### Ce qui est sauvegardé

Il n'y a pas de base de données. Le script archive ce qui n'existe que sur le serveur :

- `/var/www/alexandre-faye-3d` (le site publié) ;
- `/var/www/alexandre-faye-3d-api` (le formulaire) ;
- `/etc/alexandre-faye-3d` (les réglages du formulaire) ;
- `/etc/apache2/alexandre-faye-3d` et `alexandre-faye-3d-le-ssl.conf` (la configuration Apache).

> La vraie source du site, c'est le **PC** (`D:\Documents\Claude\alexandre-faye-3d\`). Le serveur
> ne contient que le site généré : pense à sauvegarder aussi ce dossier du PC.
{.is-info}

### Ce que fait le script, étape par étape

| Étape | Détail |
|---|---|
| 1. Archive | `tar.gz` (garde propriétaires et droits, contrairement au zip) dans `/home/alexandre/Partage/alexandre-faye-3d/backup/`, lisible par root seulement |
| 2. Vérification | l'archive est relue en entier (`tar -tzf`) |
| 3. Copie | vers `/media/alexandre/sauvegarde-site/alexandre-faye-3d/`, **seulement si le disque est monté**, puis comparaison octet par octet (`cmp`) |
| 4. Nettoyage | suppression des sauvegardes de plus de 52 jours, seulement si tout a réussi |
| 5. Compte rendu | journal clair (résultat en premier, `[OK]` / `[ECHEC]` par étape, tailles, espace disque, liste des sauvegardes) envoyé par e-mail : « Sauvegarde site 3D : OK » ou « ECHEC » |

### Les corrections en cours de route

1. **Première version** : la copie visait `/media/alexandre/sauvegarde-wordpress` et la nuit
   suivante l'e-mail disait « disque de sauvegarde non branché ».
2. On a renommé la destination en `sauvegarde-site`, et **réécrit le journal**, qui n'était pas
   assez clair (on ne voyait pas ce qui avait réussi ou non).
3. Toujours « non monté » : `/etc/fstab` a montré que le disque NTFS est monté sur
   **`/media/alexandre` lui-même**. `sauvegarde-site` n'est qu'un dossier de ce disque. Le script
   vérifie donc maintenant `mountpoint /media/alexandre`.
4. Résultat : e-mail « RÉUSSIE », archive de 76 Mo, 126 éléments, copie identique.

Au passage, deux défauts ont été repérés dans l'ancien script WordPress : l'e-mail d'échec
ne peut jamais partir (adresse mal écrite avec un « à », et les `&&` sautent tout le bloc en cas
d'erreur), et le mot de passe MySQL est écrit en clair.

### Installation

```bash
sudo install -m 700 -o root -g root ~/serveur/alexandre-faye-3d-sauvegarde.sh /scripts/alexandre-faye-3d-sauvegarde.sh
```

Ligne ajoutée dans `sudo crontab -e` (3 h 30 les 1, 11, 21 et 31 du mois) :

```
# sauvegarde du site 3D alexandre-faye.fr (site + formulaire)
30 3 */10 * * /bin/bash /scripts/alexandre-faye-3d-sauvegarde.sh
```

Restauration :

```bash
sudo tar -xzf /home/alexandre/Partage/alexandre-faye-3d/backup/alexandre-faye-3d-[DATE].tar.gz -C /
```

### `nofail` sur le disque de sauvegarde

Sans cette option, si le disque NTFS ne répond pas au démarrage, MINI reste bloqué en mode de
secours (plus de site, plus de SSH). La ligne de `/etc/fstab` est devenue :

```
UUID=62EA3B04EA3AD44F /media/alexandre/ ntfs defaults,nofail 0 0
```

Vérifiée avec `sudo findmnt --verify` (0 erreur), puis `sudo systemctl daemon-reload`. Ancienne
version gardée dans `/etc/fstab.avant-nofail`.

## 8. Restaurer le site

### Quelle sauvegarde utiliser ?

Le site existe à **quatre endroits**. On prend toujours le plus proche qui fonctionne :

| Situation | Source | Section |
|---|---|---|
| MINI marche, le site est cassé (mauvaise mise à jour, fichier supprimé…) | `Partage/alexandre-faye-3d/backup/` sur MINI | **8.1** |
| MINI marche, mais `Partage` a disparu (comme le 12/11/2026) | le disque NTFS : `/media/alexandre/sauvegarde-site/alexandre-faye-3d/` | **8.2** |
| **MINI est mort, volé ou inaccessible** | **Cloudflare R2**, chiffré (copie tous les 3 mois) | **8.3** |
| Plus de serveur du tout, il faut que le site soit en ligne **tout de suite** | le dossier du **PC** ou l'archive R2, envoyés sur **Cloudflare Pages** | **8.4** |

> Ce que contient chaque archive `alexandre-faye-3d-AAAA-MM-JJ.tar.gz` : le site
> (`/var/www/alexandre-faye-3d`), le formulaire (`/var/www/alexandre-faye-3d-api`), ses réglages
> (`/etc/alexandre-faye-3d`, avec la clé secrète Turnstile) et la configuration Apache du site.
> Les propriétaires et les droits sont conservés : on extrait depuis `/` et tout retrouve sa place.
{.is-info}

> **La source du site, c'est le PC** (`D:\Documents\Claude\alexandre-faye-3d\`). Si le site
> publié est simplement abîmé, le plus simple reste souvent de **refaire une mise en ligne**
> depuis le PC (page « Mettre à jour le site en ligne »). Les sections ci-dessous servent quand
> le PC n'est pas disponible, ou quand il faut aussi récupérer le formulaire et Apache.
{.is-info}

---

### 8.1 Depuis la sauvegarde sur MINI (le cas courant)

**1. Lister les sauvegardes** (la plus récente en bas) :
```bash
sudo ls -lh /home/alexandre/Partage/alexandre-faye-3d/backup/
```

**2. Vérifier celle qu'on veut** (remplace la date). Une liste de fichiers doit défiler :
```bash
sudo tar -tzf /home/alexandre/Partage/alexandre-faye-3d/backup/alexandre-faye-3d-AAAA-MM-JJ.tar.gz | head -n 20
```

**3. Mettre la version actuelle de côté** (c'est le retour arrière si la restauration ne
convient pas) :
```bash
sudo mv /var/www/alexandre-faye-3d /var/www/alexandre-faye-3d.avant-restauration
```

**4. Restaurer.** Tout reprend sa place (site, formulaire, réglages, Apache) :
```bash
sudo tar -xzf /home/alexandre/Partage/alexandre-faye-3d/backup/alexandre-faye-3d-AAAA-MM-JJ.tar.gz -C /
```
Pour ne restaurer **que les pages du site**, sans toucher au formulaire ni à Apache, on ajoute le
chemin à la fin de la commande :
```bash
sudo tar -xzf /home/alexandre/Partage/alexandre-faye-3d/backup/alexandre-faye-3d-AAAA-MM-JJ.tar.gz -C / var/www/alexandre-faye-3d
```

**5. Vérifier Apache, puis recharger** (les trois commandes, dans l'ordre) :
```bash
sudo apache2ctl configtest
```
```bash
sudo systemctl reload apache2
```
```bash
systemctl is-active apache2
```
`Syntax OK`, puis `active`.

**6. Vider le cache Cloudflare** (Caching → Configuration → **Purge Everything**), puis ouvrir
https://alexandre-faye.fr avec **Ctrl+F5**. Tester aussi le formulaire.

**7. Si tout va bien**, supprimer l'ancienne version mise de côté (chemin complet, jamais `*`) :
```bash
sudo rm -rf /var/www/alexandre-faye-3d.avant-restauration
```

> **Retour arrière** (la restauration ne convient pas) :
> `sudo rm -rf /var/www/alexandre-faye-3d` puis
> `sudo mv /var/www/alexandre-faye-3d.avant-restauration /var/www/alexandre-faye-3d`, puis l'étape 5.
{.is-warning}

---

### 8.2 Depuis le disque de sauvegarde NTFS

Même procédure que 8.1, seul le dossier change. D'abord vérifier que le disque est bien monté :
```bash
mountpoint /media/alexandre
```
Réponse attendue : `/media/alexandre is a mountpoint`. Puis :
```bash
ls -lh /media/alexandre/sauvegarde-site/alexandre-faye-3d/
```
Et les étapes 2 à 7 de la section 8.1, avec ce chemin à la place de
`/home/alexandre/Partage/alexandre-faye-3d/backup/`.

---

### 8.3 Depuis Cloudflare R2, quand MINI n'est plus là

Le scénario : MINI ne démarre plus (ou a été volé), on repart d'une **machine neuve sous Ubuntu
24.04**, branchée sur la Freebox.

#### Ce qu'il faut avoir sous la main

| Quoi | Où le trouver si MINI est mort |
|---|---|
| L'*Access Key ID*, la *Secret Access Key* et l'*endpoint* R2 (avec `.eu.`) | Bitwarden. Le serveur Bitwarden était **sur MINI** : utilise l'application du **téléphone ou du PC**, qui garde une copie hors ligne du coffre. À défaut, crée un nouveau jeton dans Cloudflare (R2 → Gérer les jetons d'API R2) |
| Les **deux mots de passe crypt** (`password` et `password2`) | Bitwarden (application hors ligne) **ou le papier**. ⚠️ Sans eux, **rien n'est récupérable**, pas même par Cloudflare |
| Un accès au tableau de bord Cloudflare | e-mail + code reçu (One-time PIN) |

#### a. Installer rclone (version officielle) et retrouver le coffre chiffré

```bash
cd /tmp
```
```bash
wget https://downloads.rclone.org/rclone-current-linux-amd64.deb
```
```bash
sudo apt install ./rclone-current-linux-amd64.deb
```
```bash
sudo mkdir -p /root/.config/rclone
```
```bash
sudo vi /root/.config/rclone/rclone.conf
```
Contenu (les 3 valeurs viennent de Bitwarden) :
```ini
[r2]
type = s3
provider = Cloudflare
access_key_id = TON_ACCESS_KEY_ID
secret_access_key = TA_SECRET_ACCESS_KEY
endpoint = https://TON_ID.eu.r2.cloudflarestorage.com
acl = private
no_check_bucket = true
```
```bash
sudo chmod 600 /root/.config/rclone/rclone.conf
```
Puis la couche de déchiffrement, **avec exactement les mêmes réglages qu'à la création** :
```bash
sudo rclone config
```
| Question | Réponse |
|---|---|
| `n/s/q>` | `n` |
| `name>` | `r2chiffre` |
| `Storage>` | `crypt` |
| `remote>` | `r2:sauvegardes-mini` |
| `filename_encryption>` | `1` (standard) |
| `directory_name_encryption>` | `2` (false) |
| `password` | **`y`** (taper son propre mot de passe), puis coller le **1er** mot de passe |
| `password2` | **`y`**, puis coller le **2e** mot de passe |
| `Edit advanced config?` | `n` |
| `Keep this remote?` | `y`, puis `q` |

> Ici, on répond `y` et pas `g` : on ne **génère** pas de nouveaux mots de passe, on **retape**
> ceux de la création. Avec d'autres mots de passe, les fichiers resteraient illisibles.
{.is-danger}

**Vérifier qu'on voit les vrais noms :**
```bash
sudo rclone ls r2chiffre:auto/
```
On doit lire `alexandre-faye-3d-AAAA-MM-JJ.tar.gz` et `essentiel-AAAA-MM-JJ_HHhMM.tar.gz`. Si la
liste est vide ou si les noms sont illisibles, ce sont les mots de passe crypt qui sont faux.

#### b. Télécharger et vérifier

```bash
sudo mkdir -p /root/restauration
```
```bash
sudo rclone copy r2chiffre:auto/ /root/restauration/ --include "alexandre-faye-3d-*" --progress
```
```bash
sudo ls -lh /root/restauration/
```
```bash
sudo tar -tzf /root/restauration/alexandre-faye-3d-AAAA-MM-JJ.tar.gz | head -n 20
```
(Une seule archive à la fois dans `tar -tzf` : avec `*` et plusieurs fichiers, tar répond
« non trouvé dans l'archive ».)

#### c. Remettre Apache et PHP, puis le site

```bash
sudo apt update
```
```bash
sudo apt install apache2 php libapache2-mod-php certbot python3-certbot-apache
```
```bash
sudo tar -xzf /root/restauration/alexandre-faye-3d-AAAA-MM-JJ.tar.gz -C /
```
```bash
sudo a2ensite alexandre-faye-3d-le-ssl
```
```bash
sudo apache2ctl configtest
```
Si `configtest` signale une commande inconnue (`Invalid command 'Header'`, `RewriteEngine`…), il
manque un module : `sudo a2enmod headers` (ou `rewrite`, `ssl`, `expires`), puis on relance
`configtest`, jusqu'à `Syntax OK`.

#### d. Le certificat HTTPS, le réseau et le DNS

Les certificats Let's Encrypt (`/etc/letsencrypt`) **ne sont pas** dans l'archive : on les refait.

1. **Freebox** : rediriger les ports **80** et **443** vers l'adresse locale de la nouvelle
   machine.
2. **Cloudflare → DNS** : si l'IP publique a changé, mettre la nouvelle dans l'enregistrement `A`
   de `alexandre-faye.fr` (proxy orange **activé**).
3. Le certificat :
   ```bash
   sudo certbot --apache -d alexandre-faye.fr -d www.alexandre-faye.fr
   ```
4. Puis :
   ```bash
   sudo systemctl reload apache2
   ```
   ```bash
   systemctl is-active apache2
   ```
5. **Purge Everything** dans Cloudflare, et test de https://alexandre-faye.fr avec Ctrl+F5.

#### e. Ce qui ne revient pas tout seul

| Élément | Pourquoi | Que faire |
|---|---|---|
| L'envoi des e-mails du formulaire | La configuration Postfix et le relais OVH ne sont pas dans les archives | Réinstaller Postfix avec le relais OVH (le mot de passe est dans Bitwarden) ; en attendant, le site s'affiche mais le formulaire répond « erreur » |
| Le pare-feu, SSH, fail2ban, Samba | Ils sont dans l'archive **essentiel** (`/etc/ufw`, `/etc/ssh/sshd_config.d`, `/etc/fail2ban/jail.d`, `smb.conf`) | La télécharger de la même façon (`--include "essentiel-*"`) et remettre ces fichiers **un par un**, en relisant la page « Sécuriser le serveur MINI » |
| Bitwarden, le wiki, Portainer | Dans l'archive **essentiel**, avec la dernière sauvegarde de la base Bitwarden (`.BAK`) | Hors du sujet de cette page : Docker à réinstaller d'abord |

---

### 8.4 En urgence, sans aucun serveur : Cloudflare Pages

Le site est **statique** : Cloudflare peut l'héberger lui-même, gratuitement, en quelques minutes,
sans serveur. C'est la solution « le site doit revenir en ligne ce soir ».

1. **Récupérer le site** :
   - depuis le **PC** : double-clic sur `mettre-a-jour.bat`, le site complet est dans
     `D:\Documents\Claude\alexandre-faye-3d\site\` ;
   - ou depuis **R2** (sur n'importe quel ordinateur avec rclone, configuré comme en 8.3 a) :
     télécharger l'archive, puis n'en garder que le site :
     ```bash
     sudo tar -xzf /root/restauration/alexandre-faye-3d-AAAA-MM-JJ.tar.gz -C /root/restauration var/www/alexandre-faye-3d
     ```
     Le site est alors dans `/root/restauration/var/www/alexandre-faye-3d/`.
   > **Limite de Pages : 25 Mo par fichier.** Deux vidéos la dépassent :
   > `media/tennis/saint-raphael.mp4` (33 Mo) et `media/tennis/coucher-de-soleil.mp4` (26 Mo).
   > **Supprime-les de la copie à envoyer** (pas de l'original !), sinon le chargement est refusé.
   > La page Tennis s'affichera, sans ces deux vidéos.
   {.is-warning}
2. **Cloudflare → Workers & Pages → Créer → Pages → Charger des ressources** (*Upload assets*) :
   nom du projet `alexandre-faye`, puis glisser **le dossier du site** (celui qui contient
   `index.html`).
3. Le site s'ouvre sur une adresse `alexandre-faye.pages.dev` : vérifier qu'il s'affiche.
4. Onglet **Domaines personnalisés** → **Configurer un domaine** → `alexandre-faye.fr`.
   Cloudflare remplace l'enregistrement DNS qui pointait vers MINI.

| Ce qui marche | Ce qui ne marche pas |
|---|---|
| Toutes les pages, la 3D, les photos, le CV, FR et EN, les vidéos de moins de 25 Mo | Le **formulaire** (il a besoin de PHP sur un serveur) : le lien e-mail `pro@alexandre-faye.fr` reste utilisable |
| HTTPS, automatiquement | Les redirections des anciennes adresses WordPress (dans `regles.conf`, sur Apache) |

> **Retour à MINI** quand il est réparé : supprimer le domaine personnalisé dans le projet Pages,
> puis remettre l'enregistrement `A` vers l'IP de la Freebox (proxy orange), et Purge Everything.
{.is-info}

---

### 8.5 Tester la restauration (tous les 6 mois)

Une sauvegarde qu'on n'a jamais restaurée n'est pas une sauvegarde. Test sans risque, dans un
dossier temporaire :
```bash
sudo mkdir -p /tmp/test-r2
```
```bash
sudo rclone copy r2chiffre:auto/ /tmp/test-r2/ --include "alexandre-faye-3d-*"
```
```bash
ls /tmp/test-r2/
```
```bash
sudo mkdir /tmp/test-r2/extrait
```
```bash
sudo tar -xzf /tmp/test-r2/alexandre-faye-3d-AAAA-MM-JJ.tar.gz -C /tmp/test-r2/extrait var/www/alexandre-faye-3d/index.html
```
```bash
sudo diff /var/www/alexandre-faye-3d/index.html /tmp/test-r2/extrait/var/www/alexandre-faye-3d/index.html && echo IDENTIQUE
```
`IDENTIQUE` : la chaîne complète marche (MINI → chiffrement → Cloudflare → téléchargement →
déchiffrement → archive). Si le site a été mis à jour depuis la dernière archive, une différence
est normale : on teste alors un fichier qui ne change pas (`media/logo/logo.png`, avec `cmp`).
```bash
sudo rm -rf /tmp/test-r2
```
