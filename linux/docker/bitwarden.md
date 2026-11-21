---
title: Mise à jour de Bitwarden auto-hébergé
description: 
published: 1
date: 2026-11-21T18:36:07.060Z
tags: 
editor: markdown
dateCreated: 2026-11-21T18:36:07.060Z
---

# Mise à jour de Bitwarden auto-hébergé (Docker)

## 0. Avant de commencer

- Compter **30 à 45 minutes** et prévoir une période où personne n'utilise le coffre.
- Avoir **accès à ton coffre sur un appareil** (application mobile ou extension déjà synchronisée) au cas où le serveur serait indisponible un moment.
- Vérifier l'espace disque (il faut au moins 7 Go libres, car `bwdata` fait environ 6 Go) :

```bash
df -h /opt
```

- Se placer dans le dossier d'installation :

```bash
cd /opt/bitwarden
```

---

## 1. Faire un export de sécurité du coffre (facultatif mais conseillé)

Dans l'interface web : **Outils → Exporter le coffre** → format `.json (chiffré)`.
Stocker le fichier hors du serveur (autre machine, clé USB).

---

## 2. Sauvegarde complète (OBLIGATOIRE)

Arrêter Bitwarden pour que la base soit cohérente :

```bash
cd /opt/bitwarden
./bitwarden.sh stop
```

Faire la copie (remplacer la date) :

```bash
cp -a bwdata bwdata-backup-AAAA-MM-JJ
echo "code retour cp : $?"
```

Le code retour doit être `0`.

**Vérifier que la copie est complète** :

```bash
diff -rq bwdata bwdata-backup-AAAA-MM-JJ
du -sh bwdata bwdata-backup-AAAA-MM-JJ
```

- `diff` ne doit **rien afficher** ;
- les deux tailles doivent être **identiques** (environ 5,8 Go).

> ❌ Si `diff` affiche des différences, supprimer la copie (`rm -rf bwdata-backup-AAAA-MM-JJ`) et la refaire. **Ne pas continuer** tant que la copie n'est pas identique.

Copier la sauvegarde sur une autre machine :

```bash
tar czf bwdata-backup-AAAA-MM-JJ.tar.gz bwdata-backup-AAAA-MM-JJ
scp bwdata-backup-AAAA-MM-JJ.tar.gz utilisateur@autre-machine:/chemin/
```

Redémarrer Bitwarden :

```bash
./bitwarden.sh start
```

Attendre que tout soit `healthy` (1 à 3 minutes) :

```bash
docker ps | grep bitwarden
```

---

## 3. Mise à jour

Toujours dans cet ordre.

### 3.1 Mettre à jour le script

```bash
./bitwarden.sh updateself
```

Répondre `y` si le script demande de confirmer (avertissement « running as root »).

### 3.2 Mettre à jour Bitwarden

```bash
./bitwarden.sh update
```

- Compter plusieurs minutes (téléchargement des images, redémarrage).
- **Ne pas interrompre** la commande.
- Succès : le message `Bitwarden is up and running!` s'affiche, **sans exception SQL**.

Si le message `Update not needed` apparaît, la version est déjà à jour : passer au point 3.3 pour être sûr que la base l'est aussi.

### 3.3 Mettre à jour la base de données

```bash
./bitwarden.sh updatedb
```

Succès : `Database update complete`.

Si cette commande affiche une erreur `SqlException ... Name does not resolve`, aller au point **4 (dépannage)**.

---

## 4. Dépannage : erreur « Name does not resolve »

### Symptôme

Au moment de la migration, le script affiche :

```
Unhandled exception. Microsoft.Data.SqlClient.SqlException ...
SocketException ... Name does not resolve
```

### Cause (constatée le 21/11/2026)

Dans `bwdata/scripts/run.sh`, le script ne donne accès au réseau Docker au conteneur d'installation que si la chaîne de connexion contient `Data Source=tcp:mssql,1433`. Sur ce serveur, la chaîne contient `Data Source=tcp:bitwarden-mssql,1433` : la condition est fausse, et le conteneur ne trouve pas la base.

### Vérifications

1. Nom d'hôte réellement configuré :

```bash
grep -o 'Data Source=[^;]*' bwdata/env/global.override.env
```

Résultat attendu : `Data Source=tcp:bitwarden-mssql,1433`.

2. Contenu actuel de la condition dans le script :

```bash
grep -n "Data Source" bwdata/scripts/run.sh
sed -n '188,198p' bwdata/scripts/run.sh
```

### Correctif (à refaire si `run.sh` a été réécrit)

Faire d'abord une copie :

```bash
cp bwdata/scripts/run.sh run.sh.bak
```

Modification 1 : adapter la condition au nom d'hôte réel.

```bash
sed -i 's|Data Source=tcp:mssql,1433|Data Source=tcp:bitwarden-mssql,1433|' bwdata/scripts/run.sh
```

Modification 2 : lancer le conteneur d'installation sur le réseau `docker_default`.

```bash
sed -i 's|--network container:\$MSSQL_ID|--network docker_default|' bwdata/scripts/run.sh
```

Contrôle des deux modifications :

```bash
grep -n "Data Source" bwdata/scripts/run.sh
grep -n "docker_network_args=" bwdata/scripts/run.sh
```

On doit voir `tcp:bitwarden-mssql,1433` et `--network docker_default`.

Relancer la migration :

```bash
./bitwarden.sh updatedb
```

### Test DNS (si ça échoue encore)

Vérifier que le nom de la base est visible sur le réseau :

```bash
docker network ls
docker run --rm --network docker_default busybox nslookup mssql
```

Si ce test échoue, le problème vient du DNS interne de Docker (vérifier `/etc/docker/daemon.json`, entrée `dns`).

---

## 5. Vérifications finales

1. Tous les conteneurs sont `healthy`, en nouvelle version :

```bash
docker ps | grep bitwarden
```

On doit voir 11 conteneurs `bitwarden-*` (nginx, admin, sso, notifications, api, icons, events, identity, attachments, web, mssql).

2. Se connecter à http://lock.alexandre-faye.fr :
   - ouvrir un élément,
   - en créer un de test,
   - le supprimer.

3. Pas d'erreur SQL dans les logs de l'API (le message `LaunchDarkly SDK key invalid` est normal) :

```bash
docker logs bitwarden-api --tail 30
```

---

## 6. Retour arrière (si le coffre ne fonctionne plus)

La sauvegarde contient aussi les numéros de version : la restaurer ramène l'ancienne version et l'ancienne base.

```bash
cd /opt/bitwarden
./bitwarden.sh stop
mv bwdata bwdata-casse
cp -a bwdata-backup-AAAA-MM-JJ bwdata
./bitwarden.sh start
```

Attendre que les conteneurs soient `healthy`, puis tester la connexion.

---

## 7. Nettoyage

Quand tout fonctionne depuis plusieurs jours :

- supprimer les anciennes sauvegardes locales (garder la plus récente + la copie sur l'autre machine) ;
- supprimer `bwdata-casse` s'il existe ;
- garder `run.sh.bak` jusqu'à la prochaine mise à jour pour pouvoir comparer.

---

## 8. Bonnes pratiques

- Utiliser le script avec un **utilisateur dédié** (`bitwarden`, membre du groupe `docker`) plutôt qu'en root, comme le recommande la documentation.
- Toujours faire **la sauvegarde avant** et vérifier avec `diff -rq`.
- Ne jamais lancer `update` sans avoir fait `updateself` avant.
- Surveiller les annonces de sécurité sur https://bitwarden.com/help/updating-on-premise/ et dans les notes de version Bitwarden.

## 9. Historique des mises à jour

| Date | Version | Remarque |
|------|---------|----------|
| 21/11/2026 | 2026.11.0 | Correctif nginx CVE-2026-123007 / CVE-2026-123633. Migration manuelle nécessaire (voir point 4). |