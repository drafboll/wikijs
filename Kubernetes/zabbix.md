---
title: Installer Zabbix sur kubernetes
description: 
published: 1
date: 2026-09-24T18:26:35.572Z
tags: kubernetes, zabbix
editor: markdown
dateCreated: 2025-05-05T14:30:22.779Z
---

# Déploiement de Zabbix sur Kubernetes avec Longhorn et Reverse Proxy Caddy
 
## Objectifs

Ce guide a pour but d’installer **Zabbix** sur un cluster Kubernetes avec :

- Un **stockage persistant** via Longhorn
- Une **exposition web via un reverse proxy Caddy** (en DMZ)
- Une sécurité réseau centralisée (pas d'ouverture directe de port sur les workers)
- Utilisation du domaine : `http://zabbix.alpha-pro.fr`
- Utilisation du port : `30777` pour exposition en NodePort interne, proxifié via Caddy

---

## 1. Pré-requis

- Cluster Kubernetes fonctionnel
- Helm installé (`helm version`)
- Longhorn installé et configuré (avec `StorageClass: longhorn`)
- Serveur Caddy déjà configuré dans ta DMZ (IP ex: `192.168.20.100`)
- DNS `zabbix.alpha-pro.fr` pointant vers l'IP publique de ton Caddy

---

## 2. Créer un PersistentVolumeClaim pour Zabbix

Créer un fichier `zabbix-pvc.yaml` :

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: zabbix-pvc
  namespace: zabbix
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: longhorn
```

Appliquer le PVC :

```bash
kubectl create namespace zabbix
kubectl apply -f zabbix-pvc.yaml
```

---

## 3. Ajouter le repo Helm Bitnami

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
```

---

## 4. Créer le fichier `values.yaml` pour personnaliser l'installation

Créer un fichier `values.yaml` :

```yaml
zabbix:
  enabled: true
  service:
    type: NodePort
    nodePort: 30777

  persistence:
    enabled: true
    existingClaim: zabbix-pvc

postgresql:
  enabled: true
  auth:
    postgresPassword: zabbix123
    username: bn_zabbix
    password: zabbix123
    database: bitnami_zabbix

  primary:
    persistence:
      enabled: true
      storageClass: longhorn
      size: 10Gi
```

---

## 5. Installer Zabbix avec Helm

```bash
helm install zabbix bitnami/zabbix -n zabbix -f values.yaml
```

---

## 6. Configurer Caddy comme reverse proxy

Sur ton serveur **Caddy** (IP ex: `192.168.20.100`), édite le fichier :

```bash
sudo nano /etc/caddy/Caddyfile
```

Ajouter :

```caddyfile
http://zabbix.alpha-pro.fr {
    reverse_proxy http://192.168.40.10:30777
}
```

Puis recharger Caddy :

```bash
sudo caddy fmt --overwrite /etc/caddy/Caddyfile
sudo caddy reload --config /etc/caddy/Caddyfile
```

---

## 7. Vérification

- Accéder à l’interface web : http://zabbix.alpha-pro.fr
- Identifiants par défaut :
  - 🧑 Utilisateur : `Admin`
  - 🔐 Mot de passe : `zabbix`

---

## Conseils & Remarques

- 🔁 Tu peux changer `30777` si tu as un conflit avec un autre service NodePort.
- 🧱 Tu peux ajouter des règles de firewall sur le serveur Caddy si nécessaire.
- 🔐 Pour activer HTTPS avec Caddy, il suffit d’utiliser ton domaine et laisser Caddy gérer le certificat (via Let's Encrypt).

---

## Stockage requis

- 10Gi pour Zabbix suffit pour des tests ou petite infra
- En production : prévoir au minimum 20Gi (logs, historiques, graphs, etc.)

---
