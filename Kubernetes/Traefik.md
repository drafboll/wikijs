---
title: Traefik
description: 
published: 1
date: 2026-09-25T15:41:16.754Z
tags: 
editor: markdown
dateCreated: 2025-05-08T18:07:06.447Z
---

## Qu’est-ce qu’un Ingress ?

Un **Ingress** est une ressource Kubernetes qui définit comment exposer des services HTTP/HTTPS depuis l’extérieur vers vos services **ClusterIP** internes.  
- **Ingress Controller** : un composant (pod+service) qui lit ces objets Ingress et configure un reverse-proxy pour router le trafic.  
- **Ingress** = règle (hostname → service/port, éventuellement TLS, réécritures, middlewares, etc.).
 
### Pourquoi utiliser Traefik ?

[Traefik](https://traefik.io/) est un Ingress Controller « tout-en-un » qui :
- **Découvre** automatiquement vos Ingress et Services via l’API Kubernetes.  
- Gère les **routages** HTTP, path-based ou host-based.  
- Peut intégrer en natif un **cert-resolver** Let’s Encrypt (non utilisé ici, car Caddy assure TLS).  
- Propose un **dashboard** et des métriques intégrées.  
- Se couple facilement à des CRD pour des middlewares, mais fonctionne aussi avec du simple Ingress standard.

Dans notre architecture, Traefik sera chargé de **router** tout le trafic interne HTTP (port 31080) vers :
- WordPress  
- Longhorn UI  
- Zabbix  
…  
pendant que **Caddy** en façade prendra en charge le TLS et exposera les noms de domaine publics.

---

## Tutoriel d’installation

### 1. Installer Traefik (HTTP only)

```bash
# 1. Ajouter le repo et le mettre à jour
helm repo add traefik https://traefik.github.io/charts
helm repo update
```

```bash
# 2. Installer Traefik en NodePort (HTTP uniquement)
helm install traefik traefik/traefik \
  --namespace traefik --create-namespace \
  --set service.type=NodePort \
  --set service.nodePorts.web=31080 \
  --set service.ports.websecure.enabled=false \
  --set ingressClass.enabled=true \
  --set ingressClass.isDefaultClass=true

# Vérifier
kubectl get pods,svc -n traefik
kubectl get ingressclass
```

Verifier ensuite si le port est bien le bon :
```bash
kubectl -n traefik get svc traefik
```
tu dois voir : 80:31080/TCP,443:0/TCP

Sinon :
```bash
kubectl -n traefik patch svc traefik --type='json' -p='[
  {
    "op": "replace",
    "path": "/spec/ports/0/nodePort",
    "value": 31080
  }
]'
```




