---
title: Installation de passbolt sur kubernetes
description: 
published: 1
date: 2026-09-24T18:26:24.888Z
tags: kubernetes, passbolt
editor: markdown
dateCreated: 2025-05-05T13:48:19.438Z
---


# Déploiement de Passbolt Community Edition sur Kubernetes avec Helm et Longhorn

Ce guide détaille étape par étape l'installation de **Passbolt CE** sur un cluster Kubernetes, avec stockage persistant géré par **Longhorn** et exposition via **Caddy** en tant que reverse proxy.
 
---

## Prérequis

- Un cluster Kubernetes fonctionnel (avec `kubectl` et `helm`)
- Helm 3.x installé : `helm version`
- Longhorn installé et actif (avec une `StorageClass` par défaut nommée `longhorn`)
- Un sous-domaine pointant vers ton cluster, par exemple : `passbolt.alpha-pro.fr`
- Un reverse proxy (comme **Caddy**) déjà configuré
- Une adresse email pour les notifications (facultatif)

---

## Étape 1 : Ajouter le dépôt Helm Passbolt

```bash
helm repo add passbolt https://charts.passbolt.com/
helm repo update
```

> Ceci permet de récupérer le chart officiel de Passbolt.

---

## Étape 2 : Préparer un fichier de configuration `values.yaml`

Crée un fichier nommé `passbolt-values.yaml` contenant :

```yaml
image:
  repository: passbolt/passbolt
  tag: latest
  pullPolicy: IfNotPresent

env:
  APP_FULL_BASE_URL: "https://passbolt.alpha-pro.fr"
  EMAIL_DEFAULT_FROM: "mail@alpha-pro.fr"
  EMAIL_TRANSPORT_DEFAULT_HOST: "mail.alpha-pro.fr"
  EMAIL_TRANSPORT_DEFAULT_PORT: 587
  EMAIL_TRANSPORT_DEFAULT_USERNAME: "passbolt@alpha-pro.fr"
  EMAIL_TRANSPORT_DEFAULT_PASSWORD: "mdp"
  EMAIL_TRANSPORT_DEFAULT_TLS: true
  DATASOURCES_DEFAULT_HOST: "passbolt-mariadb"
  DATASOURCES_DEFAULT_USERNAME: "passbolt"
  DATASOURCES_DEFAULT_PASSWORD: "passbolt"
  DATASOURCES_DEFAULT_DATABASE: "passboltdb"

mariadb:
  enabled: true
  auth:
    rootPassword: "rootpass"
    database: "passboltdb"
    username: "passbolt"
    password: "passbolt"
  primary:
    persistence:
      enabled: true
      storageClass: "longhorn"
      size: 1Gi

persistence:
  enabled: true
  accessModes:
    - ReadWriteOnce
  storageClass: "longhorn"
  size: 2Gi

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: false
```

---

## Étape 3 : Créer un namespace dédié

```bash
kubectl create namespace passbolt
kubectl config set-context --current --namespace=passbolt
```

> Cela isole les ressources Passbolt dans leur propre espace.

---

## Étape 4 : Installer Passbolt avec Helm

```bash
helm install passbolt passbolt/passbolt   --namespace passbolt   --values passbolt-values.yaml
```

---

## Étape 5 : Vérifier les pods déployés

```bash
kubectl get pods -n passbolt
```

Tu dois voir deux pods en `Running` :
- `passbolt-...` (application)
- `passbolt-mariadb-...` (base de données)

---

## Étape 6 : Exposer Passbolt avec Caddy

Ajoute la configuration suivante à ton `Caddyfile` :

```caddyfile
passbolt.alpha-pro.fr {
  reverse_proxy http://<IP_NODE>:<NODE_PORT> {
    transport http {
      tls_insecure_skip_verify
    }
  }
}
```

> Tu peux aussi utiliser un LoadBalancer ou un Ingress Controller si tu préfères.

---

## Étape 7 : Initialiser Passbolt

1. Accède à `https://passbolt.alpha-pro.fr`
2. Suis la procédure de configuration initiale (clé GPG, admin, etc.)

---

## Étape 8 : Vérifications et gestion

- Vérifie les PVC :
  ```bash
  kubectl get pvc -n passbolt
  ```
- Vérifie les volumes créés dans l’interface web de **Longhorn**
- Accède à Passbolt et crée des groupes, utilisateurs, mots de passe partagés

---

## Recommandations

- Configure un SMTP fonctionnel pour recevoir les invitations et alertes
- Active la double authentification (MFA)
- Planifie des sauvegardes régulières de la base et du volume principal

---

## Références

- [Documentation officielle Passbolt](https://www.passbolt.com/help/)
- [Chart Helm Passbolt](https://github.com/passbolt/passbolt-helm-chart)
- [Reverse proxy Caddy](https://caddyserver.com/docs/)

