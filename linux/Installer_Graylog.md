---
title: Installer Graylog
description: 
published: 1
date: 2026-09-25T15:48:16.057Z
tags: graylog, nas
editor: markdown
dateCreated: 2025-03-30T12:43:59.381Z
---

# Installation de Graylog sur Red Hat 9 et exporter les logs sur un nas

## Prérequis
- Machine Red Hat 9 à jour
- Droits root ou sudo
- Pare-feu désactivé ou ports requis ouverts (27017, 9000, 8999...)
- Système de fichiers recommandé : **XFS**

---
 
## 1. Mise à jour du système et configuration de base

```bash
sudo dnf update -y
sudo hostnamectl set-hostname svr-graylog-01
sudo timedatectl set-timezone Europe/Paris
```

Ajouter l'entrée dans `/etc/hosts` si besoin :
```bash
sudo vi /etc/hosts
# Ajouter :
192.168.150.206 svr-graylog-01
```

---

## 2. Créer une partition ISCSI, un système de fichiers et monter le volume pour les données Graylog

### Connexion à un partage iSCSI (NAS)

### Sur le NAS :

- Créer un volume ZVOL (ex: `graylog_iscsi`)
- Créer une cible iSCSI avec un `Extent` associé à ce volume
- Autoriser l'accès depuis l'IP du serveur (ex : `192.168.150.208`)

---

### Sur le serveur Graylog (Rocky Linux 9) :

#### 🔹 Installer les outils iSCSI :

```bash
sudo dnf install -y iscsi-initiator-utils
sudo systemctl enable --now iscsid
```

#### 🔹 Découvrir les cibles disponibles (remplace l’IP par celle de ton NAS) :

```bash
sudo iscsiadm -m discovery -t st -p 192.168.150.207
```

```bash
sudo iscsiadm -m node --targetname iqn.2005-10.org.freenas.ctl:lun1 --portal 192.168.150.207 --login
```

### Côté client (svr-graylog-01)
```bash
sudo parted /dev/sda mklabel gpt
sudo parted -a opt /dev/sda mkpart primary ext4 0% 100%
sudo mkfs.ext4 /dev/sda1

sudo mkdir -p /mnt/graylog-data
sudo mount /dev/sda1 /mnt/graylog-data
```

### Ajout dans `/etc/fstab` pour montage automatique au boot :
```bash
sudo vi /etc/fstab
```
```bash
/dev/sda1 /mnt/graylog-data ext4 defaults,_netdev 0 0
```
Puis teste avec :
```bash
sudo systemctl daemon-reload
sudo mount -a
```
Créer l'utilisateur `graylog` si ce n'est pas déjà fait :

```bash
sudo useradd --system --home /var/lib/graylog-server --shell /sbin/nologin graylog-datanode
```

Création des dossiers :
```bash
sudo groupadd graylog-shared
sudo usermod -aG graylog-shared graylog
sudo usermod -aG graylog-shared graylog-datanode

sudo mkdir -p /mnt/graylog-data/journal
sudo mkdir -p /mnt/graylog-data/datanode/opensearch/logs
sudo mkdir -p /mnt/graylog-data/datanode/opensearch/data

sudo chown -R root:graylog-shared /mnt/graylog-data
sudo chmod -R 770 /mnt/graylog-data
sudo find /mnt/graylog-data -type d -exec chmod g+s {} \;

```
Tester les droits d'écritures :
```bash
sudo -u graylog touch /mnt/graylog-data/journal/.testfile
sudo -u graylog-datanode touch /mnt/graylog-data/datanode/opensearch/logs/.testfile
```
---

## 3. Installation de MongoDB (v7.0)

MongoDB est une base de données NoSQL conçue pour stocker et gérer les données de manière flexible, évolutive et performante. Intégrée à la pile Graylog, MongoDB sert de base de métadonnées pour le système.

La documentation officielle de MongoDB propose un tutoriel utile sur l'installation avec Red Hat. Pour plus d'informations sur l'installation de MongoDB pour une utilisation avec Graylog, nous vous recommandons de suivre la procédure ci-dessous.

Créer le fichier de repo :
```bash
sudo vi /etc/yum.repos.d/mongodb-org.repo
```
```bash
[mongodb-org-6.0]
name=MongoDB Repository
baseurl=https://repo.mongodb.org/yum/redhat/$releasever/mongodb-org/6.0/x86_64/
gpgcheck=1
enabled=1
gpgkey=https://www.mongodb.org/static/pgp/server-6.0.asc
```

Installation :
```bash
sudo dnf install -y mongodb-org
```

Par défaut, MongoDB n'écoute que localement. Vous devez modifier le paramètre ou dans le fichier de configuration de MongoDB afin que le port soit lié à une adresse d'interface ou à un nom d'hôte, ou qu'il écoute sur toutes les interfaces. Voir les exemples de configuration ci-dessous.
```
sudo vi /etc/mongod.conf
``` 
Pour écouter sur toutes les interfaces :
```
net:
  port: 27017
  bindIpAll: true
```

Démarrage du service :
```bash
sudo systemctl daemon-reload
sudo systemctl enable mongod --now
```

---

## 4. Installation du Data Node
Le nœud de données est un composant de l'architecture Graylog conçu pour gérer l'ingestion, le traitement et l'indexation des journaux, en gérant la communication avec OpenSearch pour un stockage et une interrogation efficaces des journaux. Comme indiqué dans l'architecture principale, le service du nœud de données doit être géré sur son propre nœud, distinct de Graylog et de MongoDB.

Pour installer Data Node, nous vous recommandons de suivre le processus ci-dessous : 
```bash
sudo rpm -Uvh https://packages.graylog2.org/repo/packages/graylog-6.1-repository_latest.rpm
sudo dnf install -y graylog-datanode
```

Configuration de la mémoire virtuelle :
```bash
sudo sysctl -w vm.max_map_count=262144
echo 'vm.max_map_count=262144' | sudo tee /etc/sysctl.d/99-graylog-datanode.conf
sudo sysctl --system
```

Créez votre password_secret. Il s'agit d'une chaîne sécurisée, générée aléatoirement, utilisée pour chiffrer les données sensibles du système : A GARDER PRESIEUSEMENT 
```bash
< /dev/urandom tr -dc A-Z-a-z-0-9 | head -c96; echo
```

Ouvrez le fichier de configuration du nœud de données :
```
sudo vi /etc/graylog/datanode/datanode.conf
``` 

```ini
password_secret = votre_secret_genere
opensearch_heap = 4g
mongodb_uri = mongodb://localhost:27017/graylog
path_data = /mnt/graylog-data/datanode
opensearch_data_location = /mnt/graylog-data/datanode/opensearch/data
opensearch_logs_location = /mnt/graylog-data/datanode/opensearch/logs

```

Dans ce fichier, vous devez ajuster les paramètres du segment de mémoire jvm.optionsdu service Data Node, comme indiqué dans la section « Configurations supplémentaires » . Ce service nécessite très peu de mémoire et peut être configuré à 1 ou 2 Go, selon vos besoins. Il est recommandé de définir la même taille pour les deux ; ainsi, définir un minimum de 2 Go Xms2get un maximum de 2 Go Xmx2gse présente comme suit :
```
sudo vi /etc/graylog/datanode/jvm.options
``` 

```ini
-Xms2g
-Xmx2g
```

Démarrage du service Data Node :
```bash
sudo systemctl daemon-reload
sudo systemctl enable graylog-datanode.service --now
```
Ajoute une dépendance dans le service pour qu’il attende que le montage soit fait avant de démarrer :
```ini
sudo systemctl edit graylog-datanode
```
Et ajoute ce contenu :

```ini
[Unit]
After=network-online.target remote-fs.target
RequiresMountsFor=/mnt/graylog-data
```

```
sudo systemctl daemon-reexec
sudo systemctl daemon-reload
sudo systemctl restart graylog-datanode
```
---

## 5. Installation de Graylog Server
Une fois l'installation de Data Node terminée, vous devez installer le service principal Graylog. L'architecture principale est conçue pour que Graylog et MongoDB soient installés sur le même nœud.

Pour installer Graylog, nous vous recommandons de suivre le processus ci-dessous : 


```bash
sudo rpm -Uvh https://packages.graylog2.org/repo/packages/graylog-6.1-repository_latest.rpm
sudo dnf install -y graylog-server
```

Générer le mot de passe admin (SHA256) :
```bash
echo -n "votreMotDePasse" | sha256sum | cut -d" " -f1
```

Ouvrez le fichier de configuration du serveur Graylog :
```
sudo vi /etc/graylog/server/server.conf
```

```ini
root_password_sha2 = <hash_du_mot_de_passe>
password_secret = <le_même_secret_que_data_node>
```
Définissez la http_bind_addressvaleur du fichier de configuration Graylog sur le nom d'hôte public ou l'adresse IP publique sur laquelle le serveur Web et API Graylog écoutera les requêtes HTTP entrantes. (Cette propriété est commentée par défaut et répertoriée sous la # HTTP settingssection du fichier de configuration)

Ajustez les paramètres de votre journal. Nous vous recommandons de configurer l'âge maximal de votre journal à 72 heures et sa taille en fonction du volume total de journaux prévu sur une période de 72 heures. Ainsi, si le volume quotidien prévu est de 30 Go, la taille maximale doit être ajustée à 90 Go, comme dans l'exemple suivant :
```ini
http_bind_address = 0.0.0.0:9000
message_journal_max_age = 72h
message_journal_max_size = 90gb
message_journal_dir = /mnt/graylog-data/journal
```

Dans ce fichier, ajustez vos paramètres de tas à la moitié de la mémoire système, jusqu'à un maximum de 16 Go pour le service Graylog, comme recommandé dans la section Configuration supplémentaire . Il est recommandé de définir les valeurs minimale et maximale identiques. Ainsi, si vous définissez la valeur minimale à 2 Go, comme indiqué dans la section -Xms2g, et la valeur maximale à 2 Go, comme indiqué dans la section -Xmx2g, le paramètre pourrait ressembler à ceci : 

```
sudo vi /etc/sysconfig/graylog-server
```

```bash
GRAYLOG_SERVER_JAVA_OPTS="-Xms2g -Xmx2g -server -XX:+UseG1GC -XX:-OmitStackTraceInFastThrow"
```

Démarrage du service :
```bash
sudo systemctl daemon-reload
sudo systemctl enable graylog-server --now
```

---

## 6. Firewall (si activé)

```bash
sudo firewall-cmd --permanent --add-port=9000/tcp
sudo firewall-cmd --permanent --add-port=27017/tcp
sudo firewall-cmd --permanent --add-port=8999/tcp
sudo firewall-cmd --permanent --add-port=9200/tcp
sudo firewall-cmd --permanent --add-port=443/tcp
sudo firewall-cmd --permanent --add-port=12201/tcp
sudo firewall-cmd --reload
```

---

## Accès Web

Accéder à l'interface Graylog depuis :
```
http://<IP_DU_SERVEUR>:9000
```
Identifiants de prévol : voir les logs `/var/log/graylog-server/server.log`

Lors du premier démarrage de Graylog, tu ne te connectes pas avec le mot de passe que tu as mis dans server.conf (root_password_sha2). Ce mot de passe ne sera actif qu'après le "preflight".
```
sudo cat /var/log/graylog-server/server.log | grep "and password"
```
---

## Fin de l'installation
Votre instance Graylog est maintenant installée et fonctionnelle sur Red Hat 9 avec stockage NFS et backend Data Node.


## Configuration HTTPS de Graylog avec certificat auto-signé (SAN + Truststore)

---

### ☕ 0. Installer Java (si non déjà installé)

Graylog requiert une machine virtuelle Java (JVM) pour fonctionner.
Cette commande installe la version 17 d’OpenJDK (Java Development Kit), recommandée pour Graylog. Elle inclut les outils nécessaires pour exécuter et sécuriser l’application avec TLS.
```bash
sudo dnf install java-17-openjdk-devel -y
```

---

### 📁 1. Préparer le dossier TLS

```bash
sudo mkdir -p /opt/graylog/tls/
sudo chown graylog:graylog /opt/graylog/tls/
```

---

### 🛡️ 2. Générer un certificat auto-signé avec SAN (Subject Alternative Name)

Avant tout ouvrer le port 443
```bash
sudo firewall-cmd --permanent --add-port=443/tcp
sudo firewall-cmd --reload
```

Créer le fichier de configuration attention à bien changer les information avec l'ip de votre serveur :

```bash
cat <<EOF | sudo tee /opt/graylog/tls/openssl-graylog.cnf
[ req ]
distinguished_name = req_distinguished_name
x509_extensions = v3_req
prompt = no

[ req_distinguished_name ]
CN = 192.168.30.30

[ v3_req ]
subjectAltName = @alt_names

[ alt_names ]
IP.1 = 192.168.30.30
EOF
```

Générer la paire de clés + certificat :

```bash
sudo openssl req -x509 -nodes -days 365 \
  -newkey rsa:2048 \
  -keyout /opt/graylog/tls/private.key \
  -out /opt/graylog/tls/fullchain.pem \
  -config /opt/graylog/tls/openssl-graylog.cnf
```

---

### 3. Définir les permissions

```bash
sudo chown graylog:graylog /opt/graylog/tls/*
sudo chmod 600 /opt/graylog/tls/private.key
sudo chmod 644 /opt/graylog/tls/fullchain.pem
```

---

### 4. Ajouter le certificat dans le Truststore Java

> Le mot de passe par défaut est `changeit`

#### 4.1 Supprimer l’ancien alias (s’il existe)

```bash
sudo keytool -delete -alias graylog-selfsigned \
  -keystore /etc/graylog/graylog.jks \
  -storepass changeit || true
```

#### 4.2 Importer le certificat

```bash
sudo keytool -importcert \
  -alias graylog-selfsigned \
  -keystore /etc/graylog/graylog.jks \
  -file /opt/graylog/tls/fullchain.pem \
  -storepass changeit \
  -noprompt
```

---

### 5. Modifier la configuration de Graylog

Modifier `/etc/graylog/server/server.conf` :

```ini
http_bind_address = 0.0.0.0:443
http_publish_uri = https://192.168.30.30/
http_enable_tls = true
http_tls_cert_file = /opt/graylog/tls/fullchain.pem
http_tls_key_file = /opt/graylog/tls/private.key
```

---

### 6. Ajouter la configuration TLS au démarrage Java

Ce paramétrage est utilisé pour configurer les options de démarrage de Graylog pour y ajouter deux paramètres importants 
Éditer `sudo vi /etc/sysconfig/graylog-server` :

```bash
-Djavax.net.ssl.trustStore=/etc/graylog/graylog.jks
-Djavax.net.ssl.trustStorePassword=changeit
```
trustStore : indiquent à Java d'utiliser le fichier /etc/graylog/graylog.jks comme magasin de certificats de confiance.

trustStorePassword : précisent le mot de passe pour accéder à ce fichier (par défaut changeit, sauf si tu l’as changé).

À ajouter à la ligne :

```bash
GRAYLOG_SERVER_JAVA_OPTS="... -Djavax.net.ssl.trustStore=/etc/graylog/graylog.jks -Djavax.net.ssl.trustStorePassword=changeit"
```
Assure-toi de ne pas supprimer les autres options Java existantes (comme -Xms, -Xmx, etc.). Tu les ajoutes simplement à la suite dans la même ligne.

---

### 7. Autoriser l’usage du port 443

```bash
sudo sed -i '/^LimitNOFILE=/a AmbientCapabilities=CAP_NET_BIND_SERVICE' /usr/lib/systemd/system/graylog-server.service
sudo systemctl daemon-reexec
sudo systemctl daemon-reload
```

---

### 8. Redémarrer Graylog

```bash
sudo systemctl restart graylog-server
```

---

### 9. Accéder à l’interface web

![graylog_https.png](/graylog/graylog_https.png)
- URL : `https://192.168.30.30`
- Accepter le certificat auto-signé dans le navigateur
- Vérifier que toutes les données se chargent (plus d'erreur `TrustManager` dans les logs)

---
