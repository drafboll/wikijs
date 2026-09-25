---
title: OpenStack Multinœud (Kolla-Ansible)
description: 
published: 1
date: 2026-09-25T15:49:07.548Z
tags: 
editor: markdown
dateCreated: 2026-04-29T08:29:48.825Z
---

# Tutoriel de Déploiement OpenStack Multinœud (Kolla-Ansible)

Bienvenue dans ce retour d'expérience ! Vous trouverez ici toute la méthodologie employée pour construire notre cloud OpenStack multinœud avec Kolla-Ansible sur Ubuntu 24.04. L'idée est simple : consigner chaque commande et chaque configuration pour vous permettre de remonter cette architecture de A à Z, sans prise de tête.

---
 
## 1. Architecture retenue

### 1.1 Nœuds
L’infrastructure repose sur 4 machines virtuelles :

| Nœud | Rôle | IP management (ens37) | IP Externe (ens38) | IP Internet (NAT) |
| :--- | :--- | :--- | :--- | :--- |
| **controller** | plan de contrôle OpenStack | `10.10.10.10` | Non IP | 192.168.50.10 |
| **network1** | services réseau Neutron | `10.10.10.30` | Non IP | 192.168.50.30 |
| **compute1** | exécution des machines virtuelles | `10.10.10.20` | Non IP | 192.168.50.20 |
| **storage1** | stockage bloc Cinder | `10.10.10.40` | Non IP | 192.168.50.40 |

*L'adresse IP Virtuelle (VIP) d'accès au cluster est fixée à **10.10.10.100**.*


![capture_d'écran_2026-04-29_103431.png](/openstack/capture_d'écran_2026-04-29_103431.png)
### 1.2 Schéma d’architecture
```text
 +----------------------+
 | Navigateur           |
 | Horizon / API VIP    |
 | http://10.10.10.100  |
 +----------+-----------+
            |
       Réseau MGMT
       10.10.10.0/24
            |
 -----------------------------------------------------------------
 |             |               |              |                  |
+-------+--------+ +---------+-------+ +------+--------+ +---------+--------+
| controller     | | network1        | | compute1      | | storage1         |
| 10.10.10.11    | | 10.10.10.12     | | 10.10.10.13   | | 10.10.10.14      |
| Keystone       | | Neutron server  | | Nova compute  | | Cinder volume    |
| Glance         | | L3 agent        | | OVS agent     | | LVM backend      |
| Nova API       | | DHCP agent      | |               | | /dev/sdb         |
| Horizon        | | Metadata agent  | |               | |                  |
| MariaDB        | | OVS agent       | |               | |                  |
| HAProxy        | |                 | |               | |                  |
+-------+--------+ +---------+-------+ +------+--------+ +---------+--------+
            |                  |              |                  |
           NAT                NAT            NAT                NAT
     (accès internet)  (accès internet) (accès internet)  (accès internet)
      192.168.50.10     192.168.50.30    192.168.50.20     192.168.50.40   
            |                  |              |                  |
+-------+--------+ +---------+-------+ +------+--------+ +---------+--------+
                           |
                  Réseau provider EXT
                  lab-ext / physnet1
                  192.168.100.0/24
```
![capture_d'écran_2026-04-29_101453.png](/openstack/capture_d'écran_2026-04-29_101453.png)

### 1.3 Pourquoi cette architecture
Cette architecture a été choisie pour respecter le besoin du TP tout en gardant une topologie lisible :
* **Le nœud Controller (Le Cerveau) :**
  Il centralise l'intelligence du Cloud. Il héberge la base de données (**MariaDB**) qui stocke l'état de toute l'infrastructure, le bus de messages (**RabbitMQ**) qui permet aux services de se parler, les API publiques (Keystone, Nova-API, Glance) et l'interface Web (**Horizon**). Séparer le controller garantit que même si tes machines virtuelles consomment 100% du CPU de ton nœud Compute, ton interface d'administration restera fluide et accessible.

* **Le nœud Network (L'Aiguilleur) :**
  Le réseau logiciel (**Neutron**) est très gourmand en ressources. Ce nœud gère les routeurs virtuels, le NAT, l'attribution des adresses IP (serveurs DHCP virtuels) et le routage des paquets entre le réseau privé des VMs et l'extérieur. Isoler Neutron sur son propre nœud évite que le traitement massif de paquets réseau ne ralentisse l'exécution des machines virtuelles.

* **Le nœud Compute (Le Muscle) :**
  Il est exclusivement dédié à l'exécution des machines virtuelles via l'hyperviseur (KVM/QEMU) et **Nova-compute**. Son rôle est de fournir de la RAM et du CPU bruts. En le déchargeant des bases de données et du routage, on s'assure que 100% de ses ressources sont allouées aux VMs des utilisateurs.

* **Le nœud Storage (Le Coffre-fort) :**
  Le stockage bloc géré par **Cinder** nécessite de fortes capacités d'écriture/lecture sur les disques (I/O). Si Cinder était sur le nœud Controller, les écritures disques intenses d'une VM pourraient bloquer la base de données MariaDB. Avoir un nœud dédié avec son propre disque dur (LVM) permet de garantir des performances de stockage optimales.

---

## 2. Ressources des machines virtuelles

### 2.1 Répartition retenue
| Nœud | vCPU | RAM | Disque système | Disque supplémentaire |
| :--- | :--- | :--- | :--- | :--- |
| **controller** | 4 | 8 Go | 80 Go | - |
| **network1** | 2 |4 Go | 60 Go | - |
| **compute1** | 2 | 4 Go | 100 Go | - |
| **storage1** | 2 | 4 Go | 60 Go | 100 Go (`/dev/sdb`) |

---

## 3. Interfaces réseau par machine

### 3.1 Affectation des cartes réseau

* **Management (`ens37`)** : C'est le cœur du système. Kolla-Ansible l'utilise pour se connecter aux machines, et les services OpenStack l'utilisent pour communiquer entre eux (par exemple, Nova qui parle à Neutron) et pour faire transiter les données des VMs (Tunnels VXLAN).
* **Externe (`ens38`)** : Elle n'a **volontairement pas d'IP**. C'est Open vSwitch (OVS), piloté par Neutron, qui va s'en emparer. Elle sert de "tuyau" direct pour relier les routeurs virtuels de tes futures VMs vers le réseau physique extérieur. **192.168.100.0/24**.
* **Internet / NAT** : Utilisée pour télécharger les paquets Linux, les conteneurs Docker et les images systèmes (CirrOS).

### 3.2 Pourquoi cette séparation

* **L'interface de Management (`ens37` - IP Fixe requise) :**
  C'est la **colonne vertébrale** de ton Cloud. Elle a besoin d'une IP fixe et d'une stabilité absolue pour trois raisons majeures :
  1. **L'automatisation :** Kolla-Ansible utilise cette adresse IP pour se connecter en SSH et déployer les conteneurs Docker.
  2. **Le dialogue interne :** C'est sur ce réseau que Nova (Compute) demande à Neutron (Network) de créer un port, ou que Cinder (Storage) communique avec la base de données.
  3. **Le tunnel de données (VXLAN) :** Quand une VM sur le nœud Compute communique avec le routeur sur le nœud Network, leurs paquets sont "encapsulés" et voyagent de manière invisible à travers cette interface physique `ens37`.

* **L'interface Externe (`ens38` - Surtout AUCUNE adresse IP système) :**
  C'est le réseau "Provider", celui qui relie le Cloud OpenStack au vrai réseau physique (Internet ou ton réseau d'entreprise).
 * **Pourquoi aucune IP sur Ubuntu ?** Si tu donnes une adresse IP à cette carte dans Netplan, le noyau Linux d'Ubuntu va s'en emparer et essayer de router son propre trafic internet par là.
  * **Le rôle du commutateur virtuel :** Dans OpenStack, on veut que ce soit **Open vSwitch (OVS)** (piloté par Neutron) qui s'empare totalement de cette carte physique. OVS transforme cette vraie carte réseau en un "câble virtuel direct". Il va brancher les routeurs virtuels de tes VMs directement sur cette interface physique. Si Ubuntu et OVS se battent pour la même carte réseau, tu auras des conflits d'IP, des boucles de routage, et les Floating IPs (IPs publiques) de tes VMs ne fonctionneront pas.

---

## 4. Configuration réseau Netplan

*(À adapter selon vos interfaces exactes `enp10s0` ou `enp1s0` pour le NAT)*

### 4.1 Exemple pour controller (`/etc/netplan/01-netcfg.yaml`)
```yaml
network:
  version: 2
  ethernets:
    ens33:
      dhcp4: no
      addresses: [192.168.50.10/24]
      routes:
        - to: default
          via: 192.168.50.2
      nameservers:
        addresses: [8.8.8.8, 1.1.1.1]
    ens37:
      dhcp4: no
      addresses: [10.10.10.10/24]
    ens38:
      dhcp4: no
```
Appliquer :
```bash
sudo netplan apply
```
*(Faire de même sur les autres nœuds avec les IPs .20, .30, .40)*

---

## 5. Préparation Système et Utilisateurs

### 5.1 Installation des paquets de base
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y git vim curl net-tools python3-dev libffi-dev gcc libssl-dev python3-venv libdbus-1-dev libdbus-glib-1-dev pkg-config python3-docker
```
> **Explication :**
> Cette commande met à jour Ubuntu et installe les prérequis de compilation et les bibliothèques Python nécessaires pour qu'Ansible puisse fonctionner correctement.

### 5.2 Fichier `/etc/hosts` (Sur chaque nœud)
```text
10.10.10.10 controller
10.10.10.30 network1
10.10.10.20 compute1
10.10.10.40 storage1
```
Ce fichier agit comme un "annuaire" local. Au lieu que les serveurs cherchent l'adresse IP sur Internet, ils consultent ce fichier. Cela garantit que `controller` pointera toujours vers `10.10.10.10`, même si votre connexion internet est coupée. C'est indispensable pour que les composants d'OpenStack (comme RabbitMQ) puissent communiquer entre eux par leur nom.

### 5.3 Création de l’utilisateur kolla (Sur chaque nœud)
```bash
sudo useradd -m -s /bin/bash kolla
sudo passwd kolla
sudo usermod -aG sudo kolla
echo "kolla ALL=(ALL) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/kolla
```

### 5.4 Préparation SSH sans mot de passe
Se connecter en `kolla` sur **controller** :

> ATTENTION : Cette étape (génération de clé) se réalise UNIQUEMENT sur le nœud CONTROLLER.
{.is-danger}


```bash
su - kolla
ssh-keygen -t ed25519
ssh-copy-id -i /home/kolla/.ssh/id_ed25519.pub kolla@10.10.10.10
ssh-copy-id -i /home/kolla/.ssh/id_ed25519.pub kolla@10.10.10.20
ssh-copy-id -i /home/kolla/.ssh/id_ed25519.pub kolla@10.10.10.30
ssh-copy-id -i /home/kolla/.ssh/id_ed25519.pub kolla@10.10.10.40
```
**Explication des commandes :**
1.  **`useradd -m -s /bin/bash kolla`** : 
    * `-m` : Crée automatiquement un répertoire personnel (`/home/kolla`).
    * `-s /bin/bash` : Définit Bash comme terminal par défaut pour cet utilisateur.
2.  **`passwd kolla`** : Définit le mot de passe de l'utilisateur (requis pour la première connexion).
3.  **`usermod -aG sudo kolla`** : Ajoute l'utilisateur au groupe `sudo` pour qu'il puisse exécuter des commandes administratives.
4.  **`echo "kolla ALL=(ALL) NOPASSWD:ALL" | ...`** : C'est l'étape la plus critique. Elle autorise l'utilisateur `kolla` à utiliser `sudo` **sans que le système ne demande de mot de passe**. C'est obligatoire pour Ansible, qui doit installer des logiciels automatiquement sur les 4 nœuds à distance sans intervention humaine.

### 5.5 Verification

```
ssh controller hostname
ssh network1 hostname
ssh compute1 hostname
ssh storage1 hostname
```
![capture_d'écran_2026-04-28_154702.png](/openstack/capture_d'écran_2026-04-28_154702.png)

---

## 6. Préparation du stockage Cinder sur storage1

Sur le nœud **storage1**, préparer le volume LVM :

```bash
sudo pvcreate /dev/sdb
sudo vgcreate cinder-volumes /dev/sdb
```
**À quoi ça sert ?** Cinder (gestionnaire de disques) utilise LVM pour découper ce disque physique brut (`/dev/sdb`) en petits volumes virtuels à la demande pour les futures VMs.

---

## 7. Installation et Configuration de Kolla-Ansible (sur controller)

### 7.1 Installation

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y python3 python3-pip python3-venv python3-dev python3-docker \
                    git vim curl net-tools libffi-dev gcc libssl-dev libdbus-1-dev libdbus-glib-1-dev pkg-config
```
> **Explication des dépendances Python et système :**
> * **`python3`, `python3-pip`, `python3-venv`** : OpenStack et Ansible sont massivement écrits en Python. Il nous faut le langage de base, son gestionnaire de paquets (`pip`), et l'outil pour créer des environnements virtuels (`venv`) afin de ne pas polluer l'OS.
> * **`python3-dev`, `gcc`, `libffi-dev`, `libssl-dev`** : Ce sont des outils de compilation et des bibliothèques cryptographiques. Lorsque `pip` va installer certains modules complexes pour Ansible ou OpenStack, il aura besoin de compiler du code C en arrière-plan. Sans ces paquets, l'installation échouera.
> * **`python3-docker`** : Permet à Ansible de piloter le moteur Docker directement via Python.
```bash
python3 -m venv ~/venv
source ~/venv/bin/activate
pip install -U pip
pip install 'ansible-core>=2.15,<2.17'
pip install git+https://opendev.org/openstack/kolla-ansible@stable/2025.2
pip install docker dbus-python
```
> **Explication :**
> Utiliser un `venv` (Environnement Virtuel) permet d'isoler l'installation de Kolla-Ansible du reste du système Ubuntu. Cela évite que les modules Python du système d'exploitation n'entrent en conflit avec ceux requis par OpenStack.

### 7.2 Copie des fichiers de base
```bash
mkdir -p ~/openstack
cd ~/openstack
sudo mkdir -p /etc/kolla
sudo chown -R $USER:$USER /etc/kolla
cp -r ~/venv/share/kolla-ansible/etc_examples/kolla/* /etc/kolla
cp ~/venv/share/kolla-ansible/ansible/inventory/multinode ~/openstack/multinode
```
On copie les modèles de configuration fournis par défaut par Kolla vers les répertoires de travail locaux (`/etc/kolla` pour la configuration système, et `~/openstack` pour les inventaires) afin de pouvoir les personnaliser.

### 7.3 Fichier `/etc/kolla/globals.yml`
**Explication des paramètres clés :**
* **VIP Address (`10.10.10.100`) :** C'est une IP virtuelle gérée par HAProxy. Toutes les requêtes (API, interface Web Horizon) vont taper sur cette IP, ce qui permet de faire de la Haute Disponibilité si on avait plusieurs controllers.
* **Virt Type (`qemu`) :** Par défaut, OpenStack s'attend à être installé sur du métal (des vrais serveurs) et utilise `kvm`. Comme nous sommes déjà dans des VMs VMware, nous devons utiliser `qemu` pour faire de la virtualisation logicielle (virtualisation imbriquée) et éviter de faire planter l'hyperviseur physique.

```yaml
kolla_base_distro: "ubuntu"
kolla_install_type: "source"
openstack_release: "2025.2"

# Adresse VIP (HAProxy)
kolla_internal_vip_address: "10.10.10.100"

# Configuration des interfaces réseau
network_interface: "ens37"
api_interface: "ens37"
neutron_external_interface: "ens38"

# Composants à installer
enable_keystone: "yes"
enable_glance: "yes"
enable_nova: "yes"
enable_neutron: "yes"
enable_horizon: "yes"
enable_cinder: "yes"
enable_cinder_backend_lvm: "yes"
cinder_volume_group: "cinder-volumes"

# Optimisations VM et Réseau
nova_compute_virt_type: "qemu"
neutron_plugin_agent: "openvswitch"
enable_fluentd: "yes"
enable_haproxy: "yes"
```

### 7.4 Fichier `~/openstack/multinode`
Mettez à jour les 4 premières sections de ce long fichier :

```ini
[control]
controller ansible_host=127.0.0.1 ansible_connection=local ansible_user=kolla ansible_become=true

[network]
network1 ansible_host=10.10.10.30 ansible_user=kolla ansible_become=true ansible_ssh_private_key_file=/home/kolla/.ssh/id_ed25519

[compute]
compute1 ansible_host=10.10.10.20 ansible_user=kolla ansible_become=true ansible_ssh_private_key_file=/home/kolla/.ssh/id_ed25519

[storage]
storage1 ansible_host=10.10.10.40 ansible_user=kolla ansible_become=true ansible_ssh_private_key_file=/home/kolla/.ssh/id_ed25519

[monitoring]
controller

[loadbalancer]
controller
```
*(Conservez le reste du fichier tel quel avec ses groupes `[monitoring]`, `[loadbalancer]`, etc.)*

**Explication :**
C'est la cartographie de votre Cloud. Ansible lit ce fichier pour savoir exactement quelle machine héberge quel rôle. Sans ce fichier, le script de déploiement ne saurait pas où installer Neutron ou Nova.

---

## 8. Déploiement

### 8.1 Préparation et Mots de passe
```bash
kolla-genpwd
kolla-ansible install-deps
kolla-ansible bootstrap-servers -i ~/openstack/multinode
```

**Explication :**
OpenStack utilise des dizaines de mots de passe pour faire communiquer ses bases de données et ses API en interne. Cette commande génère des mots de passe ultra-sécurisés et les stocke dans `/etc/kolla/passwords.yml`.

### 8.2 Vérification et Installation
Phase 1 : Bootstrap
```bash
kolla-ansible bootstrap-servers -i ~/openstack/multinode
```
Phase 2 : Prechecks
```bash
kolla-ansible prechecks -i ~/openstack/multinode
```
![capture_d'écran_2026-04-28_164641.png](/openstack/capture_d'écran_2026-04-28_164641.png)

Phase 3 : Deploy
```bash
kolla-ansible deploy -i ~/openstack/multinode
```

> **Explication des 3 commandes :**
> 1. **Bootstrap :** Se connecte aux nœuds, installe le moteur Docker et configure les noyaux Linux.
> 2. **Prechecks :** Vérifie que tout est prêt (RAM suffisante, ports réseau non bloqués, groupe LVM `cinder-volumes` bien présent).
> 3. **Deploy :** Télécharge les images Docker de chaque composant OpenStack depuis internet et les lance sur les machines définies dans le fichier d'inventaire. C'est l'installation proprement dite.


### 8.3 Post-déploiement et Chargement de l'environnement

```bash
kolla-ansible post-deploy -i ~/openstack/multinode
```
![capture_d'écran_2026-04-28_164709.png](/openstack/capture_d'écran_2026-04-28_164709.png)

```bash
source ~/venv/bin/activate
pip install python-openstackclient
source /etc/kolla/admin-openrc.sh
```
**Explication :**
`post-deploy` génère le fichier `admin-openrc.sh`. La commande `source` charge ce fichier dans votre terminal actuel. Il contient votre login (admin), votre mot de passe, et l'adresse de votre Cloud (`10.10.10.100`). Sans cette étape, le client en ligne de commande (CLI) d'OpenStack rejetterait toutes vos commandes.

---

## 9. Création de l'Environnement Cloud (Ressources OpenStack)

### 9.1 Image système et Gabarit (Flavor)
```bash
wget [https://download.cirros-cloud.net/0.6.2/cirros-0.6.2-x86_64-disk.img](https://download.cirros-cloud.net/0.6.2/cirros-0.6.2-x86_64-disk.img)
openstack image create "cirros" --file cirros-0.6.2-x86_64-disk.img --disk-format qcow2 --container-format bare --public
openstack flavor create --ram 512 --disk 1 --vcpus 1 m1.tiny
```
**Explication :**
* **Glance (Image)** : Vous téléchargez CirrOS (un Linux ultra-léger conçu pour tester les clouds) et l'ajoutez au catalogue d'images d'OpenStack.
* **Nova (Flavor)** : Vous créez un "gabarit" (comme une taille de T-shirt) définissant la puissance de la future machine (512 Mo de RAM, 1 CPU).

### 9.2 Le Réseau Externe (Provider Network)
```bash
openstack network create public1 --external --provider-network-type flat --provider-physical-network physnet1
openstack subnet create public1-subnet --network public1 --subnet-range 192.168.50.0/24 --no-dhcp --gateway 192.168.50.1 --allocation-pool start=192.168.50.100,end=192.168.50.200
```
> **Explication :**
> Ce réseau explique à OpenStack comment le réseau de votre entreprise (ou de votre lab VMware) est configuré. Il dit à Neutron de se brancher sur la carte physique `ens38` et de réserver des IPs de 100 à 200 pour les assigner publiquement à vos futures VMs (Les Floating IPs).

### 9.3 Le Réseau Privé (Tenant Network) et le Routeur
```bash
openstack network create private1
openstack subnet create private1-subnet --network private1 --subnet-range 10.0.0.0/24

openstack router create router1
openstack router set router1 --external-gateway public1
openstack router add subnet router1 private1-subnet
```
**Explication :**
On crée ici un réseau local virtuel isolé (`10.0.0.0/24`) pour les VMs. Le `router1` joue le rôle de passerelle (comme une box internet) : il connecte le réseau privé au réseau public et fait la traduction d'adresses (NAT) pour que les VMs puissent accéder à Internet.

---

## 10. Lancement de la première Instance

### 10.1 Pare-feu (Security Groups) et Clés SSH
```bash
ssh-keygen -t ed25519
openstack keypair create --public-key ~/.ssh/id_ed25519.pub mykey
```

```bash
openstack security group rule create --proto icmp default
openstack security group rule create --proto tcp --dst-port 22 default
```
OpenStack utilise une politique de "Zero Trust" (Aucune confiance). Par défaut, tout le trafic entrant vers une VM est bloqué. Il faut explicitement créer des règles de pare-feu pour autoriser le ping (`icmp`) et l'accès terminal (`tcp 22`). La clé SSH est injectée dans la VM au démarrage pour s'y connecter sans mot de passe.

### 10.2 Lancement de la VM
```bash
openstack server create --flavor m1.tiny --image cirros --network private1 --key-name mykey test-vm
```
C'est l'aboutissement du Cloud ! Nova trouve de la place sur `compute1`, télécharge l'image CirrOS depuis Glance, demande à Neutron de créer une carte réseau connectée à `private1`, et boot la VM.

### 10.3 Floating IP (Exposition sur le réseau externe)
```bash
openstack floating ip create public1
openstack server add floating ip test-vm <IP_Générée>
```

La VM est cachée dans son réseau privé 10.0.0.x. Pour qu'elle soit joignable depuis votre PC Windows, on lui alloue une Floating IP (ex: 192.168.50.100). Le routeur virtuel fera une règle de redirection (DNAT) pour lier l'IP publique à l'IP privée de la VM.

### 10.4 Ajout de Stockage Bloc (Cinder)
```bash
openstack volume create --size 2 volume1
openstack server add volume test-vm volume1
```
Valide le bon fonctionnement de `storage1`. Cinder découpe un morceau de 2 Go dans le groupe LVM, le transporte via le réseau en protocole iSCSI, et le branche à chaud sur la VM (qui le verra comme un disque `/dev/vdb`).

### 10.5 Test SSH à la machine virtuelle
Pour tester que votre machine est bien fonctionnelle :

```bash
ssh cirros@<IP_Générée>
# Le mot de passe par défaut de l'image CirrOS est souvent : gocubsgo ou cubswin:)
```
![capture_d'écran_2026-04-28_182739.png](/openstack/capture_d'écran_2026-04-28_182739.png)

---

## 11. Interface Web Horizon

* **URL :** `http://10.10.10.100` (IP de la VIP HAProxy)
* **Utilisateur :** `admin`
* **Mot de passe :** Obtenez-le en tapant :
```bash
grep keystone_admin_password /etc/kolla/passwords.yml
```
Horizon est le tableau de bord graphique officiel d'OpenStack. Il appelle les mêmes API que vos lignes de commandes pour vous permettre de gérer vos instances, réseaux et volumes visuellement.
![capture_d'écran_2026-04-28_175106.png](/openstack/capture_d'écran_2026-04-28_175106.png)

![capture_d'écran_2026-04-29_152831.png](/openstack/capture_d'écran_2026-04-29_152831.png)

---

## 12. Maintenance et Reset

### 12.1 Synchronisation après changement du mot de passe Admin

Si vous modifiez le mot de passe de l'utilisateur `admin` via l'interface Web (Horizon), vos variables d'environnement actuelles deviendront invalides (**Erreur 401**). Vous devez impérativement mettre à jour vos fichiers de configuration locaux.

1.  **Modifier le fichier de configuration :**
    Ouvrez le fichier `admin-openrc.sh` avec l'éditeur `nano` :
    ```bash
    sudo nano /etc/kolla/admin-openrc.sh
    ```

2.  **Mettre à jour la variable `OS_PASSWORD` :**
    Cherchez la ligne correspondante et remplacez l'ancienne valeur par votre nouveau mot de passe :
    ```bash
    export OS_PASSWORD=votre_nouveau_mdp
    ```

3.  **Recharger l'environnement :**
    Pour que le terminal actuel prenne en compte ce changement, vous devez "sourcer" à nouveau le fichier :
    ```bash
    source /etc/kolla/admin-openrc.sh
    ```

4.  **Vérification :**
    Testez la validité de vos nouveaux identifiants avec la commande suivante :
    ```bash
    openstack token issue
    ```

> [!IMPORTANT]
> La modification manuelle est nécessaire car la commande `kolla-ansible post-deploy` ne connaît que le mot de passe généré aléatoirement dans le fichier `passwords.yml`. Elle ne peut pas deviner le mot de passe que vous avez choisi manuellement sur l'interface Web.

### 12.1 Arrêt Propre (Shutdown)
```bash
su - kolla
source ~/venv/bin/activate
kolla-ansible stop -i ~/openstack/multinode
```
*Éteindre les VMs physiques dans l'ordre : compute1 > storage1 network1 > controller.*
**Explication :** Cette commande demande à Docker d'arrêter doucement tous les conteneurs (bases de données, API) pour éviter de corrompre les données avant une extinction physique.

### 12.2 Redémarrage (Restart)
*Démarrer les VMs physiques dans l'ordre : controller > network1 > storage1 > compute1.*
```bash
su - kolla
source ~/venv/bin/activate
kolla-ansible deploy -i ~/openstack/multinode
```
Relancer un "deploy" est la meilleure façon de redémarrer un cluster Kolla. Ansible va vérifier chaque conteneur et s'assurer qu'il est bien relancé dans son état nominal.

### 12.3 Reset Complet (Remise à Zéro)
En cas d'erreur irrécupérable, cette procédure détruit tout et remet l'environnement à neuf :

```bash
# 1. Stoppe les services
kolla-ansible stop -i ~/openstack/multinode

# 2. Force la suppression de tous les conteneurs et volumes Docker sur chaque nœud
sudo docker rm -f $(docker ps -aq)
sudo docker volume rm $(docker volume ls -q)

# 3. Supprime les fichiers de configuration Kolla
kolla-ansible destroy -i ~/openstack/multinode --yes-i-really-really-mean-it

# 4. Nettoyage manuel des dossiers résiduels (Sur tous les nœuds)
sudo rm -rf /etc/kolla/* /var/lib/docker/* /var/log/kolla/*
sudo systemctl restart docker

# 5. Formatage du disque de stockage (Sur storage1)
sudo lvremove -y cinder-volumes
sudo vgremove -y cinder-volumes
sudo pvremove -y /dev/sdb
sudo pvcreate /dev/sdb
sudo vgcreate cinder-volumes /dev/sdb
```
**Explication :** Cette méthode "Terre brûlée" efface absolument tout (Bases de données, configurations, volumes Cinder). Cela permet de relancer un déploiement depuis l'étape 6 sur des bases parfaitement saines sans avoir à réinstaller les systèmes d'exploitation Ubuntu.