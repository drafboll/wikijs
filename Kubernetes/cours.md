---
title: TP ESGI K8S
description: 
published: 1
date: 2026-09-25T15:41:06.499Z
tags: 
editor: markdown
dateCreated: 2026-03-03T09:19:05.796Z
---

# TRAVAUX PRATIQUE K8S

## Support de Cours et Travaux Pratiques Kubernetes (K8S)

### Description
Ce document regroupe l'ensemble des exercices pratiques (Labs) pour la maîtrise des concepts fondamentaux de Kubernetes. 
> [k8s-v3.2.pdf](/k8s/k8s-v3.2.pdf)
{.is-success}
 
---

# TRAVAUX PRATIQUE K8S

## Lab1 Pods
### Premier Pod
Créer un pod nommé web basé sur l’image nginx:1.21-alpine

### Deuxième Pod
Créer un pod nommé debug basé sur une image alpine:3.17
Se connecter sur le pod debug et faire un curl sur l’IP du pod web

### Troisième Pod
Créer un pod nommé all basé sur 2 conteneurs (nginx et alpine)
Se connecter sur le pod all dans le conteneur alpine et faire un curl sur l’IP localhost

## Lab1 Solution
### Premier Pod
Créer un pod nommé web basé sur une image nginx
Terminal window
```bash
    vi web.yml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web
spec:
  containers:
  - name: www
    image: nginx:1.21-alpine
```

Terminal window
```bash
    kubectl apply -f web.yml
```

### Deuxième Pod
Créer un pod nommé debug basé sur une image alpine
Terminal window
```bash
    vi debug.yml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: debug
spec:
  containers:
  - name: debug
    image: alpine:3.17
    command: ["sleep","3600"]
```

Terminal window
```bash
    kubectl apply -f debug.yml
```

Se connecter sur le pod debug et faire un curl sur l’IP du pod web
Terminal window
```bash
    kubectl exec -it debug -- sh
    apk add curl
    curl <IP WEB>
```

### Troisième Pod
Créer un pod nommé all basé sur 2 conteneurs (nginx et alpine)
Terminal window
```bash
    vi all.yml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: all
spec:
  containers:
  - name: www
    image: nginx:1.21-alpine
  - name: debug
    image: alpine:3.17
    command: ["sleep","3600"]
```

Terminal window
```bash
    kubectl apply -f all.yml
```

Se connecter sur le pod all dans le conteneur alpine et faire un curl sur l’IP localhost
Terminal window
```bash
    kubectl exec -it all -c debug -- sh
    apk add curl
    curl 127.0.0.1
```

## Lab2 Scheduler
### Premier Pod
Créer un pod nommé web basé sur l’image nginx:1.21-alpine
Ajouter un label app=web
Ajouter un nodeSelector sur un label disktype=ssd
Appliquer le fichier de spécification et remarquer que le status du pod est Pending
### Deuxième Pod
Créer un pod nommé debug basé sur l’image alpine:3.17
Ajouter une Affinity de type pod sur un Label app=web
Appliquer le fichier de spécification et remarquer que le status du pod est Pending
### Worker
Créer sur un worker un Label disktype=ssd
Remarquer que le status des pods nginx et alpine est Running

## Lab2 Scheduler solution
### Premier Pod
Créer un pod nommé web basé sur une image nginx
Terminal window
```bash
vi web.yml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web
  labels:
    app: web
spec:
  containers:
  - name: www
    image: nginx:1.19-alpine
  nodeSelector:
    disktype: ssd
```

Appliquer les spécifications
Terminal window
```bash
kubectl apply -f web.yml
```

Visualiser le résultat
Terminal window
```bash
kubectl get pods -o wide --show-labels
```

### Deuxième PoD
Créer un pod nommé debug basé sur une image alpine
Terminal window
```bash
vi debug.yml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: debug
spec:
  containers:
  - name: debug
    image: alpine:3.12
    command: ["sleep","3600"]
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: app
            operator: In
            values:
            - web
        topologyKey: kubernetes.io/hostname
```

Appliquer les spécifications
Terminal window
```bash
kubectl apply -f debug.yml
```

Visualiser le résultat
Terminal window
```bash
kubectl get pods -o wide --show-labels
```

### Worker
Créer sur un worker un Label disktype=ssd
Terminal window
```bash
vi host13.yml
```

```yaml
apiVersion: v1
kind: Node
metadata:
  labels:
    disktype: ssd
  name: host13
```

Appliquer la spécification
Terminal window
```bash
kubectl apply -f host13.yml
```

Remarquer que le status des pods nginx et alpine est Running
Terminal window
```bash
kubectl get pods -o wide --show-labels
```


## Lab3 Service
### Service ClusterIP
Créer un service nommé s1 de type ClusterIP rassemblant des pods ayant comme label app1=web
Créer un pod web1 basé sur l’image nginx:1.21-alpine avec comme label app1=web
Créer un pod web2 basé sur l’image httpd:2.4-alpine avec comme label app1=web
Créer un pod debug basé sur l’image alpine
Se connecter sur le pod debug et exécuter plusieurs fois la commande “curl s1”

### Service NodePort
Créer un service nommé s2 de type NodePort rassemblant des pods ayant comme label app2=web
Créer un pod web3 basé sur l’image nginx:1.21-alpine avec comme label app2=web
Créer un pod web4 basé sur l’image httpd:2.4-alpine avec comme label app2=web
Depuis le poste de travail, exécuter plusieurs fois la commande “curl node1:30000”


## Lab3 Service Solution
### Service ClusterIP

Créer un service nommé s1 de type ClusterIP rassemblant des pods ayant comme label app1=web
Terminal window
```bash
vi s1.yml
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: s1
spec:
  selector:
    app1: web
  type: ClusterIP
  ports:
    - port: 80
      targetPort: 80
```

Créer un pod web1 basé sur l’image nginx avec comme label app1=web
Terminal window
```bash
vi web1.yml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web1
  labels:
    app1: web
spec:
  containers:
  - name: nginx
    image: nginx:1.21-alpine
```

Créer un pod web2 basé sur l’image httpd avec comme label app1=web
Terminal window
```bash
vi web2.yml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web2
  labels:
    app1: web
spec:
  containers:
  - name: nginx
    image: httpd:2.4-alpine
```

Créer un pod debug basé sur l’image alpine
Terminal window
```bash
vi debug.yml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: debug
spec:
  containers:
  - name: debug
    image: alpine:3.13
    command: ["sleep","3600"]
```

Se connecter sur le pod debug et exécuter plusieurs fois la commande “curl s1”
Appliquer les spécifications
Terminal window
```bash
kubectl apply -f .
```

```text
pod/debug created
service/s1 created
pod/web1 created
pod/web2 created
```

Visualiser les pods et le service
Terminal window
```bash
kubectl get svc,pods
```

```text
NAME                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
service/s1           ClusterIP   10.96.245.226   <none>        80/TCP    63s

NAME        READY   STATUS    RESTARTS   AGE
pod/debug   1/1     Running   0          63s
pod/web1    1/1     Running   0          63s
pod/web2    1/1     Running   0          63s
```

Se connecter sur le pod debug et exécuter plusieurs fois la commande “curl s1”
Terminal window
```bash
kubectl exec -it debug -- sh
```

Terminal window
```bash
apk add curl
while true
do
curl s1
done
```

### Service NodePort

Créer un service nommé s2 de type NodePort rassemblant des pods ayant comme label app2=web
Terminal window
```bash
vi s2.yml
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: s2
spec:
  selector:
    app2: web
  type: NodePort
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30000
```

Créer un pod web3 basé sur l’image nginx avec comme label app2=web
Terminal window
```bash
vi web3.yml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web3
  labels:
    app2: web
spec:
  containers:
  - name: nginx
    image: nginx:1.21-alpine
```

Créer un pod web4 basé sur l’image httpd avec comme label app2=web
Terminal window
```bash
vi web4.yml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web4
  labels:
    app2: web
spec:
  containers:
  - name: nginx
    image: httpd:2.4-alpine
```

Depuis le poste de travail, exécuter plusieurs fois la commande “curl node1:30000”
Appliquer les spécifications
Terminal window
```bash
kubectl apply -f .
```

```text
service/s2 created
pod/web3 created
pod/web4 created
```

Visualiser les pods et le service
Terminal window
```bash
kubectl get svc,pods
```

```text
NAME                 TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
service/s2           NodePort    10.96.34.252   <none>        80:30000/TCP   57s

NAME       READY   STATUS    RESTARTS   AGE
pod/web3   1/1     Running   0          57s
pod/web4   1/1     Running   0          57s
```

Depuis le poste de travail, exécuter plusieurs fois la commande “curl node1:30000”
Terminal window
```bash
while true
do
curl node1:30000
done
```



## Lab4 Deployment & DaemonSet
### Deployment
Créer un deploiement nommé web (avec 4 replicas) basé sur l’image nginx:1.17-alpine
Créer un service nommé web de type NodePort rassemblant les pods du deploiment web sur le nodePort 30001
### DaemonSet
Créer un DaemonSet nommé debug basé sur l’image alpine:3.12

Scale: Augmenter le nombre de replicas du deploiement web
par édition du fichier
par la commande kubectl scale

Rolling Update du deploiement web
```bash
kubectl set image deployment/frontend www=image:v2               # Rolling update du conteneur "www" du déploiement "frontend", par mise à jour de son image
kubectl rollout history deployment/frontend                      # Vérifie l'historique de déploiements incluant la révision
kubectl rollout undo deployment/frontend                         # Rollback du déploiement précédent
kubectl rollout undo deployment/frontend --to-revision=2         # Rollback à une version spécifique
kubectl rollout status -w deployment/frontend                    # Écoute (Watch) le status du rolling update du déploiement "frontend" jusqu'à ce qu'il se termine
kubectl rollout restart deployment/frontend                      # Rolling restart du déploie
```

## Lab4 Deployment & DaemonSet Solution
### Deployment
Créer un deploiement nommé web (avec 4 replicas) basé sur l’image httpd
Terminal window
```bash
vi deployment.yml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app3-web
spec:
  replicas: 4
  selector:
    matchLabels:
      app3: web
  template:
    metadata:
      labels:
        app3: web
    spec:
      containers:
      - name: nginx
        image: nginx:1.17-alpine
```

Créer un service nommé web de type NodePort rassemblant les pods du deploiment web sur le nodePort 30001
Terminal window
```bash
vi service.yml
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  selector:
    app3: web
  type: NodePort
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30001
```

Appliquer les spécifications
Terminal window
```bash
kubectl apply -f .
```

Visualiser le résultat
Terminal window
```bash
kubectl get deploy,rs,pods,svc -o wide
```

```text
NAME                  READY   UP-TO-DATE   AVAILABLE   AGE     CONTAINERS   IMAGES             SELECTOR
deployment.apps/web   4/4     4            4           5m20s   apache       httpd:2.4-alpine   app3=web

NAME                             DESIRED   CURRENT   READY   AGE     CONTAINERS   IMAGES             SELECTOR
replicaset.apps/web-66bcf7cf77   4         4         4       5m20s   apache       httpd:2.4-alpine   app3=web,pod-template-hash=66bcf7cf77

NAME                       READY   STATUS        RESTARTS   AGE     IP              NODE     NOMINATED NODE   READINESS GATES
pod/app3-ds-7ldcc          1/1     Terminating   0          3m25s   192.168.0.145   host13   <none>           <none>
pod/app3-ds-vkt88          1/1     Terminating   0          3m25s   192.168.0.72    host12   <none>           <none>
pod/web-66bcf7cf77-2q644   1/1     Running       0          5m20s   192.168.0.143   host13   <none>           <none>
pod/web-66bcf7cf77-bnchn   1/1     Running       0          5m20s   192.168.0.144   host13   <none>           <none>
pod/web-66bcf7cf77-kmmfk   1/1     Running       0          5m20s   192.168.0.71    host12   <none>           <none>
pod/web-66bcf7cf77-pwnnq   1/1     Running       0          5m20s   192.168.0.70    host12   <none>           <none>

NAME                 TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE     SELECTOR
service/web          NodePort    10.97.155.18   <none>        80:30001/TCP   5m20s   app3=web
```

### DaemonSet
Créer un DaemonSet nommé debug basé sur l’image alpine
Terminal window
```bash
vi daemonset.yml
```

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: app3-ds
spec:
  selector:
    matchLabels:
      app3: daemon
  template:
    metadata:
      labels:
        app3: daemon
    spec:
      containers:
      - name: daemon
        image: alpine:3.12
        command: ["sleep","3600"]
```

Appliquer la spécification
```bash
kubectl apply -f daemonset.yml
```

Visualiser le résultat
Terminal window
```bash
kubectl get ds,pods -o wide
```

```text
NAME                     DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE   CONTAINERS   IMAGES        SELECTOR
daemonset.apps/app3-ds   2         2         2       2            2           <none>          47s   daemon       alpine:3.12   app3=daemon

NAME                READY   STATUS    RESTARTS   AGE   IP              NODE     NOMINATED NODE   READINESS GATES
pod/app3-ds-l6gcv   1/1     Running   0          47s   192.168.0.146   host13   <none>           <none>
pod/app3-ds-tr6z2   1/1     Running   0          47s   192.168.0.73    host12   <none>      
```


## Lab5 Namespace
### Namespace
Créer un namespace nommé production
Créer un namespace nommé development

### Context
Créer un contexte prod qui fait référence au namespace production
Créer un contexte dev qui fait référence au namespace development

### Utilisation
Lancer un pod web dans le namespace development
Lancer le même pod web dans le namespace production


## Lab5 Namespace Solution
### Namespace
Créer un namespace nommé production
Terminal window
```bash
vi prod.yml
```

```yaml
apiVersion: v1
kind: Namespace
metadata:
    name: production
```

Créer un namespace nommé development
Terminal window
```bash
vi dev.yml
```

```yaml
apiVersion: v1
kind: Namespace
metadata:
    name: development
```

Appliquer
Terminal window
```bash
kubectl apply -f dev.yml
kubectl apply -f prod.yml
```

Afficher le résultat
Terminal window
```bash
kubectl get namespaces
```

```text
NAME              STATUS   AGE
default           Active   3d12h
development       Active   15h
kube-node-lease   Active   3d12h
kube-public       Active   3d12h
kube-system       Active   3d12h
production        Active   15h
```

### Context
Créer un contexte prod qui fait référence au namespace production
Terminal window
```bash
kubectl config set-context prod --cluster=cluster.local --user=kubernetes-admin --namespace=production
```

Créer un contexte dev qui fait référence au namespace development
Terminal window
```bash
kubectl config set-context dev --cluster=cluster.local --user=kubernetes-admin --namespace=development
```

Afficher le résultat
Terminal window
```bash
kubectl config get-contexts
```

```text
CURRENT   NAME                          CLUSTER      AUTHINFO           NAMESPACE
            dev                           kubernetes   kubernetes-admin   development
* kubernetes-admin@kubernetes   kubernetes   kubernetes-admin
            prod                          kubernetes   kubernetes-admin   production
```

### Utilisation
Lancer un pod web dans le contexte dev
Terminal window
```bash
vi web.yml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
    name: web
    labels:
    app: web
spec:
    containers:
    - name: www
    image: nginx:1.17-alpine
```

Terminal window
```bash
kubectl config use-context dev
kubectl apply -f web.yml
```

Lancer le même pod web dans le contexte prod
Terminal window
```bash
    kubectl config use-context prod
    kubectl apply -f web.yml
```

Afficher le résultat
Terminal window
```bash
kubectl get pods --all-namespaces
```

```text
NAMESPACE     NAME                                      READY   STATUS    RESTARTS   AGE
development   web                                       1/1     Running   0          8s
kube-system   calico-kube-controllers-bcc6f659f-r6rkz   1/1     Running   3          3d1h
kube-system   calico-node-992r2                         1/1     Running   1          3d1h
kube-system   calico-node-dxwqg                         1/1     Running   1          3d1h
kube-system   calico-node-n5d6v                         1/1     Running   5          3d1h
kube-system   coredns-74ff55c5b-dztqx                   1/1     Running   3          3d13h
kube-system   coredns-74ff55c5b-kptvf                   1/1     Running   4          3d13h
kube-system   etcd-host11                               1/1     Running   5          3d13h
kube-system   kube-apiserver-host11                     1/1     Running   4          3d13h
kube-system   kube-controller-manager-host11            1/1     Running   5          3d13h
kube-system   kube-proxy-6tzzg                          1/1     Running   3          3d13h
kube-system   kube-proxy-bcxjr                          1/1     Running   1          3d1h
kube-system   kube-proxy-tfml9                          1/1     Running   1          3d1h
kube-system   kube-scheduler-host11                     1/1     Running   5          3d13h
production    web                                       1/1     Running   0          14s
```

## Lab6 Volumes
### Producteur
Créer un pvc web et un pv web qui pointe sur un volume de type nfs pour créer un fichier index.html
Serveur NFS: 192.168.1.5 Path: /home/shares/userX
Créer un daemonSet nommé web basé sur l’image alpine
Monter le pvc web dans le conteneur sur le chemin /web
Exécuter la commande date et la commande hostname et rediriger la sortie vers le fichier /web/index.html
    command: ["/bin/sh","-c","while true ; do echo `date` `hostname` >> /web/index.html; sleep 60 ;done"]

### Consommateur
Créer un déploiement nommé web basé sur l’image nginx
Monter le pvc web dans le conteneur sur le chemin /usr/share/nginx/html de Nginx

### Service NodePort
Créer un service nommé web de type NodePort qui rassemble les pods du Deploy web

### Test
Depuis le poste de travail, exécuter la commande “curl node1:30000”


## Lab6 Volumes Solutions
### Producteur
Créer un pvc web et un pv web qui pointe sur un volume web de type nfs pour créer un fichier index.html
Création du pv nfs
Terminal window
```bash
vi nfs-pv.yml
```

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteMany
  nfs:
    path: /home/shares/user12
    server: 192.168.1.5
```

Création du pvc nfs
Terminal window
```bash
vi nfs-pvc.yml
```

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs-pvc
spec:
  accessModes:
  - ReadWriteMany
  resources:
      requests:
        storage: 1Gi
```

Créer un daemonSet nommé web basé sur l’image alpine
Terminal window
```bash
vi ds-nfs.yml
```

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: web-ds
spec:
  selector:
    matchLabels:
      app: web-ds
  template:
    metadata:
      labels:
        app: web-ds
    spec:
      containers:
      - name: daemon
        image: alpine:3.12
        command: ["/bin/sh"]
        args: ["-c","while true; do echo `hostname` `date` >> /mnt/index.html ; sleep 60 ;done"]
        volumeMounts:
        - name: data
          mountPath: /mnt
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: nfs-pvc
```

### Consommateur
Créer un déploiement nommé web basé sur l’image httpd
Terminal window
```bash
vi deploy-nfs.yml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-deploy
spec:
  replicas: 4
  selector:
    matchLabels:
      app: web-deploy
  template:
    metadata:
      labels:
        app: web-deploy
    spec:
      containers:
      - name: nginx
        image: nginx:1.17-alpine
        ports:
        - containerPort: 80
        volumeMounts:
        - name: data
          mountPath: /usr/share/nginx/html
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: nfs-pvc
          readOnly: true
```

### Service NodePort
Créer un service nommé web de type NodePort qui rassemble les pods du DaemonSet web
Terminal window
```bash
vi service-nfs.yml
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-svc
spec:
  selector:
    app: web-deploy
  type: NodePort
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30001
```

Appliquer les spécifications
Terminal window
```bash
kubectl apply -f .
```

### Test
Depuis le poste de travail, exécuter la commande “curl node1:30001”
Terminal window
```bash
curl host111:30001
```

## Lab7 ConfigMap
### ConfigMap de type Volume
Copier le fichier /etc/nginx/nginx.conf à partir d’un pod web existant
Modifier le fichier et créer un ConfigMap nommé nginx-conf
Créer un pod web avec le ConfigMap nginx-conf
Vérifier le fichier de conf /etc/nginx/nginx.conf du pod web
### ConfigMap de type variable d’environnement
Créer un ConfigMap nommé mysql-pass qui contient une clé password et une valeur root
Créer un pod mysql et initialiser le MYSQL_ROOT_PASSWORD de mysql:8
Vérifier que le pod mysql est Running


## Lab7 ConfigMap Solution
### ConfigMap de type Volume
Copier le fichier /etc/nginx/nginx.conf à partir d’un pod web existant
Terminal window
```bash
kubectl cp web:/etc/nginx/nginx.conf nginx.conf
```

Modifier le fichier et créer un ConfigMap nommé nginx-conf
Terminal window
```bash
kubectl create configmap nginx-conf --from-file=nginx.conf
```

Créer un pod web avec le ConfigMap nginx-conf
Terminal window
```bash
vi web.yml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web
  labels:
    app: web
spec:
  containers:
  - name: www
    image: nginx:1.21-alpine
    volumeMounts:
    - name: config
      mountPath: "/etc/nginx/nginx.conf"
      subPath: "nginx.conf"
  volumes:
  - name: config
    configMap:
      name: nginx-conf
```

Vérifier le fichier de conf /etc/nginx/nginx.conf du pod web
Terminal window
```bash
kubectl exec -it web -- sh
cat /etc/nginx/nginx.conf
```

### ConfigMap de type variable d’environnement
Créer un ConfigMap nommé mysql-pass qui contient une clé password et une valeur root
Terminal window
```bash
vi cm-mysql.yml
```

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mysql-pass
data:
  password: root
```

Terminal window
```bash
kubectl apply -f cm-mysql.yml
```

Créer un pod mysql et initialiser le ROOT_PASSWORD de mysql
Terminal window
```bash
vi mysql.yml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mysql
spec:
  containers:
  - image: mysql:8.0
    name: mysql
    env:
    - name: MYSQL_ROOT_PASSWORD
      valueFrom:
        configMapKeyRef:
          name: mysql-pass
          key: password
```

Terminal window
```bash
kubectl apply -f mysql.yml
```

Vérifier que le pod mysql est Running
Terminal window
```bash
kubectl get pods
```

## Lab8 Secrets
### Secrets de type Volume
Copier le fichier /etc/nginx/nginx.conf à partir d’un pod web existant
Modifier le fichier et créer un Secret nommé nginx-conf
Créer un pod web avec le Secret nginx-conf
Vérifier le fichier de conf /etc/nginx/nginx.conf du pod web
### Secret de type variable d’environnement
Créer un Secret nommé mysql-pass qui contient la clé password et la valeur root
Créer un pod mysql et initialiser le ROOT_PASSWORD de mysql
Vérifier que le pod mysql est Running


## Lab8 Secret Solution
### Secret de type Volume
Copier le fichier /etc/nginx/nginx.conf à partir d’un pod web existant
Terminal window
```bash
kubectl cp web:/etc/nginx/nginx.conf nginx.conf
```

Modifier le fichier et créer un Secret nommé nginx-conf
Terminal window
```bash
kubectl create secret generic nginx-conf --from-file=nginx.conf
```

Créer un pod web avec le Secret nginx-conf
Terminal window
```bash
vi web.yml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web
  labels:
    app: web
spec:
  containers:
  - name: www
    image: nginx:1.17-alpine
    volumeMounts:
    - name: config
      mountPath: "/etc/nginx/nginx.conf"
      subPath: "nginx.conf"
  volumes:
  - name: config
    secret:
      secretName: nginx-conf
```

Vérifier le fichier de conf /etc/nginx/nginx.conf du pod web
Terminal window
```bash
kubectl exec -it web -- sh
cat /etc/nginx/nginx.conf
```

### Secret de type variable d’environnement
Créer un Secret nommé mysql-pass qui contient une clé password et une valeur root
Terminal window
```bash
kubectl create secret generic mysql-pass --from-literal=password=root
```

Visualiser le Secret
Terminal window
```bash
kubectl get sc/mysql-pass -o yaml
```

Créer un pod mysql et initialiser le ROOT_PASSWORD de mysql
Terminal window
```bash
vi mysql.yml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mysql
spec:
  containers:
  - image: mysql:5.6
    name: mysql
    env:
    - name: MYSQL_ROOT_PASSWORD
      valueFrom:
        secretKeyRef:
          name: mysql-pass
          key: password
```

Terminal window
```bash
kubectl apply -f mysql.yml
```

Vérifier que le pod mysql est Running
Terminal window
```bash
kubectl get pods
```


## Lab9 Job & CronJob
### Job
Créer un pvc et un pv de type nfs
Créer un job nommé init permettant de créer un répertoire de sauvegarde sur le pv créé précédemment dont le nom est passé par Variable d’Environnement
### CronJob
Créer un CronJob nommé backup qui permet de sauvegarder les logs (/var/log) du Node Master du cluster
La sauvegarde doit être stockée sur le répertoire créé précédemment avec comme extension la date de sauvegarde


## Lab9 Job & CronJob Solution
### Job
Créer un job nommé init permettant de créer un répertoire de sauvegarde dont le nom est passé par Variable d’Environnement
Créer un ConfigMap pour la variable d’environnement
Terminal window
```bash
vi cm-dir.yml
```

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: backup-dir
data:
  dir: sauvegarde
```

Terminal window
```bash
kubectl apply -f cm-dir.yml
```

Créer un Job pour Créer le répertoire de sauvegarde
Terminal window
```bash
vi job-init.yml
```

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: init
spec:
  template:
    spec:
      restartPolicy: OnFailure
      containers:
      - name: init
        image: alpine:3.12
        command: ["/bin/sh"]
        args: ["-c","mkdir /mnt/$BACKUP_DIR"]
        env:
        - name: BACKUP_DIR
          valueFrom:
            configMapKeyRef:
              name: backup-dir
              key: dir
        volumeMounts:
        - name: data
          mountPath: /mnt
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: nfs-pvc
```

Vérifier que le répertoire a été créé
Terminal window
```bash
ls -l /home/shares/userX
```

### CronJob
Créer un CronJob nommé backup qui permet de sauvegarder les logs (/var/log) du Node du cluster
Terminal window
```bash
vi cronjob.yml
```

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: cron
spec:
  schedule: "*/1 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup
            image: alpine:3.12
            command: ["/bin/sh"]
            args: ["-c","tar -cvf /mnt/$BACKUP_DIR/backup.tar.`date +%h%d-%H%M` /logs_master"]
            env:
            - name: BACKUP_DIR
              valueFrom:
                configMapKeyRef:
                  name: backup-dir
                  key: dir
            volumeMounts:
            - name: logs
              mountPath: /logs_master
            - name: data
              mountPath: /mnt
          restartPolicy: OnFailure
          affinity:
            nodeAffinity:
              requiredDuringSchedulingIgnoredDuringExecution:
                nodeSelectorTerms:
                - matchExpressions:
                  - key: node-role.kubernetes.io/control-plane
                    operator: Exists


          volumes:
          - name: data
            persistentVolumeClaim:
              claimName: nfs-pvc
          - name: logs
            hostPath:
              path: /var/log
              type: Directory
```

Terminal window
```bash
kubectl apply -f cronjob.yml
```

Vérifier que la sauvegarde s’est bien effectuée
Terminal window
```bash
ls -l /home/shares/userX/sauvegarde
```


## DASHBOARD WITH HELM
Installer la commande helm sur le poste de travail
```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```

Installer le dashboard avec HELM
Aller sur le site [https://artifacthub.io/packages/helm/k8s-dashboard/kubernetes-dashboard](https://artifacthub.io/packages/helm/k8s-dashboard/kubernetes-dashboard)
Exécuter
Terminal window
```bash
helm repo add kubernetes-dashboard https://kubernetes.github.io/dashboard/
helm repo update
helm install dashboard kubernetes-dashboard/kubernetes-dashboard
```

```text
NAME: dashboard
LAST DEPLOYED: Wed Sep 11 10:57:55 2024
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
*************************************************************************************************
*** PLEASE BE PATIENT: Kubernetes Dashboard may need a few minutes to get up and become ready ***
*************************************************************************************************

Congratulations! You have just installed Kubernetes Dashboard in your cluster.

To access Dashboard run:
    kubectl port-forward svc/dashboard-kong-proxy 800X:443

NOTE: In case port-forward command does not work, make sure that kong service name is correct.
      Check the services in Kubernetes Dashboard namespace using:
        kubectl -n default get svc

Dashboard will be available at:
  https://localhost:800X
```

Créer le User
Terminal window
```bash
vi user.yml
```

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
```

Créer le Role Binding
Terminal window
```bash
vi role-binding.yml
```

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: default
```

Créer le token Binding
Terminal window
```bash
kubectl -n default create token admin-user
```

## Création d'un Chart HELM
Générer un Chart
Utilisez cette commande pour créer un nouveau chart dans un nouveau répertoire
Terminal window
```bash
helm create myhelm
```

Helm créera un nouveau répertoire dans votre projet appelé myhelm avec la structure ci-dessous.
```text
myhelm/
├── charts
├── Chart.yaml          // metadatas du Chart.
├── templates           // les manifestes
│   ├── deployment.yaml
│   ├── _helpers.tpl
│   ├── hpa.yaml
│   ├── ingress.yaml
│   ├── NOTES.txt       // Notes affichées à l'installation du Chart
│   ├── serviceaccount.yaml
│   ├── service.yaml
│   └── tests
│       └── test-connection.yaml
└── values.yaml         // valeurs des variables des templates
```

Templates
On retrouve dans le répertoire templates les définitions YAML pour vos services, déploiements et autres objets Kubernetes.
Si vous avez déjà des définitions pour votre application, il vous suffit de remplacer les fichiers YAML générés par les vôtres.
::: note ::: title Note :::
Helm exécute chaque fichier de ce répertoire via le moteur Go template. Helm étend le langage de modèle, en ajoutant un certain nombre de fonctions utilitaires pour écrire des charts. :::
Terminal window
```bash
cat myhelm/templates/service.yaml
```

```yaml
apiVersion: v1
kind: Service
metadata:
name: {{ template "fullname" . }}
labels:
    chart: "{{ .Chart.Name }}-{{ .Chart.Version | replace "+" "_" }}"
spec:
type: {{ .Values.service.type }}
ports:
- port: {{ .Values.service.externalPort }}
    targetPort: {{ .Values.service.internalPort }}
    protocol: TCP
    name: {{ .Values.service.name }}
selector:
    app: {{ template "fullname" . }}
```

Il s’agit d’une définition de service de base utilisant des modèles. Lors du déploiement du graphique, Helm générera une définition qui ressemblera beaucoup plus à un service valide. Nous pouvons effectuer une simulation d’une installation helm et activer le débogage pour inspecter les définitions générées :
Terminal window
```bash
helm install myapp --dry-run --debug ./myhelm
```

```yaml
apiVersion: v1
kind: Service
metadata:
name: myhelm
labels:
    chart: "myhelm-1.0"
spec:
type: ClusterIP
ports:
- port: 80
    targetPort: 80
    protocol: TCP
    name: nginx
selector:
    app: myhelm
.../...
```

Values
Le fichier Values est un élément clé des charts Helm. Il fournit les valeurs par défaut pour toutes les variables utilisées dans les templates.
Terminal window
```bash
cat myhelm/values.yaml
```

```yaml
# Default values for myhelm.

replicaCount: 1

image:
repository: nginx
pullPolicy: IfNotPresent
# Overrides the image tag whose default is the chart appVersion.
tag: ""


service:
type: ClusterIP
port: 80

resources: {}
# limits:
#   cpu: 100m
#   memory: 128Mi
# requests:
#   cpu: 100m
#   memory: 128Mi

autoscaling:
enabled: false
minReplicas: 1
maxReplicas: 100

nodeSelector: {}

tolerations: []

affinity: {}
```

::: note ::: title Note :::
On peut aussi surcharger ces variables à l’installation du chart par l’option —set :::
Terminal window
```bash
helm install myapp --dry-run  ./myhelm --set service.internalPort=8080
```

```text
NAME: myapp
LAST DEPLOYED: Wed Apr  6 06:27:29 2022
NAMESPACE: development
STATUS: pending-install
REVISION: 1
HOOKS:
---
# Source: myhelm/templates/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-myhelm
spec:
  type: ClusterIP
  ports:
    - port: 80
      targetPort: http
      protocol: TCP
      name: http
  selector:
    app.kubernetes.io/name: myhelm
---
# Source: myhelm/templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-myhelm
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: myhelm

 .../...

NOTES:
1. Get the application URL by running these commands:
export POD_NAME=$(kubectl get pods --namespace development -l "app.kubernetes.io/name=myhelm,app.kubernetes.io/instance=myapp" -o jsonpath="{.items[0].metadata.name}")
export CONTAINER_PORT=$(kubectl get pod --namespace development $POD_NAME -o jsonpath="{.spec.containers[0].ports[0].containerPort}")
echo "Visit http://127.0.0.1:8080 to use your application"
kubectl --namespace development port-forward $POD_NAME 8080:$CONTAINER_PORT
```

Documentation
Un autre fichier utile dans le répertoire templates/ est le fichier NOTES.txt. Il contient une documenation qui sera affichée à la fin de l’installation du chart
::: note ::: title Note :::
Il peut aussi afficher les valeurs des variables des templates. :::
Terminal window
```bash
cat myhelm/templates/NOTES.txt
```

```text
1. Get the application URL by running these commands:
{{- if .Values.ingress.enabled }}
    {{- range $host := .Values.ingress.hosts }}
    {{- range .paths }}
        http{{ if $.Values.ingress.tls }}s{{ end }}://{{ $host.host }}{{ .path }}
    {{- end }}
{{- end }}
{{- else if contains "NodePort" .Values.service.type }}
export NODE_PORT=$(kubectl get --namespace {{ .Release.Namespace }} -o jsonpath="{.spec.ports[0].nodePort}" services {{ include "myhelm.fullname" . }})
export NODE_IP=$(kubectl get nodes --namespace {{ .Release.Namespace }} -o jsonpath="{.items[0].status.addresses[0].address}")
echo http://$NODE_IP:$NODE_PORT
export POD_NAME=$(kubectl get pods --namespace {{ .Release.Namespace }} -l "app.kubernetes.io/name={{ include "myhelm.name" . }},app.kubernetes.io/instance={{ .Release.Name }}" -o jsonpath="{.items[0].metadata.name}")
export CONTAINER_PORT=$(kubectl get pod --namespace {{ .Release.Namespace }} $POD_NAME -o jsonpath="{.spec.containers[0].ports[0].containerPort}")
echo "Visit http://127.0.0.1:8080 to use your application"
kubectl --namespace {{ .Release.Namespace }} port-forward $POD_NAME 8080:$CONTAINER_PORT
{{- end }}
```



## Créer un package HELM
Créer le package
Helm permet de créer un package pour partager le Chart. La pckage créé est au format tar
Terminal window
```bash
helm package myhelm
```

```text
Successfully packaged chart and saved it to: /home/user9/myhelm-0.1.0.tgz
```

Installer un Chart à partir d’un package
Terminal window
```bash
helm install myapp myhelm-0.1.0.tgz
```

```text
NAME: myapp
NAMESPACE: development
STATUS: deployed
REVISION: 1
NOTES:
1. Get the application URL by running these commands:
export NODE_PORT=$(kubectl get --namespace development -o jsonpath="{.spec.ports[0].nodePort}" services myapp-myhelm)
export NODE_IP=$(kubectl get nodes --namespace development -o jsonpath="{.items[0].status.addresses[0].address}")
echo http://$NODE_IP:$NODE_PORT
```

Serveur de Chart
Afin de faciliter le partage de packages, Helm dispose d’un support intégré pour l’installation de packages à partir d’un serveur HTTP. Helm lit un index de référentiel hébergé sur le serveur qui décrit les packages de cartes disponibles et leur emplacement.


## Deployer un Chart HELM
Installer le chart
Le chart généré par défaut exécute un serveur NGINX exposé via un service de type ClusterIP.
Pour l’exposer avec un service NodePort, exécuter la commande suivante :
Terminal window
```bash
helm install mynginx ./myhelm --set service.type=NodePort
```

```text
NAME: mynginx
NAMESPACE: development
STATUS: deployed
REVISION: 1
NOTES:
1. Get the application URL by running these commands:
export NODE_PORT=$(kubectl get --namespace development -o jsonpath="{.spec.ports[0].nodePort}" services mynginx-myhelm)
export NODE_IP=$(kubectl get nodes --namespace development -o jsonpath="{.items[0].status.addresses[0].address}")
echo http://$NODE_IP:$NODE_PORT
```

Lister les Charts installés
Terminal window
```bash
helm list
```

```text
NAME    NAMESPACE   REVISION    UPDATED                                     STATUS      CHART           APP VERSION
mynginx development 1           2021-03-09 09:53:10.185241039 +0200 CEST    deployed    myhelm-0.1.0    1.16.0
```

Lister les objets kubernetes créés
Terminal window
```bash
kubectl get deploy,svc
```

```text
NAME                             READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/mynginx-myhelm   1/1     1            1           8m

NAME                     TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
service/mynginx-myhelm   NodePort   10.107.172.175   <none>        80:32298/TCP   8m
```

Modifier le chart
Pour cela, il faut modifier certaines valeurs du ficher values.yaml
Terminal window
```bash
vi myhelm/values.yaml
```

```yaml
replicaCount: 2
image:
  repository: httpd
  pullPolicy: IfNotPresent
  tag: "2.4-alpine"

service:
  type: NodePort
  port: 80
```

Terminal window
```bash
helm install myweb ./myhelm
```

```text
NAME: myweb
NAMESPACE: development
STATUS: deployed
REVISION: 1
NOTES:
1. Get the application URL by running these commands:
export NODE_PORT=$(kubectl get --namespace development -o jsonpath="{.spec.ports[0].nodePort}" services myweb-myhelm)
export NODE_IP=$(kubectl get nodes --namespace development -o jsonpath="{.items[0].status.addresses[0].address}")
echo http://$NODE_IP:$NODE_PORT
```



## Installation des Metrics
D’abord installer les metrics voir le site ### Télécharger le fichier yml des Metrics
```bash
wget https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```
Configuration de deploy
ajouter flag
`--kubelet-insecure-tls`

dans le components.yaml

Appliquer le fichier de Metrics
```bash
kubectl apply -f components.yaml
```


## AutoScaling Apache
```bash
kubectl apply -f https://k8s.io/examples/application/php-apache.yaml
```
Lancer l’autoscaler
```bash
kubectl autoscale deployment php-apache --cpu-percent=50 --min=1 --max=10
```
Vérifier l’autoscaler
```bash
kubectl get hpa
```
Simuler une charge
```bash
kubectl run -i --tty load-generator --rm --image=busybox --restart=Never -- /bin/sh -c "while sleep 0.01; do wget -q -O- http://php-apache; done"
```
Vérifier à nouveau l’autoscaler
```bash
kubectl get hpa -w
```
Vérifier le nombre d’instances
```bash
kubectl get deployment php-apache
```


## Ingress Controller


### Hébergement virtuel basé sur le nom
Les hôtes virtuels basés sur des noms prennent en charge le routage du trafic HTTP vers plusieurs noms d’hôte basés sur la même adresse IP.
Terminal window
```text
dep1.formation.local  -->|                |->  service1:80
                         | 192.168.1.111  |
dep2.formation.local  -->|                |->  service2:80
```

### Hébergement virtuel basé sur le chemin
Une configuration de type fanout achemine le trafic d’une adresse IP unique vers plusieurs services, en se basant sur l’URI HTTP demandée.
Terminal window
```text
host111.formation.local  -> 192.168.1.111 -> / dep1    service1:80
                                             / dep2    service2:80
```

### Installation du contrôleur
Documentation du contrôleur Ingress NGINX
Installer le contrôleur avec HELM
Terminal window
```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install my-release ingress-nginx/ingress-nginx
```

Vérifier le contrôler Ingress
Terminal window
```bash
POD_NAME=$(kubectl get pods -l app.kubernetes.io/name=ingress-nginx -o jsonpath='{.items[0].metadata.name}')

kubectl exec -it $POD_NAME -- /nginx-ingress-controller --version
```

### Exemple basé sur le domaine
Terminal window
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-domain
spec:
  ingressClassName: nginx
  rules:
  - host: dep1.formation.local
    http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: service1
            port:
              number: 80
  - host: dep2.formation.local
    http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: service2
            port:
              number: 80
```

### Exemple basé sur le chemin
Terminal window
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-path
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: host111.formation.local
    http:
      paths:
      - path: /dep1
        pathType: Prefix
        backend:
          service:
            name: service1
            port:
              number: 80
      - path: /dep2
        pathType: Prefix
        backend:
          service:
            name: service2
            port:
              number: 80
```

Terminal window
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web1-deployment
  labels:
    app: web1
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web1
  template:
    metadata:
      labels:
        app: web1
    spec:
      containers:
      - name: httpd
        image: httpd:2.4-alpine
        ports:
        - containerPort: 80
        resources:
          limits:
            cpu: 500m
          requests:
            cpu: 200m

---

apiVersion: v1
kind: Service
metadata:
  name: service1
spec:
  selector:
    app: web1
  type: ClusterIP
  ports:
    - port: 80
      targetPort: 80
```

Terminal window
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web2-deployment
  labels:
    app: web2
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web2
  template:
    metadata:
      labels:
        app: web2
    spec:
      containers:
      - name: nginx
        image: nginx:1.17
        ports:
        - containerPort: 80
        resources:
          limits:
            cpu: 500m
          requests:
            cpu: 200m

---

apiVersion: v1
kind: Service
metadata:
  name: service2
spec:
  selector:
    app: web2
  type: ClusterIP
  ports:
    - port: 80
      targetPort: 80
```


## Statefulset
### Introduction
Un StatefulSet gère des Pods qui sont basés sur une même spécification de conteneur. Il maintient une identité pour chacun de ces Pods.
::: note ::: title Note :::
Ces Pods sont créés à partir de la même spec, mais ne sont pas interchangeables : chacun a un identifiant persistant qu’il garde à travers tous ses re-scheduling. :::

### Use Case
Les StatefulSets sont utiles pour des applications qui nécessitent :
Des identifiants réseau stables et uniques.
Un stockage persistant stable.
Un déploiement et une mise à l’échelle ordonnés et contrôlés.
Des mises à jour continues (rolling update) ordonnées et automatisées.
::: note ::: title Note :::
Il faut fournir du stockage persistent à chaque instantce du StatefulSet. :::

### Spécifications du Statefulset
Terminal window
```text
FIELDS:
spec <Object>
    replicas  <integer>
    selector  <Object>
        matchExpressions       <[]Object>
        matchLabels    <map[string]string>
    template  <Object>
        spec   <Object>
            containers  <[]Object>
    volumeClaimTemplates      <[]Object>
        spec   <Object>
```



### StatefulSet Exemple
Terminal window
```bash
ls
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  type: NodePort
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30000
  selector:
    app: nginx
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  selector:
    matchLabels:
      app: nginx
  serviceName: "nginx"
  replicas: 2
  template:
    metadata:
      labels:
        app: nginx
    spec:
      terminationGracePeriodSeconds: 10
      containers:
      - name: nginx
        image: nginx:1.17-alpine
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 80
          name: web
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 1Gi
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: www1
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  nfs:
    server: 192.168.1.5
    path: /home/shares/user10/vol1
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: www2
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  nfs:
    server: 192.168.1.5
    path: /home/shares/user10/vol2
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: www3
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  nfs:
    server: 192.168.1.5
    path: /home/shares/user10/vol3
```