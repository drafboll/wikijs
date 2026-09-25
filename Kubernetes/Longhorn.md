---
title: Longhorn
description: 
published: 1
date: 2026-09-25T15:40:55.104Z
tags: kubernetes, longhorn
editor: markdown
dateCreated: 2025-05-02T07:45:05.900Z
---

# Guide d'installation de Longhorn avec Reverse Proxy Caddy

## Qu'est-ce que Longhorn ?

Longhorn est une solution de stockage distribué pour Kubernetes. Elle permet de créer des volumes persistants répliqués automatiquement entre plusieurs nœuds.
 
**Fonctionnalités principales :**

* Réplication des volumes pour la tolérance aux pannes
* Snapshots, backups, restauration
* Interface web de gestion
* Compatible CSI/Kubernetes

##  Prérequis

* Un cluster Kubernetes fonctionnel (3 nœuds recommandés)
* Accès Internet depuis les nœuds
* 3 nœuds avec un espace disque libre non utilisé pour Longhorn
* `curl`, `open-iscsi`, `nfs-common` installés sur chaque nœud (via apt ou dnf/yum)

```bash
# Debian/Ubuntu
sudo apt update
sudo apt install -y open-iscsi nfs-common curl

# Rocky/RedHat/CentOS
sudo dnf install -y iscsi-initiator-utils nfs-utils curl
```

## Installation de Longhorn via `kubectl`

```bash
kubectl apply -f https://raw.githubusercontent.com/longhorn/longhorn/v1.8.1/deploy/longhorn.yaml
```

### Suivre le déploiement

```bash
kubectl get pods -n longhorn-system -w
```

### Accéder à l'interface Longhorn

Longhorn expose une interface web sur le service : `longhorn-frontend`

```bash
kubectl get svc -n longhorn-system
```

Par défaut, le service est de type `ClusterIP`. Pour l'exposer via un reverse proxy externe :

```bash
kubectl patch svc longhorn-frontend -n longhorn-system -p '{"spec": {"type": "ClusterIP"}}'
kubectl patch svc longhorn-frontend -n longhorn-system -p '{"spec": {"type": "NodePort"}}'
```
Pour voir le port exposé :
```bash
kubectl get svc longhorn-frontend -n longhorn-system
```

---

## Réglages Longhorn clés
Ouvrez l’UI → Settings et ajustez selon votre topologie :
| Réglage                                          | 3 workers                                     | 2 workers | Pourquoi ?                                 |
| ------------------------------------------------ | --------------------------------------------- | --------- | ------------------------------------------ |
| Default Replica Count                            | `3`                                           | `2`       | Nombre de copies de chaque volume.         |
| Pod Deletion Policy When Node is Down            | `delete-deployment-pod`                       | idem      | Supprime automatiquement les pods bloqués. |
| Node Drain Policy                                | `block-for-eviction-if-contains-last-replica` | idem      | Garde au moins 1 réplica healthy.          |
| Allow Volume Creation with Degraded Availability | ✅                                             | ✅         | Autorise volume si réplicas manquants.     |
| Auto-Delete Workload Pod on Unexpected Detach    | ✅                                             | ✅         | Relance automatique du pod.                |
| Replica Auto Balance                             | `best-effort`                                 | idem      | Rééquilibre si nœuds dispo.                |
| Default Data Locality                            | `disabled`                                    | idem      | Pas de contrainte de localisation.         |

## Passage de 3→2 workers (Test de bascule)

Changez dans Settings :
```bash
Default Replica Count → 2
```
Appliquez aux futurs volumes :
```bash
export KUBE_EDITOR="nano"
kubectl edit cm longhorn-storageclass -n longhorn-system

volumeBindingMode: WaitForFirstConsumer
parameters:
  numberOfReplicas: "2"
```

## tester 

Eteindre le worker ou les volumes sont situer 
Puis attendre si les pods sont bien recrée sur le(s) autre(s) worker(s)

# Reverse Proxy Caddy depuis une DMZ

## 1. Prérequis sur la machine Caddy (par exemple : 192.168.20.100)

* Avoir installé Caddy
* Avoir accès au port HTTP (80)

```bash
sudo dnf install -y caddy  # Sur Rocky Linux
```

## 2. Configuration du fichier Caddyfile

```caddyfile
{
  email admin@votredomaine.fr
}

http://192.168.60.10:6000 {
  reverse_proxy http://<IP_K8S_MASTER>:<PORT_NODEPORT>
}
```

> Remplacez `<IP_K8S_MASTER>` et `<PORT_NODEPORT>` par l'adresse IP et le port affiché dans `kubectl get svc -n longhorn-system`

## 3. Activer le reverse proxy

```bash
sudo caddy fmt --overwrite /etc/caddy/Caddyfile
sudo caddy reload --config /etc/caddy/Caddyfile
```

## 4. Accéder à Longhorn

Depuis un navigateur :

```
http://192.168.60.10
```

---

## 📄 Notes Complémentaires

* Assurez-vous que le firewall sur la machine Caddy permet le port 80 :

```bash
sudo firewall-cmd --permanent --add-port=80/tcp
sudo firewall-cmd --reload
```

* Si vous utilisez un nom de domaine (ex : `longhorn.alpha-pro.fr`), utilisez ce nom dans le Caddyfile au lieu de l'IP.

## Test Final

* Interface Longhorn fonctionnelle sur `http://192.168.40.40`
* Volumes créables et attachables sur vos déploiements Kubernetes

---
pour moi 
export KUBE_EDITOR="nano"
kubectl edit cm longhorn-storageclass -n longhorn-system

volumeBindingMode: WaitForFirstConsumer
parameters:
  numberOfReplicas: "2"

kubectl -n default delete pod -l app=wordpress
kubectl -n default delete pod -l app=mysql

