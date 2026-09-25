---
title: Installation de Caddy
description: 
published: 1
date: 2026-09-25T15:47:26.986Z
tags: 
editor: markdown
dateCreated: 2025-05-02T07:40:38.672Z
---

# Tutoriel : Installer et configurer Caddy sur Rocky Linux

Ce guide complet vous explique :
1. **Qu’est-ce que Caddy ?**  
2. **Installation** sur Rocky Linux  
3. **Configuration de base** (Caddyfile)  
4. **Reverse-proxy + Let’s Encrypt** pour exposer une application interne  
5. **Commandes utiles**  
6. **Options avancées**
 
---

## 1. Qu’est-ce que Caddy ?

Caddy est un serveur web moderne et sécurisé, écrit en Go, qui :
- Gère **automatiquement** les certificats TLS via Let’s Encrypt.  
- Supporte HTTP/2 et HTTP/3 par défaut.  
- Utilise un fichier **Caddyfile** simple, inspiré de Markdown, pour configurer sites statiques, reverse-proxy, PHP, etc.  
- Dispose de **modules** pour authentification, redirection, filtrage, etc.

---

## 2. Installation sur Rocky Linux

### 2.1 Prérequis  
- Rocky Linux 8 ou 9  
- Accès root/sudo  
- Connexion Internet pour les dépôts

### 2.2 Activer le dépôt COPR officiel  
```bash
sudo dnf install -y 'dnf-command(copr)'
sudo dnf copr enable -y @caddy/caddy
```

### 2.3 Installer Caddy  
```bash
sudo dnf install -y caddy
```

### 2.4 Démarrer et activer  
```bash
sudo systemctl enable --now caddy
sudo systemctl status caddy
caddy version
```

---

## 3. Configuration de base : le Caddyfile

Le fichier de configuration se trouve dans `/etc/caddy/Caddyfile`.  
Chaque bloc commence par le(s) nom(s) de domaine ou l’adresse à écouter, suivi d’accolades.

### 3.1 Exemple : site statique  
```caddyfile
:80 {
  root * /usr/share/caddy
  file_server
}
```

---

## 4. Reverse-proxy & Let’s Encrypt

Pour exposer une application interne (ex. Graylog) en HTTPS public :

### 4.1 Préparer le réseau  
1. DNS :  
   ```
   graylog.alpha-pro.fr.  A  votre_IP_publique
   ```  
2. Routeur / UniFi : forward ports **80** et **443** vers la machine Caddy.

### 4.2 Écrire le Caddyfile  
```caddyfile
{
  # Email pour ACME (Let’s Encrypt)
  email administrateur@alpha-pro.fr
}

graylog.alpha-pro.fr {
  reverse_proxy http://192.168.30.30:9000
}
```
- Caddy effectue un **HTTP-01 challenge** sur le port 80, obtient le certif, puis sert en HTTPS.  
- Le backend Graylog reste en HTTP sur le port 9000.

---

## 5. Commandes utiles

| Action                          | Commande                                                       |
|---------------------------------|----------------------------------------------------------------|
| Formater le Caddyfile           | `sudo caddy fmt --overwrite /etc/caddy/Caddyfile`             |
| Recharger la config sans arrêt  | `sudo caddy reload --config /etc/caddy/Caddyfile`             |
| Voir le statut du service       | `sudo systemctl status caddy`                                 |
| Consulter les logs (dernière h) | `sudo journalctl -u caddy --no-pager --since "1 hour ago"`    |

---

## 6. Options avancées

Pour des logs plus avancés, préférez un fichier `/etc/logrotate.d/caddy` plutôt que la rotation interne.

---

> **Félicitations !**  
> Vous avez maintenant un serveur Caddy installé, configuré pour servir des sites statiques ou faire du reverse-proxy TLS avec Let’s Encrypt sur Rocky Linux.  
> Copiez-collez ce tutoriel dans votre éditeur de notes et lancez-vous !
