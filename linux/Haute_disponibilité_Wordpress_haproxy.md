---
title: Haute disponibilité
description: 
published: 1
date: 2026-09-25T15:46:29.578Z
tags: haproxy, keepalive, wordpress
editor: markdown
dateCreated: 2025-02-05T09:37:22.598Z
---

# Tutoriel : Déploiement de l'infrastructure avec Galera, HAProxy et Nginx
![ha-webservers.png](/haproxy/ha-webservers.png)
## 1. Config machine vierge
Installer une machine rocky minima
https://rockylinux.org/fr-FR/download
Mettre 30 Go à la machine
![custom.png](/haproxy/custom.png)
![2._standart_partition.png](/haproxy/2._standart_partition.png)
![3._partition.png](/haproxy/3._partition.png)

Partitionnement manuel – Exemple Rocky Linux 9.5 (UEFI) avec 30Go 
 
| Point de montage | Taille        | Périphérique     | Système de fichiers | Type de partition      |
|------------------|---------------|------------------|----------------------|----------------------
| `/tmp`           | 1024 MiB      | `nvme0n1p3`       | ext4                 | Standard Partition      |
| `/var/tmp`       | 1023 MiB      | `nvme0n1p6`       | xfs                  | Standard Partition      |
| `/var/log`       | 1024 MiB      | `nvme0n1p5`       | ext4                 | Standard Partition      |
| `BIOS Boot`      | 2 MiB         | `nvme0n1p1`       | xfs                  | BIOS Boot Partition 
| `/`              | 27 GiB        | `nvme0n1p2`       | ext4                 | Standard Partition    |

>**Remarque :** Cette configuration utilise une partition **BIOS Boot**, ce qui est nécessaire uniquement en **mode Legacy (non-UEFI)**.  
> Si tu es en mode **UEFI**, il faut à la place :
> - Supprimer la partition BIOS Boot
> - Créer une partition `/boot/efi` de 300 MiB en **FAT32** de type **EFI System Partition**


## Ensuite penser à ne pas mettre de root !
Mettre le clavier en francais

![capture_d'écran_2025-02-05_104905.png](/haproxy/capture_d'écran_2025-02-05_104905.png)

## 2. Préparation des machines

### Installation des outils de base
```bash
sudo dnf install bash-completion vim policycoreutils-python-utils setools-console -y
```

### Configuration réseau
```bash
sudo nmtui
sudo systemctl restart NetworkManager
```
![nmtui.png](/haproxy/nmtui.png)

### Configuration des fichiers hôtes
```bash
sudo vim /etc/hostname
sudo vim /etc/hosts
```
Ajoutez les entrées suivantes :
```
192.168.50.100 mariadb1.ha.lan
192.168.50.101 mariadb2.ha.lan
192.168.50.102 mariadb3.ha.lan
192.168.50.110 web1.ha.lan
192.168.50.111 web2.ha.lan
192.168.50.120 haproxy1.ha.lan
192.168.50.121 haproxy2.ha.lan
```

---

## 3. Installation et configuration de Galera

### Configuration du pare-feu
```bash
sudo firewall-cmd --add-service=galera --permanent
sudo firewall-cmd --reload
sudo firewall-cmd --list-all
```

### Installation de MariaDB et Galera
```bash
sudo dnf install mariadb-server mariadb-server-galera galera -y
```

### Configuration du fichier Galera
```bash
sudo vi /etc/my.cnf.d/galera.cnf
```
Ajoutez la configuration suivante :
```
[galera]
wsrep_on                 = ON
wsrep_provider           = /usr/lib64/galera/libgalera_smm.so
wsrep_cluster_name       = "MariaDB Galera Cluster"
wsrep_provider_options   = "pc.ignore_quorum=1"
wsrep_cluster_address    = 
gcomm://192.168.50.100,192.168.50.101,192.168.50.102
binlog_format            = row
default_storage_engine   = InnoDB
innodb_autoinc_lock_mode = 2
innodb_buffer_pool_size  = 128M
wsrep_node_name          = DB1
wsrep_node_address       = 192.168.50.100
wsrep_sst_method         = rsync
bind-address             = 0.0.0.0
```

### Démarrage du cluster
```bash
sudo galera_new_cluster
sudo firewall-cmd --reload
sudo systemctl enable --now mariadb
```

### Sécurisation de MariaDB
```bash
sudo mysql_secure_installation
```

### Création des bases de données pour WordPress
```bash
sudo mysql -u root
```
Dans MySQL :
```sql
CREATE USER 'wordpress'@'%' IDENTIFIED BY 'password';
CREATE DATABASE wordpress CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
GRANT ALL PRIVILEGES ON wordpress.* TO 'wordpress'@'%';
FLUSH PRIVILEGES;

CREATE USER 'wordpress2'@'%' IDENTIFIED BY 'password';
CREATE DATABASE wordpress2 CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
GRANT ALL PRIVILEGES ON wordpress2.* TO 'wordpress2'@'%';
FLUSH PRIVILEGES;
```

---

## 4. Installation et configuration de HAProxy

### Installation de HAProxy et Keepalived
```bash
sudo dnf install haproxy keepalived -y
```

### Configuration du pare-feu
```bash
sudo firewall-cmd --add-service=galera --permanent
sudo firewall-cmd --reload
```

### Configuration de Keepalived
```bash
sudo vim /etc/keepalived/keepalived.conf
```
Ajoutez la configuration suivante pour le serveur **master** :
```
! Configuration File for keepalived

global_defs {
   notification_email {
     acassen@firewall.loc
     failover@firewall.loc
     sysadmin@firewall.loc
   }
}

# Instance VRRP pour les serveurs Nginx (VIP: 192.168.50.210)
vrrp_instance VI_HTTP {
    state MASTER
    interface ens160
    virtual_router_id 51  # ID unique
    priority 101
    advert_int 1

    authentication {
        auth_type PASS
        auth_pass 1111
    }

    virtual_ipaddress {
        192.168.50.210/32 brd 192.168.50.255 scope global  # VIP pour les serveurs Nginx
    }
}

# Instance VRRP pour les serveurs MariaDB (VIP: 192.168.50.220)
vrrp_instance VI_DB {
    state MASTER
    interface ens160
    virtual_router_id 52  # ID unique
    priority 101
    advert_int 1

    authentication {
        auth_type PASS
        auth_pass secret
    }

    virtual_ipaddress {
        192.168.50.220/32 brd 192.168.50.255 scope global # VIP pour les serveurs MariaDB
    }
}
```

Ajoutez la configuration suivante pour le serveur **slave** :
```bash
! Configuration File for keepalived
global_defs {
   notification_email {
     acassen@firewall.loc
     failover@firewall.loc
     sysadmin@firewall.loc
   }
}

# Instance VRRP pour les serveurs Nginx (VIP: 192.168.50.210)
vrrp_instance VI_HTTP {
    state BACKUP
    interface ens160
    virtual_router_id 51
    priority 100
    advert_int 1

    authentication {
        auth_type PASS
        auth_pass 1111
    }

    virtual_ipaddress {
        192.168.50.210/32 brd 192.168.50.255 scope global
    }
}

# Instance VRRP pour les serveurs MariaDB (VIP: 192.168.50.220)
vrrp_instance VI_DB {
    state BACKUP
    interface ens160
    virtual_router_id 52
    priority 100
    advert_int 1

    authentication {
        auth_type PASS
        auth_pass secret
    }

    virtual_ipaddress {
        192.168.50.220/32 brd 192.168.50.255 scope global
    }
}
```

Redémarrez Keepalived :
```bash
sudo systemctl enable --now keepalived
```

### Configuration de HAProxy
https://www.haproxy.com/documentation/
```bash
sudo vim /etc/haproxy/haproxy.cfg
```
Ajoutez la configuration suivante :
```
#---------------------------------------------------------------------
# VIP WORDPRESS
#---------------------------------------------------------------------
frontend main
    bind 192.168.50.210:80
    option http-server-close
#    option forwardfor
    default_backend             app-main


backend app-main
    balance     leastconn
    server  web1  192.168.50.111:80 check
    server  web2  192.168.50.110:80 check

#---------------------------------------------------------------------
# VIP DB
#---------------------------------------------------------------------
frontend mysql_front
  bind 192.168.50.220:3306
  mode tcp
  option tcplog
  default_backend mariadb

backend mariadb
  mode tcp                           # Mode TCP (pour MySQL)
  option tcpka                       # Keep-alive TCP
  option redispatch                  # Réattribue la connexion si le serveur est indisponible
  balance leastconn                   # Répartition de charge selon le plus petit nombre de connexions
  retries 2                          # Réduit le nombre de tentatives pour accélérer l'échec d'un serveur
  timeout connect 3s                 # Timeout plus court pour établir une connexion
  timeout server 10s                 # Timeout réduit pour éviter d'attendre trop longtemps un serveur
  timeout client 10s                 # Timeout client ajusté
  server mariadb1 192.168.50.100:3306 weight 2 check inter 1s rise 2 fall 1 on-marked-down shutdown-sessions
  server mariadb2 192.168.50.101:3306 weight 3 check inter 1s rise 2 fall 1 on-marked-down shutdown-sessions
  server mariadb3 192.168.50.102:3306 weight 1 check inter 1s rise 2 fall 1 on-marked-down shutdown-sessions
```

## si vous n'arrivez pas à redemarrer haproxy 
```bash
sudo semanage port -a -t http_port_t -p tcp 3306
sudo semanage port -l | grep 3306
sudo systemctl restart haproxy

```

---

## 5. Installation et configuration de Nginx

### Configuration du pare-feu
```bash
sudo firewall-cmd --add-service=http --permanent
sudo firewall-cmd --add-service=https --permanent
sudo firewall-cmd --reload
```

### Installation de PHP et Nginx
```bash
sudo dnf install -y nginx php php-fpm php-mysqlnd php-curl php-xml php-mbstring php-gd php-zip php-intl
sudo systemctl enable --now nginx
sudo systemctl enable --now php-fpm
```

### Configuration de PHP-FPM
```bash
sudo vim /etc/php-fpm.d/www.conf
```
Modifiez les valeurs :
```
user = nginx
group = nginx
listen = 127.0.0.1:9000
```

### Installation de WordPress
```bash
sudo wget https://wordpress.org/latest.tar.gz
sudo tar -xzvf latest.tar.gz
sudo mv wordpress /var/www/
sudo chown -R nginx:nginx /var/www/wordpress
sudo chmod -R 755 /var/www/wordpress
```

### Configuration de Nginx pour WordPress
```bash
sudo vim /etc/nginx/conf.d/wordpress.conf
```
Ajoutez la configuration suivante à modifier selon le serveur sur lequel on fait la conf:
```
server {
    listen 80;
    server_name 192.168.50.110;

    root /var/www/wordpress;
    index index.php index.html;

    location / {
        try_files $uri $uri/ /index.php?$args;
    }

    location ~ \.php$ {
        include /etc/nginx/fastcgi_params;
        fastcgi_pass 127.0.0.1:9000;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    }

    location ~ /\.ht {
        deny all;
    }
}
```

### Finalisation et redémarrage des services
```bash
sudo setsebool -P httpd_can_network_connect on
sudo setsebool -P httpd_read_user_content on
sudo nginx -t
sudo systemctl restart nginx php-fpm
```


# 6.TEST
Vous pouvez faire un test en mettant en pause la db1, db3, web1 et haproxy1 et voir si tout fonctionne

