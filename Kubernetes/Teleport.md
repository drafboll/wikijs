---
title: Installation de Teleport
description: 
published: 1
date: 2026-09-24T18:26:16.740Z
tags: teleport
editor: markdown
dateCreated: 2025-05-02T08:00:36.978Z
---


# Déploiement de Teleport Bastion sur Kubernetes avec Helm et Caddy
 
Ce guide explique comment installer **Teleport Community Edition** sur Kubernetes avec **stockage persistant via Longhorn** et exposition HTTPS via un reverse proxy **Caddy**.

---

## Prérequis

- Un domaine configuré, ex : `teleport.alpha-pro.fr`
- Un cluster Kubernetes opérationnel (K8s v1.17+)
- Helm 3.x installé (`helm version`)
- PersistentVolume ou StorageClass dynamique (ex : **Longhorn**)
- Reverse proxy en place (Caddy)
- Adresse mail pour le certificat Let's Encrypt

---

## Étape 1 : Ajouter le dépôt Helm Teleport

```bash
helm repo add teleport https://charts.releases.teleport.dev
helm repo update
```

---

## Étape 2 : Créer un namespace dédié

```bash
kubectl create namespace teleport-cluster
kubectl label namespace teleport-cluster 'pod-security.kubernetes.io/enforce=baseline'
kubectl config set-context --current --namespace=teleport-cluster
```

---

## Étape 3 : Créer un fichier de configuration `teleport-cluster-values.yaml`

Crée un fichier nommé `teleport-cluster-values.yaml` :

```yaml
clusterName: teleport.alpha-pro.fr
proxyListenerMode: multiplex
acme: true
acmeEmail: administrateur@alpha-pro.fr
persistence:
  enabled: true
  size: 5Gi
  storageClass: longhorn
```

>  Le champ `clusterName` doit correspondre au domaine de ton cluster Teleport.

---

## Étape 4 : Installer Teleport via Helm

```bash
helm install teleport-cluster teleport/teleport-cluster   --version 17.4.7   --namespace teleport-cluster   --values teleport-cluster-values.yaml
```

---

## Étape 5 : Obtenir l’adresse externe du service

```bash
kubectl get svc teleport-cluster
```

Repère l'**EXTERNAL-IP** ou le **hostname** (si fourni par ton cloud provider).

---

## Étape 6 : Ajouter la configuration dans Caddy

Dans ton `Caddyfile`, ajoute :

```caddyfile
teleport.alpha-pro.fr {
  reverse_proxy https://<EXTERNAL-IP>:443 {
    transport http {
      tls_insecure_skip_verify
    }
  }
}
```

> Caddy fera office de reverse proxy HTTPS vers Teleport.

---

## Étape 7 : Créer un utilisateur local

1. Crée un fichier `member.yaml` :

```yaml
kind: role
version: v7
metadata:
  name: member
spec:
  allow:
    kubernetes_groups: ["system:masters"]
    kubernetes_labels:
      '*': '*'
    kubernetes_resources:
      - kind: '*'
        namespace: '*'
        name: '*'
        verbs: ['*']
```

2. Applique le rôle :

```bash
kubectl exec -i deployment/teleport-cluster-auth -- tctl create -f - < member.yaml
```

3. Crée un utilisateur :

```bash
kubectl exec -ti deployment/teleport-cluster-auth -- tctl users add myuser --roles=member,access,editor
```

> Un lien d’invitation temporaire sera généré pour finaliser la configuration dans l’interface web.

---

## Étape 8 : Connexion au cluster avec `tsh`

1. Installe le client `tsh` v17.4.7+ sur ta machine.
2. Connecte-toi :

```bash
tsh login --proxy=teleport.alpha-pro.fr:443 --user=myuser
```

3. Liste les clusters Kubernetes :

```bash
tsh kube ls
```

4. Connecte-toi au cluster :

```bash
KUBECONFIG=$HOME/teleport-kubeconfig.yaml tsh kube login teleport.alpha-pro.fr
KUBECONFIG=$HOME/teleport-kubeconfig.yaml kubectl get pods
```

---

## Vérifications

- Pods Teleport doivent être en `Running` :
```bash
kubectl get pods -n teleport-cluster
```

- Accès Web sur `https://teleport.alpha-pro.fr` doit fonctionner

---

## Étapes suivantes

- Intégrer l’authentification SSO si nécessaire
- Gérer les accès RBAC finement via les rôles Teleport
- Ajouter d’autres clusters, bases de données ou serveurs SSH à ton infrastructure Teleport

---

## Références

- [Documentation officielle Teleport](https://goteleport.com/docs/)
- [Chart Helm Teleport](https://github.com/gravitational/teleport/tree/master/examples/chart)
- [Guide TLS officiel](https://graylog.org/post/how-to-guide-securing-graylog-with-tls/)
