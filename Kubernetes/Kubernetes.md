---
title: Installation d'un Cluster Kubernetes
description: 
published: 1
date: 2026-09-24T18:25:24.302Z
tags: cluster, debian, k8s, kubernetes, rocky
editor: markdown
dateCreated: 2025-03-15T12:47:04.468Z
---

# **Installation d'une Infrastructure Kubernetes**

# **Pour debian 12**
 
## **Configuration :**
- **CRI** : Cri-O
- **CNI** : Cilium
- **Nombre de nodes** : 1 master + 2 workers (bien nommer ses machines !)
- **OS** : debian
- **Superviseur** : Vmware Workstation
- **Ram** : au moins 2Go / debian 
- **Disque** : 60Go

---

## **I. Initialisation des nodes (À faire sur toutes les nodes)**

Définir les versions :
```sh
KUBERNETES_VERSION=v1.32
CRIO_VERSION=v1.32
```

Mettre à jour le système et installer les paquets nécessaires :
```sh
sudo apt-get update
sudo apt-get install -y software-properties-common curl
```

### **1. Ajout du dépôt Kubernetes**
Permet d’installer les outils essentiels (kubeadm, kubelet, kubectl) depuis une source officielle.
```sh
curl -fsSL https://pkgs.k8s.io/core:/stable:/$KUBERNETES_VERSION/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/$KUBERNETES_VERSION/deb/ /" |  sudo tee /etc/apt/sources.list.d/kubernetes.list
```

### **2. Ajout du dépôt CRI-O**
Permet d’installer CRI-O, un runtime de conteneurs optimisé pour Kubernetes.

 CRI-O (Container Runtime Interface for OpenShift) est un moteur d’exécution de conteneurs.

Il remplace Docker pour exécuter des conteneurs dans Kubernetes.
Il est plus léger, plus rapide et spécialement optimisé pour Kubernetes.
Kubernetes n’a pas son propre moteur de conteneur, il a besoin d’un runtime comme CRI-O, containerd, ou (anciennement) Docker.
```sh
curl -fsSL https://download.opensuse.org/repositories/isv:/cri-o:/stable:/$CRIO_VERSION/deb/Release.key |  sudo gpg --dearmor -o /etc/apt/keyrings/cri-o-apt-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/cri-o-apt-keyring.gpg] https://download.opensuse.org/repositories/isv:/cri-o:/stable:/$CRIO_VERSION/deb/ /" |  sudo tee /etc/apt/sources.list.d/cri-o.list
```

Mettre à jour les paquets :
```sh
sudo apt-get update
```

### 3. Installer **Cri-O, Kubelet, Kubeadm et Kubectl** :

kubeadm sert à installer et configurer Kubernetes.
kubelet tourne en arrière-plan sur chaque machine pour exécuter les conteneurs.
kubectl permet d’administrer Kubernetes depuis le terminal
```sh
sudo apt-get install -y cri-o kubelet kubeadm kubectl
```

Démarrer le service **Cri-O** :
```sh
sudo systemctl start crio.service
```

Désactiver **le swap** :
```sh
sudo sed -i '/# swap/{n;s/^/#/;n;s/^/#/}' /etc/fstab
sudo swapoff -a
```
Mais aussi dans votre fichier fstable
```sh
sudo nano /etc/fstab
```
Et commenter cette ligne 
```sh
#/dev/mapper/SVR--K8S--WORKER--03--vg-swap_1 none            swap    sw              0       0
```

Activer les modules nécessaires :
```sh
sudo modprobe br_netfilter
sudo sysctl -w net.ipv4.ip_forward=1
sudo sysctl --system
```

### **4. Activer le routage IPv4 de manière persistante**

Kubernetes a besoin que le routage IPv4 (`ip_forward`) soit activé pour assurer la communication entre les pods et les nœuds. Pour le rendre persistant après redémarrage, exécutez :

```sh
echo "net.ipv4.ip_forward=1" | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
sudo sysctl --system
```
<br><br> 
## **II. Initialisation du cluster (À faire sur le master)**

```sh
sudo kubeadm init
```
![capture_d'écran_2025-03-15_141614.png](/k8s/capture_d'écran_2025-03-15_141614.png)



Ces commandes permettent de configurer l'accès à Kubernetes pour ton utilisateur actuel :

```sh
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Une fois cette étape terminée, vous devez copier la **commande `kubeadm join`** générée sur le master et l'exécuter sur **chaque Worker** pour les intégrer au cluster. 

Si vous avez perdu la commande, vous pouvez la régénérer sur le Master avec :
```sh
kubeadm token create --print-join-command
```
Puis, exécutez cette commande sur chaque Worker pour qu'il rejoigne le cluster.
![capture_d'écran_2025-03-15_144217.png](/k8s/capture_d'écran_2025-03-15_144217.png)
<br>
### **1. Installation du ****CNI (Cilium****) sur le master**
<br>

### **Qu’est-ce que Cilium ?**
Cilium est une solution **CNI (Container Network Interface)** basée sur **eBPF** qui gère la connectivité réseau des pods Kubernetes avec des fonctionnalités avancées :
- **Meilleures performances** grâce à eBPF.
- **Sécurité avancée** avec des règles de filtrage réseau dynamiques.
- **Optimisation du load balancing** intégré au noyau Linux.
- **Alternative à kube-proxy**, réduisant la latence réseau.

```sh
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
CLI_ARCH=amd64
if [ "$(uname -m)" = "aarch64" ]; then CLI_ARCH=arm64; fi
curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
sha256sum --check cilium-linux-${CLI_ARCH}.tar.gz.sha256sum
sudo tar xzvfC cilium-linux-${CLI_ARCH}.tar.gz /usr/local/bin
rm cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
```

Installation de Cilium avec une version spécifique :

```sh
cilium install --version 1.17.1
```

Vérification du statut de Cilium :

```sh
cilium status --wait
```
![status_cilium.png](/k8s/status_cilium.png)
<br><br>
## **III. Vérification du bon fonctionnement du cluster**

Une fois le cluster opérationnel, vous pouvez exécuter les commandes suivantes pour vérifier que tout fonctionne correctement :

### Vérifier l'état des nœuds du cluster
```sh
kubectl get nodes
```
👉 Cette commande affiche tous les nœuds et leur statut (`Ready` signifie qu'ils sont opérationnels).

### Vérifier les pods en cours d'exécution dans `kube-system`
```sh
kubectl get pods -n kube-system
```
👉 Permet de voir si tous les composants internes de Kubernetes (comme Cilium, CoreDNS, etc.) fonctionnent bien.

### Vérifier les logs d'un pod spécifique
```sh
kubectl logs -n kube-system <nom_du_pod>
```
👉 Utile si un pod est en `CrashLoopBackOff` ou `Pending` pour comprendre le problème.

### Déployer un pod de test (Nginx)
```sh
kubectl run nginx-test --image=nginx --port=80
```
👉 Cela lance un simple pod avec Nginx pour tester si Kubernetes exécute bien les conteneurs.

Vérifiez ensuite si le pod est bien en `Running` :
```sh
kubectl get pods
```

Si tout fonctionne bien, votre cluster est totalement opérationnel ! 🚀

# **Pour Rocky Linux 9**

## Topologie du cluster

| Rôle        | Hostname         | IP             |
|-------------|------------------|----------------|
| Master      | `SVR-K8S-MAS`    | `192.168.40.10`|
| Worker 1    | `SVR-K8S-WOK-01` | `192.168.40.20`|
| Worker 2    | `SVR-K8S-WOK-02` | `192.168.40.30`|

---

## 1. Configuration de base (sur tous les nœuds)

```bash
sudo dnf update -y
sudo hostnamectl set-hostname <nom_du_serveur>  # exemple : SVR-K8S-MAS
```

Ajoutez dans `/etc/hosts` sur **tous** les nœuds :

```
192.168.40.10   SVR-K8S-MAS
192.168.40.20   SVR-K8S-WOK-01
192.168.40.30   SVR-K8S-WOK-02
```

---

Désactivation de SELinux et de swap (obligatoire)

```bash
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing/SELINUX=permissive/' /etc/selinux/config

sudo swapoff -a
sudo sed -i '/swap/d' /etc/fstab
```

>  *Kubernetes ne supporte pas SELinux en enforcing ni le swap actif.*

---

Activation des modules noyau et configuration réseau

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter
```

Configurer les paramètres réseau nécessaires :

```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```

> 📌 *Ces réglages activent la redirection IP et la compatibilité réseau entre pods.*

---

## 2. Firewall : ouverture des ports

Sur le **master** :
```bash
sudo firewall-cmd --permanent --add-port={6443,2379,2380,10250,10251,10252,10257,10259,179}/tcp
sudo firewall-cmd --permanent --add-port=4789/udp
sudo firewall-cmd --reload
```

Sur les **workers** :
```bash
sudo firewall-cmd --permanent --add-port={179,10250,30000-32767}/tcp
sudo firewall-cmd --permanent --add-port=4789/udp
sudo firewall-cmd --reload
```

>  *Le port 4789 est utilisé par Calico (VXLAN). Les ports kubelet, kube-apiserver et kube-proxy sont également essentiels.*

---

## 3. Installation de containerd (runtime) sur tous les nœuds

```bash
sudo dnf install -y yum-utils
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo dnf install -y containerd.io
```

Configuration de containerd :

```bash
sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml > /dev/null
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

sudo systemctl restart containerd
sudo systemctl enable containerd
```

> 📌 *Le paramètre `SystemdCgroup = true` est crucial pour que containerd fonctionne avec kubelet.*

---

## 4. Installation de kubeadm, kubelet et kubectl (tous les nœuds)

```bash
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.28/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.28/rpm/repodata/repomd.xml.key
EOF

sudo dnf makecache
sudo dnf install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
sudo systemctl enable --now kubelet
```

---

## 5. Initialiser le master (`SVR-K8S-MAS`)

```bash
sudo kubeadm init --pod-network-cidr=192.168.0.0/16
```

Configurer kubectl pour l’utilisateur courant :

```bash
mkdir -p $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

---

Joindre les nœuds workers au cluster

Sur chaque worker (`SVR-K8S-WOK-01` et `SVR-K8S-WOK-02`), exécutez la commande fournie par `kubeadm init` :

```bash
sudo kubeadm join 192.168.40.10:6443 --token <token> \
--discovery-token-ca-cert-hash sha256:<hash>
```

> *Cette commande relie le nœud au cluster en utilisant un certificat de confiance.*

---

## 6. Installer le CNI Calico (sur le master uniquement)

```bash
curl -O https://raw.githubusercontent.com/projectcalico/calico/v3.27.2/manifests/calico.yaml
```

Modifier **calico.yaml** :
- Dans la `ConfigMap` nommée `calico-config` : mettre `veth_mtu: "1440"`
- Dans les variables d’environnement du conteneur `calico-node` : ajouter
  ```yaml
  - name: CALICO_IPV4POOL_CIDR
    value: "192.168.0.0/16"
  ```

Puis appliquer :
```bash
kubectl apply -f calico.yaml
```

> *Calico gère la communication réseau entre les pods via VXLAN.*

---

## 7. Vérifications

Voir l’état des nœuds :
```bash
kubectl get nodes
```

Vérifier les pods système :
```bash
kubectl get pods -A
```

Tu dois voir :
- Tous les `calico-node` et `calico-kube-controllers` en `Running`
- Tous les `coredns` en `Running`
- Tous les nœuds en `Ready`

---