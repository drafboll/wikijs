---
title: Projet CEPH
description: 
published: 1
date: 2026-09-25T15:49:50.465Z
tags: 
editor: markdown
dateCreated: 2026-05-27T16:07:03.098Z
---

# Projet CEPH - Déploiement du Cluster Hyperconvergé

## Introduction : À quoi sert ce projet ?

Ce tutoriel a pour objectif de vous guider pas à pas dans la création d'une infrastructure **hyperconvergée** de niveau entreprise (Enterprise-Grade), en partant de simples serveurs Linux vierges. 
 
**L'hyperconvergence**, c'est l'art de réunir sur les mêmes machines physiques à la fois le stockage (les disques durs) et la puissance de calcul (les machines virtuelles). C'est exactement l'architecture utilisée par les géants du Cloud (AWS, Google) ou par des solutions comme Proxmox et VMware vSAN. 

Ici, nous allons construire notre propre "mini-cloud" en utilisant les technologies open-source les plus puissantes du marché : **Ceph** pour le stockage distribué, et **KVM/Libvirt** pour la virtualisation.

### Ce que vous allez apprendre et maîtriser à la fin de ce tutoriel :
1. **Le Stockage Distribué avec Ceph :** Transformer 9 disques durs vierges en un seul espace de stockage géant, intelligent et capable de s'auto-réparer en cas de panne physique (Haute Disponibilité).
2. **La Virtualisation Sécurisée (Rootless) :** Configurer un hyperviseur pour que des utilisateurs standards puissent créer et gérer des machines virtuelles sans jamais avoir besoin des droits administrateur (`sudo`).
3. **Le Stockage Fichier Partagé (CephFS) :** Créer un "NAS" intégré au cluster pour stocker les images ISO, monté automatiquement au démarrage des serveurs grâce à Systemd.
4. **La Migration à Chaud (Live Migration) :** Déplacer la mémoire (RAM) d'une machine virtuelle allumée d'un serveur physique à un autre instantanément, sans aucune coupure réseau pour l'utilisateur.
5. **L'Administration Système Avancée :** Gérer les clés d'authentification SSH (Full Mesh), configurer le pare-feu (Firewalld), et maîtriser les règles de sécurité strictes du noyau Linux (SELinux et permissions).

Préparez-vous, vous allez construire une architecture robuste, sécurisée et évolutive !

## Étape 1 : Vérification de la présence des disques vierges
Si vous avez besoin de l'aide pour creer les machines vous pouvez vous aidez de ce tuto : https://wiki.alexandre-faye.fr/fr/linux/intro_CEPH

Chaque nœud doit posséder 3 disques durs de 100 Go non partitionnés dédiés à Ceph. (À vérifier sur chaque machine).
```bash
lsblk
```
![capture_d'écran_2026-05-27_170103.png](/ceph/capture_d'écran_2026-05-27_170103.png)

## Étape 2 : Initialisation du cluster (Bootstrap avec cephadm sur Node1)
Création du premier nœud maître (Seed) et configuration du mot de passe permanent pour le Dashboard. À faire uniquement sur le Node 1.
```bash
# Lancement du bootstrap
sudo cephadm bootstrap \
  --mon-ip 192.168.50.20 \
  --initial-dashboard-password cephadmin \
  --dashboard-password-noupdate \
  --allow-fqdn-hostname
```
![capture_d'écran_2026-05-27_171616.png](/ceph/capture_d'écran_2026-05-27_171616.png)
![capture_d'écran_2026-05-27_171704.png](/ceph/capture_d'écran_2026-05-27_171704.png)

## Étape 3 : Déploiement et Injection des Clés Secrètes sur Node2 et Node3
Méthode sécurisée avec allocation de pseudo-terminal (`-t`) pour contourner la restriction de saisie du mot de passe `sudo` à distance. À faire uniquement depuis le Node 1.

```bash
# 1. Extraction et préparation de la clé publique de Ceph sur Node1
sudo cp /etc/ceph/ceph.pub /home/rocky/ceph.pub
sudo chown rocky:rocky /home/rocky/ceph.pub

# 2. Transfert de la clé publique vers l'espace utilisateur de Node2 et Node3
scp /home/rocky/ceph.pub rocky@192.168.50.30:/home/rocky/
scp /home/rocky/ceph.pub rocky@192.168.50.40:/home/rocky/

# 3. Injection de la clé dans les autorisations root de Node2 (avec terminal forcé)
ssh -t rocky@192.168.50.30 "sudo mkdir -p /root/.ssh && sudo cp /home/rocky/ceph.pub /root/.ssh/authorized_keys"

# 4. Injection de la clé dans les autorisations root de Node3 (avec terminal forcé)
ssh -t rocky@192.168.50.40 "sudo mkdir -p /root/.ssh && sudo cp /home/rocky/ceph.pub /root/.ssh/authorized_keys"
```

## Étape 4 : Ajout officiel des nœuds au cluster (depuis Node 1)
Maintenant que les clés sont en place, intégrez vos nœuds au cluster et vérifiez leur présence.

![capture_d'écran_2026-05-27_175800.png](/ceph/capture_d'écran_2026-05-27_175800.png)

```bash
# Ajout du Node 2
sudo ceph orch host add node2 192.168.50.30

# Ajout du Node 3
sudo ceph orch host add node3 192.168.50.40

# Vérification de la présence des 3 nœuds
sudo ceph orch host ls
```
![capture_d'écran_2026-05-29_125624.png](/ceph/capture_d'écran_2026-05-29_125624.png)

## Étape 5 : Haute Disponibilité des Monitors et Managers
Pour assurer la survie du cluster en cas de panne, on répartit le "cerveau" (Monitors et Managers) sur les 3 nœuds.

```bash
# Déployer 3 Monitors (mon) et 3 Managers (mgr)
sudo ceph orch apply mon 3
sudo ceph orch apply mgr 3
```
`Le Monitor (MON) :` C'est le chef d'orchestre absolu. Il détient la "carte" (la Cluster Map). Il sait exactement où se trouve chaque morceau de donnée, quel serveur est vivant, et quel disque est tombé en panne. Si les Monitors meurent, les serveurs ne savent plus où sont les données, et le cluster s'arrête net.

`Le Manager (MGR) :` C'est l'assistant du chef. Il s'occupe de rassembler les statistiques, de faire tourner l'interface web (le Dashboard), de surveiller l'espace libre, etc.

## Étape 6 : Déploiement des Disques de Stockage (OSD)

Si les Monitors sont le cerveau du cluster, les **OSD (Object Storage Daemon)** en sont les muscles. Un OSD est un processus (un démon) qui prend le contrôle exclusif d'un disque dur physique. C'est lui qui effectue le vrai travail : il stocke la donnée, la réplique sur les autres serveurs pour la sécurité, et s'auto-répare si un disque voisin tombe en panne.

Dans cette étape, nous allons demander à Ceph de scanner nos serveurs, de trouver les disques vierges, et de les transformer en OSD.

```bash
# 1. Vérifier la détection des disques vierges
sudo ceph orch device ls

# 2. Formater et transformer tous les disques vierges en OSD
sudo ceph orch daemon add osd node1:data_devices=/dev/sda,/dev/sdb,/dev/sdc
sudo ceph orch daemon add osd node2:data_devices=/dev/sda,/dev/sdb,/dev/sdc
sudo ceph orch daemon add osd node3:data_devices=/dev/sda,/dev/sdb,/dev/sdc
```
![capture_d'écran_2026-05-27_175914.png](/ceph/capture_d'écran_2026-05-27_175914.png)
```bash
# 3. Vérifier que les 9 disques sont bien en ligne (9 up, 9 in)
sudo ceph -s

# 4. Afficher la répartition physique des disques sur les nœuds
sudo ceph osd tree
```
![capture_d'écran_2026-05-27_180151.png](/ceph/capture_d'écran_2026-05-27_180151.png)

## Étape 7 : Création du Pool de Stockage Bloc (RBD) pour les VMs

### Comprendre ce que l'on fait :
Une machine virtuelle se comporte comme un véritable ordinateur : pour installer son système d'exploitation (Alpine, Linux, Windows), elle refuse de s'installer dans un simple dossier réseau. Elle exige de voir un **disque dur physique brut** (un périphérique "Bloc") qu'elle pourra formater elle-même.
C'est le rôle de **RBD (RADOS Block Device)** : il crée des disques durs virtuels qui trompent la VM. La VM croit écrire sur un disque dur classique, mais en arrière-plan, Ceph découpe la donnée en milliers de blocs de 4 Mo et les éparpille sur nos 9 disques physiques (OSD) pour décupler les performances et la sécurité.

Pour que RBD fonctionne, nous devons lui créer un espace de stockage dédié appelé **Pool**.

**1. Création du Pool "grp3" et calcul des PGs :**
Dans Ceph, les données ne sont pas jetées directement sur les disques. Elles sont d'abord triées dans des paniers virtuels appelés **PG (Placement Groups)**.
La règle d'or pour calculer le nombre de paniers est : `(Nombre d'OSD × 100) / Nombre de copies`.

> **C'est quoi un PG (Placement Group) ?**
> 
> Dans un cluster Ceph, il peut y avoir des millions, voire des milliards de fichiers. Si le "cerveau" du cluster (les Monitors) devait mémoriser l'emplacement exact de chaque petit fichier sur chaque disque dur, la base de données serait gigantesque et le système deviendrait très lent.
> 
> Pour éviter ça, Ceph utilise des **Placement Groups (PG)**. Un PG est un "panier" ou un "sac de tri" virtuel.
> 
> **Voici comment ça marche (La règle de l'entonnoir) :**
> 1. Un gros fichier (ex: le disque dur de notre VM) est découpé en milliers de petits **Objets**.
> 2. Une formule mathématique extrêmement rapide (l'algorithme CRUSH) "jette" ces objets dans différents paniers virtuels (**les PGs**).
> 3. Ce sont ces paniers (PGs) qui sont ensuite stockés et répliqués physiquement sur les disques durs (**les OSD**).
> 
> **L'analogie de la Poste :** Au lieu de mémoriser l'adresse de chaque enveloppe individuelle, le centre de tri regroupe les lettres par Code Postal (le PG), et distribue les gros sacs de tri aux bons camions de livraison (les OSD). C'est ce qui rend Ceph si rapide et si facile à réparer en cas de panne !
>

Dans notre infrastructure : `(9 disques x 100) / 3 copies = 300`.
Pour optimiser l'informatique binaire, Ceph arrondit toujours à la puissance de 2 supérieure. Nous allons donc utiliser **512 PGs**.

```bash
# On crée le pool nommé "grp3" avec 512 PGs. "replicated" signifie que la donnée sera copiée.
sudo ceph osd pool create grp3 512 512 replicated
```

**2. Verrouillage de la sécurité :**
Par défaut, Ceph peut essayer de modifier le nombre de PGs tout seul, ce qui consomme beaucoup de CPU. Nous allons désactiver ça, et forcer la sécurité à 3 copies (les données de la VM existeront toujours sur 3 serveurs différents).

```bash
# On gèle le nombre de PGs à 512 définitivement
sudo ceph osd pool set grp3 pg_autoscale_mode off

# On force la réplication sur 3 serveurs distincts
sudo ceph osd pool set grp3 size 3
```

**3. Activation de l'illusion du "Disque Dur" (RBD) :**
Maintenant que notre espace (Pool) est créé et sécurisé, nous disons à Ceph que cet espace ne servira pas à stocker des fichiers classiques, mais exclusivement des disques durs virtuels de type "Bloc".

```bash
# On identifie ce pool comme étant dédié au Block Device (RBD)
sudo ceph osd pool application enable grp3 rbd

# On initialise la structure interne pour que les VMs puissent y brancher leurs disques virtuels
sudo rbd pool init grp3
```
![capture_d'écran_2026-05-27_185215.png](/ceph/capture_d'écran_2026-05-27_185215.png)

---

## Étape 8 : Mode Rootless - Sécurisation Libvirt (Full Session)
Pour respecter les normes de sécurité (isolation des VMs du système hôte), nous configurons Libvirt pour que l'utilisateur `rocky` puisse tout gérer sans `sudo`.

> **À FAIRE SUR LES 3 NŒUDS :**
{.is-danger}


**1. Configuration des droits et groupes :**
```bash
sudo usermod -aG kvm,libvirt,qemu rocky
```

**2. Modification de l'URI par défaut :**
Modifiez les fichiers `/etc/libvirt/libvirt.conf` et `/etc/libvirt/libvirt-admin.conf` en ajoutant à la fin la ligne :
```text
uri_default = "qemu:///session"
```

**4. Autoriser le réseau de la session utilisateur :**
```bash
echo "allow br0" | sudo tee /etc/qemu-kvm/bridge.conf
```

**5. Application :**
**DÉCONNECTEZ-VOUS ET RECONNECTEZ-VOUS** à votre session SSH pour activer les groupes, puis vérifiez avec la commande :
```bash
virsh uri
# Doit retourner : qemu:///session
```

---


## Étape 9 : Autorisation d'accès entre Libvirt et Ceph

**Le principe du Moindre Privilège**
Ceph et Libvirt (l'hyperviseur KVM) sont deux logiciels totalement indépendants. Par défaut, Ceph est une forteresse : il bloque toutes les connexions entrantes.
Pour que l'hyperviseur puisse ranger les disques virtuels dans le stockage Ceph, il lui faut un mot de passe.

Pour des raisons de sécurité strictes, nous n'allons pas donner à Libvirt le mot de passe Administrateur de Ceph. Nous allons lui créer un "badge d'accès" sur mesure (`client.libvirt`) qui lui donne le droit de lire et d'écrire **uniquement** dans le dossier des machines virtuelles (le pool `grp3`). Ensuite, nous allons apprendre ce mot de passe à Libvirt via son coffre-fort XML.

> **PARTIE A : À FAIRE UNIQUEMENT SUR LE NODE 1**
{.is-danger}

On crée le "badge d'accès" dans la base de données centrale de Ceph, avec des droits strictement limités au pool `grp3`.

```bash
# Création de l'utilisateur Libvirt dans Ceph et récupération de sa clé
sudo ceph auth get-or-create client.libvirt mon 'profile rbd' osd 'profile rbd pool=grp3'
```

Voici comment devrait etre la sortie : 
```bash
[client.libvirt]
        key = AQBxaWt...[une_très_longue_suite_de_lettres]...==
```

> Ici penser bine à garder la clé qui vous sera afficher dans la console 
{.is-warning}


---

> **PARTIE B : À FAIRE SUR LE NODE 1, PUIS SUR LE NODE 2, PUIS SUR LE NODE 3**
{.is-danger}

On configure l'hyperviseur local (KVM) pour qu'il retienne cette clé d'accès de manière sécurisée. On utilise un UUID fixe pour garantir que les 3 serveurs auront exactement la même configuration (indispensable pour la migration de VMs).

**1. Extraction sécurisée de la clé vers un fichier temporaire :**
Pour éviter que la clé ne se retrouve en clair dans l'historique des commandes Linux (.bash_history), on récupère la clé et on la redirige directement dans un fichier texte :
```bash
sudo ceph auth get-key client.libvirt > client.libvirt.secret
```

**2. Création du fichier décrivant le secret (l'enveloppe XML) :**
```bash
vi secret.xml
```

Et ecrire
```xml
<secret ephemeral='no' private='no'>
  <uuid>9a8b7c6d-1234-5678-abcd-102030405060</uuid>
  <usage type='ceph'>
    <name>client.libvirt secret</name>
  </usage>
</secret>
```

**3. Importation et injection de la clé :**
```bash
# Importation dans Libvirt
virsh secret-define --file secret.xml

# Injection de la clé Ceph dans ce secret
virsh secret-set-value --secret 9a8b7c6d-1234-5678-abcd-102030405060 --file client.libvirt.secret

# Nettoyage : On supprime immédiatement le fichier contenant la clé en clair !
rm client.libvirt.secret
```
Alternative : Si votre version de KVM est ancienne et renvoie une erreur avec l'option --file, utilisez cette commande qui ne laisse pas non plus de trace dans l'historique :
virsh secret-set-value --secret 9a8b7c6d-1234-5678-abcd-102030405060 --base64 $(cat client.libvirt.secret)

**PARTIE C : VÉRIFICATION SUR TOUS LES NŒUDS**
Avant de déclarer officiellement le stockage, il est crucial de s'assurer que Libvirt a bien mémorisé le mot de passe sur l'ensemble du cluster. C'est ce qui garantira la possibilité de faire de la "Live Migration" (déplacer une VM d'un serveur à l'autre sans l'éteindre).

**1. Vérifier la création de l'enregistrement de sécurité (sur les 3 nœuds) :**
```bash
virsh secret-list
```
*(Résultat attendu : La console doit afficher un tableau contenant votre UUID `9a8b7c6d-1234-5678-abcd-102030405060` associé à l'usage `ceph client.libvirt secret`)*

**2. Vérifier que la clé secrète est bien mémorisée à l'intérieur (sur les 3 nœuds) :**
```bash
virsh secret-get-value 9a8b7c6d-1234-5678-abcd-102030405060
```
*(Résultat attendu : La console doit afficher exactement la clé secrète générée par Ceph à la Partie A, soit `Votre clé que vous avez récuperé plus haut`)*

---

## Étape 10 : Création du Pool de stockage KVM (RBD)

**Le Raccourci de Stockage (Datastore)**
Maintenant que Libvirt possède le mot de passe de Ceph (Étape 9), il faut lui indiquer *l'adresse* du stockage.

Libvirt a besoin qu'on lui déclare formellement un "Storage Pool" (un Raccourci). Nous allons créer un fichier de configuration XML qui explique à l'hyperviseur comment contacter Ceph.
Ainsi, lors de la création de nos futures machines virtuelles, au lieu de retaper toutes les adresses IP et les identifiants de Ceph, nous aurons simplement à dire à la VM : *"Stocke ton disque dur dans le raccourci nommé `ceph_pool`"*.

> **Rappel : À FAIRE SUR LE NODE 1, PUIS LE NODE 2, PUIS LE NODE 3.**
{.is-danger}


**1. Création du fichier décrivant le pool de stockage :**
```bash
vi pool-ceph.xml
```

*(Appuyez sur la touche `i`, collez le texte ci-dessous, puis appuyez sur `Échap`, tapez `:wq` et validez avec `Entrée`)* :
```xml
<pool type='rbd'>
  <name>ceph_pool</name>
  <source>
    <name>grp3</name>
    <host name='192.168.50.20'/>
    <host name='192.168.50.30'/>
    <host name='192.168.50.40'/>
    <auth username='libvirt' type='ceph'>
      <secret uuid='9a8b7c6d-1234-5678-abcd-102030405060'/>
    </auth>
  </source>
</pool>
```

**2. Déclaration et activation du stockage :**
```bash
# Déclaration du stockage dans Libvirt
virsh pool-define --file pool-ceph.xml

# Démarrage du stockage
virsh pool-start ceph_pool

# Activation du démarrage automatique (pour qu'il survive à un redémarrage du serveur)
virsh pool-autostart ceph_pool

# Vérification finale (Doit afficher State: running)
virsh pool-info ceph_pool
```

## Étape 11 : Création du système de fichiers (CephFS)

**À quoi sert le MDS ?**
Contrairement au stockage Bloc (RBD) de l'étape précédente, nous voulons créer ici un vrai dossier partagé en réseau (type NAS/NFS) pour y ranger nos images ISO. Pour que les machines Linux comprennent ce dossier, il faut gérer une arborescence (`/dossier/fichier`), des noms de fichiers, et des permissions.

Or, nos disques durs (OSD) ne savent pas faire ça : ils ne gèrent que des données brutes.
Nous devons donc déployer un nouveau service : le **MDS (Metadata Server)**. 
Le MDS est comme le bibliothécaire du cluster : il retient le nom des fichiers et l'arborescence des dossiers. Quand un utilisateur veut lire une ISO, il demande le chemin au MDS, puis il va télécharger le fichier **directement sur les OSD**. Le MDS ne porte jamais la vraie donnée, ce qui évite les goulots d'étranglement !

>  **À FAIRE UNIQUEMENT SUR LE NODE 1**
{.is-danger}


**1. Déployer les serveurs de métadonnées (MDS) :**
Pour gérer des dossiers, Ceph a besoin du service MDS (Metadata Server). On en déploie 3 pour la haute disponibilité.
```bash
sudo ceph orch apply mds cephfs --placement="3"
```

**2. Créer le volume CephFS :**
```bash
sudo ceph fs volume create cephfs
```

**3. Créer le badge d'accès pour ce dossier :**
Nous créons un utilisateur spécifique ("iso") qui a le droit de lire et écrire dans ce partage. 
*(Notez bien la clé secrète générée à la fin de cette commande, elle servira à l'étape suivante !)*
```bash
sudo ceph fs authorize cephfs client.iso / rw
```

---

## Étape 12 : Montage automatique et dynamique du CephFS (Systemd Automount)

**L'Automount**
Conformément au cahier des charges, nous montons ce dossier réseau sur `/mnt/cephfs`. 
> Au lieu de forcer une connexion permanente qui pourrait bloquer le serveur au démarrage si le réseau est lent, nous utilisons la méthode "Automount". Le serveur démarre instantanément, place un "piège" sur le dossier, et ne se connecte à Ceph qu'à la milliseconde exacte où quelqu'un essaie d'ouvrir le dossier `/mnt/cephfs`.

> **À FAIRE SUR LE NODE 1, PUIS SUR LE NODE 2, PUIS SUR LE NODE 3.**
{.is-danger}


**1. Création du dossier cible :**
```bash
sudo mkdir -p /mnt/cephfs
```

**2. Sauvegarde sécurisée du mot de passe CephFS :**
Pour que Systemd puisse se connecter tout seul, nous inscrivons la clé obtenue à l'Étape 11 dans un fichier caché.
*(Remplacez `VOTRE_CLE_ISO_ICI` par la clé de l'utilisateur client.iso !)*
```bash
sudo vi /etc/ceph/cephfs.secret
```
Mettre votre clé privé :
```bash
sudo chmod 600 /etc/ceph/cephfs.secret
```

**3. Création du fichier de montage (Le "Travailleur") :**
*Règle stricte de Systemd : Le nom du fichier doit correspondre exactement au chemin du montage (`/mnt/cephfs` devient `mnt-cephfs.mount`).*
```bash
sudo vi/lib/systemd/system/mnt-cephfs.mount
```
*(Collez ce bloc. Notez qu'il n'y a volontairement pas de section `[Install]` à la fin)* :
```ini
[Unit]
Description=Montage CephFS pour les ISOs
DefaultDependencies=no
After=network.target network-online.target remote-fs-pre.target
Wants=network-online.target

[Mount]
What=192.168.50.20,192.168.50.30,192.168.50.40:/
Where=/mnt/cephfs
Type=ceph
Options=name=iso,secretfile=/etc/ceph/cephfs.secret,_netdev,x-systemd.mount-timeout=30
```

**4. Création du déclencheur dynamique (Le "Piège Automount") :**
C'est ce fichier qui va surveiller le dossier de manière invisible et lancer le fichier précédent uniquement quand on clique sur le dossier.
```bash
sudo vi /lib/systemd/system/mnt-cephfs.automount
```
*(Collez ce bloc)* :
```ini
[Unit]
Description=Automontage dynamique du CephFS pour les ISOs

[Automount]
Where=/mnt/cephfs

[Install]
WantedBy=multi-user.target
```

**5. Activation du système Automount :**
On recharge Systemd, on s'assure que l'ancien système fixe est éteint, et on active notre nouveau déclencheur dynamique.
```bash
sudo systemctl daemon-reload
sudo systemctl disable mnt-cephfs.mount
sudo systemctl enable --now mnt-cephfs.automount
```

**6. Application des droits pour le mode Rootless (ACL) :**
On donne le droit de lecture à l'utilisateur "rocky" sur le dossier monté
```bash
sudo dnf install acl -y
sudo setfacl -Rm u:rocky:rwx /mnt/cephfs
```

**7. Vérification :**
```bash
df -h | grep cephfs
```
*(Résultat attendu : La console doit afficher le point de montage `/mnt/cephfs` avec la capacité totale de votre cluster Ceph).*
Donc si vous redemarrez les machines le cluster ceph remontera tout seul au demarrage

![capture_d'écran_2026-05-29_145828.png](/ceph/capture_d'écran_2026-05-29_145828.png)

---

## Étape 13 : Préparation SSH et Pare-feu pour la Migration
Pour migrer les VMs à chaud dans tous les sens sans mot de passe, chaque nœud doit posséder sa clé SSH et l'envoyer aux deux autres.

**1. Échange des clés (Le Full Mesh) :**

**SUR LE NODE 1 (192.168.50.20) :**
```bash
ssh-keygen -t rsa -b 4096 -N "" -f ~/.ssh/id_rsa
ssh-copy-id rocky@192.168.50.30
ssh-copy-id rocky@192.168.50.40
```
**SUR LE NODE 2 (192.168.50.30) :**
```bash
ssh-keygen -t rsa -b 4096 -N "" -f ~/.ssh/id_rsa
ssh-copy-id rocky@192.168.50.20
ssh-copy-id rocky@192.168.50.40
```
**SUR LE NODE 3 (192.168.50.40) :**
```bash
ssh-keygen -t rsa -b 4096 -N "" -f ~/.ssh/id_rsa
ssh-copy-id rocky@192.168.50.20
ssh-copy-id rocky@192.168.50.30
```

**2. Ouverture des ports (À faire sur les 3 NŒUDS) :**
```bash
sudo firewall-cmd --add-port=49152-49215/tcp --permanent
sudo firewall-cmd --reload
```

---

## Étape 14 : Téléchargement d'une image ISO partagée
Pour installer une machine virtuelle, il nous faut un système d'exploitation. Pour gagner du temps et ne pas saturer le réseau, nous allons télécharger **Alpine Linux**, une distribution extrêmement légère (environ 50 Mo), parfaite pour valider notre architecture.

>  **À FAIRE UNIQUEMENT SUR LE NODE 1 :**
{.is-danger}


**1. Télécharger l'ISO directement dans le dossier partagé CephFS :**
```bash
curl -o /mnt/cephfs/alpine.iso https://dl-cdn.alpinelinux.org/alpine/v3.19/releases/x86_64/alpine-virt-3.19.1-x86_64.iso
```

**2. Vérifier la magie du stockage distribué (Optionnel, à faire sur le Node 2 ou 3) :**
```bash
ls -lh /mnt/cephfs/
```
*(Résultat : Le fichier `alpine.iso` apparaît instantanément sur les autres serveurs sans aucune copie manuelle !)*

---

## Étape 15 : Création de la Machine Virtuelle hyperconvergée
C'est le moment de vérité. Nous allons demander à KVM de créer une machine virtuelle nommée `vm-ceph`. KVM ira lire le CD d'installation dans le CephFS, et créera le disque dur virtuel de la machine directement dans notre stockage bloc sécurisé (`ceph_pool`).

**À FAIRE UNIQUEMENT SUR LE NODE 1 :**

**1. Lancer la création de la VM :**
```bash
virt-install \
  --name vm-ceph \
  --memory 1024 \
  --vcpus 1 \
  --disk pool=ceph_pool,size=2,bus=virtio \
  --cdrom /mnt/cephfs/alpine.iso \
  --network user,model=virtio \
  --graphics vnc,listen=0.0.0.0 \
  --noautoconsole
```
![capture_d'écran_2026-05-29_140805.png](/ceph/capture_d'écran_2026-05-29_140805.png)
*(Explication de la commande : `--disk pool=ceph_pool,size=2` ordonne à KVM de découper un volume de 2 Go dans notre cluster Ceph. Les données de ce disque seront automatiquement répliquées sur les 3 nœuds grâce à la configuration de notre pool).*

**2. Vérifier que la VM est bien allumée :**
```bash
virsh list --all
```
*(Résultat attendu : La console doit afficher `vm-ceph` avec le statut **running**).*

## Étape 16 : Migration à chaud (Live Migration) de la VM
Le but ultime de l'hyperconvergence est de pouvoir déplacer une machine virtuelle d'un serveur physique à un autre sans aucune coupure pour l'utilisateur. 
Puisque le disque de notre `vm-ceph` est déjà partagé sur le réseau via Ceph, il nous suffit de transférer son état d'exécution (la RAM) et son fichier de configuration (le XML) par le réseau via SSH.

**À FAIRE SUR LE NODE 1 :**

**1. Vérifier que la VM est bien allumée sur le Node 1 :**
```bash
virsh list --all
```

**1. Lancer la migration vers le Node 2 :**
```bash
virsh migrate --live --verbose --domain vm-alpine qemu+ssh://rocky@192.168.50.20/session
```
![capture_d'écran_2026-05-29_140900.png](/ceph/capture_d'écran_2026-05-29_140900.png)

**2. Vérifier le succès :**
* Sur le **Node 1**, tapez : `virsh list --all` *(La VM a disparu).*
* Sur le **Node 2**, tapez : `virsh list --all` *(La VM est arrivée en état running).*

*(Résultat attendu : La machine `vm-ceph` apparaît maintenant sur le Node 2 avec le statut **running**. La migration à chaud est un succès total !)*

## Schéma synthétique 
```
========================================================================
 1. LA COUCHE CLIENT (Ce que vous utilisez au quotidien)
========================================================================

 [ Votre Machine Virtuelle ]              [ Vos Serveurs Node 1, 2, 3 ]
  (Besoin d'un disque dur)                 (Besoin d'un dossier partagé)
             |                                          |
             V                                          V
       +-----------+                              +-----------+
       |    RBD    |                              |  CephFS   | 
       |  (Block)  |                              | (Fichier) | <--- Géré par le 
       +-----------+                              +-----------+        MDS
             |                                          |
=============|==========================================|===============
 2. LA COUCHE LOGIQUE (L'organisation dans Ceph)
=============|==========================================|===============
             |                                          |
             V                                          V
 +---------------------------------------------------------------------+
 |                               POOLS (Les Tiroirs)                   |
 |                                                                     |
 |  +--------------------+   +-------------------+ +-----------------+ |
 |  |     Pool RBD       |   |   Pool CephFS     | |  Pool CephFS    | |
 |  |    (Disques VM)    |   |   (Fichiers ISO)  | |   (Metadata)    | |
 |  |                    |   |                   | |                 | |
 |  |  [PG]  [PG]  [PG]  |   | [PG]  [PG]  [PG]  | | [PG]  [PG]  [PG]| |
 |  |   (Les Cartons)    |   |   (Les Cartons)   | |  (Les Cartons)  | |
 |  +--------------------+   +-------------------+ +-----------------+ |
 +---------------------------------------------------------------------+
       |
       |  L'algorithme "CRUSH" prend les cartons (PG) et les jette 
       |  intelligemment sur les disques durs physiques.
       V
========================================================================
 3. LA COUCHE PHYSIQUE ET LE CERVEAU (Le matériel et la gestion)
========================================================================

 +---------------------------------------------------------------------+
 |                       LES DISQUES PHYSIQUES                         |
 |                                                                     |
 |      [ NODE 1 ]             [ NODE 2 ]             [ NODE 3 ]       |
 |   - OSD 1 (/dev/sdb)     - OSD 3 (/dev/sdb)     - OSD 5 (/dev/sdb)  |
 |   - OSD 2 (/dev/sdc)     - OSD 4 (/dev/sdc)     - OSD 6 (/dev/sdc)  |
 +---------------------------------------------------------------------+

 +---------------------------------------------------------------------+
 |                     LE CERVEAU DU CLUSTER                           |
 |                                                                     |
 |  [ MON 1, 2, 3 ] (Monitors) : Maintiennent la carte du cluster      |
 |                               et assurent le système de vote.       |
 |                                                                     |
 |  [ MGR 1, 2 ]    (Managers) : Gèrent l'Orchestrateur, les stats     |
 |                               et le Dashboard Web.                  |
 |                                                                     |
 |  [ MDS 1, 2 ]    (Metadata) : Les "Bibliothécaires", indispensables |
 |                               pour gérer l'arborescence des         |
 |                               fichiers du CephFS.                   |
 +---------------------------------------------------------------------+
 ```