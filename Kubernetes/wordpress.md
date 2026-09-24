---
title: Configuration d'un wordpress
description: 
published: 1
date: 2026-09-24T18:25:11.868Z
tags: 
editor: markdown
dateCreated: 2025-05-07T21:16:09.364Z
---

# Installation et configuration complètes de WordPress avec Traefik Ingress et Caddy

Ce guide reprend toutes les étapes que nous avons réalisées pour déployer WordPress sur un cluster Kubernetes Debian (1 master + 2 workers), avec :

* **Traefik** comme Ingress Controller
* **Caddy** en front-end reverse‐proxy TLS

> Vous trouverez ci‑dessous la procédure complète, pas à pas, en Markdown.

---

## 1. Prérequis

* **Cluster Kubernetes opérationnel** (1 master @192.168.40.10 et 2 workers @192.168.40.20/30)
* **Longhorn** installé (pour le stockage persisté)
* **kubectl**, **helm**, **curl**, **open-iscsi**, **nfs-common** installés sur tous les nœuds
* Un **DNS** ou `/etc/hosts` pointant :

  * `alpha-pro.fr` → `192.168.40.10`

---

## 2. Installer Traefik Ingress

Sur le master Kubernetes :

```bash
# 1. Ajouter le repo Helm Traefik
helm repo add traefik https://traefik.github.io/charts
helm repo update

# 2. Déployer Traefik
helm install traefik traefik/traefik \
  --namespace traefik --create-namespace \
  --set service.type=NodePort \
  --set service.ports.web.nodePort=31080 \
  --set service.ports.websecure.enabled=false \
  --set ingressClass.enabled=true \
  --set ingressClass.isDefaultClass=true

# 3. Vérifier
kubectl get pods,svc -n traefik
kubectl get ingressclass
```

---

## 3. Déployer MySQL + WordPress sur Kubernetes

Dans `/kubernetes/wordpress`, crée ces fichiers :

### secrets.yaml

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysql-pass
type: Opaque
data:
  password: UmVzcG9uczExKw==   # "Respons11+" à modifier en base 64
---
apiVersion: v1
kind: Secret
metadata:
  name: wp-db-creds
type: Opaque
data:
  username: d29yZHByZXNz      # "wordpress"
  password: UmVzcG9uczExKw==  # "Respons11+"
```

### pvc-mysql.yaml

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pv-claim
spec:
  accessModes: ["ReadWriteOnce"]
  storageClassName: longhorn
  resources:
    requests:
      storage: 2Gi
```

### pvc-wp.yaml

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: wp-pv-claim
spec:
  accessModes: ["ReadWriteOnce"]
  storageClassName: longhorn
  resources:
    requests:
      storage: 5Gi
```

### mysql-deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
spec:
  replicas: 1
  selector:
    matchLabels: {app: mysql}
  template:
    metadata:
      labels: {app: mysql}
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-pass
              key: password
        ports:
        - containerPort: 3306
        volumeMounts:
        - name: mysql-storage
          mountPath: /var/lib/mysql
      volumes:
      - name: mysql-storage
        persistentVolumeClaim:
          claimName: mysql-pv-claim
---
apiVersion: v1
kind: Service
metadata:
  name: mysql
  labels:
    app: wordpress
    tier: mysql
spec:
  type: ClusterIP
  selector:
    app: wordpress
    tier: mysql
  ports:
  - name: mysql
    port: 3306
    targetPort: 3306
```

### wordpress-deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress
  labels:
    app: wordpress
    tier: frontend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: wordpress
      tier: frontend
  template:
    metadata:
      labels:
        app: wordpress
        tier: frontend
    spec:
      containers:
      - name: wordpress
        image: wordpress:6.4-php8.1-apache
        env:
        - name: WORDPRESS_DB_HOST
          value: mysql:3306
        - name: WORDPRESS_DB_USER
          valueFrom:
            secretKeyRef:
              name: wp-db-creds
              key: username
        - name: WORDPRESS_DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: wp-db-creds
              key: password
        - name: WORDPRESS_DB_NAME
          value: wordpress
        - name: WP_HOME
          value: "https://alpha-pro.fr"
        - name: WP_SITEURL
          value: "https://alpha-pro.fr"
        ports:
        - containerPort: 80
        volumeMounts:
        - name: wp-storage
          mountPath: /var/www/html
      volumes:
      - name: wp-storage
        persistentVolumeClaim:
          claimName: wp-pv-claim
---
apiVersion: v1
kind: Service
metadata:
  name: wordpress
  labels:
    app: wordpress
    tier: frontend
spec:
  type: ClusterIP
  selector:
    app: wordpress
    tier: frontend
  ports:
  - name: http
    port: 80
    targetPort: 80
```


### ingress-wordpress.yaml

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-wordpress
  annotations:
    kubernetes.io/ingress.class: traefik
spec:
  ingressClassName: traefik
  rules:
  - host: alpha-pro.fr
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: wordpress
            port: { number: 80 }
```

### kustomization.yaml

```yaml
resources:
  - secrets.yaml
  - pvc-longhorn.yaml
  - pvc-mysql.yaml 
  - mysql-deployment.yaml
  - wordpress-deployment.yaml
  - ingress-wordpress.yaml
```

---

## 4. Appliquer tous les manifests

```bash
kubectl apply -f ./
```

Vérifie que tous les Pods sont `Running` et que PVC sont `Bound` :

```bash
kubectl get pods,svc,pvc,ingress
```

---

## 5. Installer et configurer Caddy en DMZ

Sur la VM Caddy (`192.168.60.10` sous Rocky) :

1. **Caddyfile** `/etc/caddy/Caddyfile` :

   ```caddyfile
   {
     email administrateur@alpha-pro.fr
   }
   
   # WordPress devant Traefik
   alpha-pro.fr {
     reverse_proxy 192.168.40.10:31080 
   }
   ```

2. **Firewall** (Rocky) :

   ```bash
   sudo firewall-cmd --add-service=http --add-service=https --permanent
   sudo firewall-cmd --reload
   ```

3. **Reload** Caddy :

   ```bash
   sudo caddy fmt --overwrite /etc/caddy/Caddyfile
   sudo caddy reload --config /etc/caddy/Caddyfile
   ```

---


## 6. Tests finaux

1. **En interne** (depuis Master) :

   ```bash
   curl -Ik http://192.168.40.10:31080/ -H "Host: alpha-pro.fr"
   ```

   → 200 OK / redirection WordPress.

2. **En DMZ/Caddy** :
   Navigue sur **[https://alpha-pro.fr](https://alpha-pro.fr)** dans ton navigateur.


---

## 7. Vérifier la persistance du stockage WordPress

### a) Créer un contenu de test

1. Accède à **[https://alpha-pro.fr/wp-admin](https://alpha-pro.fr/wp-admin)** dans ton navigateur.
2. Crée un nouvel article ou téléverse un média (image, PDF, etc.).
3. Vérifie qu’il apparaît sur le front-end de ton site.

### b) Redémarrer le pod WordPress

```bash
kubectl rollout restart deployment/wordpress
kubectl get pods -l app=wordpress
```

Une fois le pod `wordpress` à nouveau **Running**, reteste :

```bash
curl -I http://192.168.40.10:31080/ -H "Host: alpha-pro.fr"
```

Vérifie que ton article/média est toujours présent : la persistance fonctionne.

## 10. Test de tolérance aux pannes avec Longhorn

### a) Localiser la réplique du volume

1. Ouvre le Longhorn UI (`https://longhorn.alpha-pro.fr`).
2. Choisis ton volume `wp-pv-claim` et regarde sur quels nodes ses répliques sont en `Healthy`.

### b) Simuler l’arrêt d’un node

Sur le nœud (VM) hébergeant une réplique :

```bash
# Depuis l’hôte ou vSphere/VMware :
# Éteins ou suspends la VM, ex. worker-02
```

### c) Vérifier le failover

1. Dans Longhorn UI, observe que la réplique passe sur un autre node et reste `Healthy`.
2. Relance ton déploiement WordPress :

   ```bash
   kubectl rollout restart deployment/wordpress
   ```
3. Teste ton site :

   ```bash
   curl -I http://192.168.40.10:31080/ -H "Host: alpha-pro.fr"
   ```

   Tu dois toujours obtenir **200 OK** ou **302**, confirmant que le stockage est resté accessible malgré la panne d’un node.


 
**Félicitations !**

Tu as maintenant un WordPress en production :

* Stockage persistant Longhorn
* Ingress Traefik
* Reverse‑proxy TLS Caddy

