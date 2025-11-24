# Installation complète de Graylog 6.1 sur Debian 12

## Présentation
Graylog est un outil open source pour centraliser, stocker et analyser les logs systèmes et réseaux. Il permet de regrouper les journaux sur un seul serveur, facilitant la recherche d'événements et la création d'alertes.

## Prérequis
- Un serveur Debian 12 à jour, adresse IP statique
- 8 Go de RAM recommandés
- 256 Go d’espace disque minimum
- Ports ouverts : 9000/tcp (web), 9200, 9300 (opensearch), 27017 (mongodb local)

## Préparation système
```bash
apt update
apt upgrade
apt install curl lsb-release ca-certificates gnupg2 pwgen nano wget

timedatectl set-timezone Europe/Paris
nano /etc/systemd/timesyncd.conf
# NTP=ntp.univ-rennes2.fr
# FallbackNTP=0.debian.pool.ntp.org 1.debian.pool.ntp.org 2.debian.pool.ntp.org 3.debian.pool.ntp.org
timedatectl set-ntp true
systemctl restart systemd-timesyncd.service
timedatectl timesync-status
```

## Installation de MongoDB 6
```bash
curl -fsSL https://www.mongodb.org/static/pgp/server-6.0.asc | gpg -o /usr/share/keyrings/mongodb-server-6.0.gpg --dearmor

echo "deb [ signed-by=/usr/share/keyrings/mongodb-server-6.0.gpg ] http://repo.mongodb.org/apt/debian bullseye/mongodb-org/6.0 main" | tee /etc/apt/sources.list.d/mongodb-org-6.0.list

apt update
apt-get install -y mongodb-org
```
Si erreur dépendance libssl1.1:
```bash
wget http://archive.ubuntu.com/ubuntu/pool/main/o/openssl/libssl1.1_1.1.1f-1ubuntu2.24_amd64.deb
dpkg -i libssl1.1_1.1.1f-1ubuntu2.24_amd64.deb
apt-get install -y mongodb-org
```
```bash
systemctl daemon-reload
systemctl enable mongod.service
systemctl restart mongod.service
systemctl --type=service --state=active | grep mongod
```

## Installation d'OpenSearch
```bash
curl -o- https://artifacts.opensearch.org/publickeys/opensearch.pgp | gpg --dearmor --batch --yes -o /usr/share/keyrings/opensearch-keyring
echo "deb [signed-by=/usr/share/keyrings/opensearch-keyring] https://artifacts.opensearch.org/releases/bundle/opensearch/2.x/apt stable main" | tee /etc/apt/sources.list.d/opensearch-2.x.list
apt-get update
env OPENSEARCH_INITIAL_ADMIN_PASSWORD=UnSuperMotDePasse! apt-get install opensearch
```
### Configuration d’OpenSearch
```bash
nano /etc/opensearch/opensearch.yml
# cluster.name: graylog
# node.name: ${HOSTNAME}
# path.data: /var/lib/opensearch
# path.logs: /var/log/opensearch
# discovery.type: single-node
# network.host: 127.0.0.1
# action.auto_create_index: false
# plugins.security.disabled: true
nano /etc/opensearch/jvm.options
# -Xms2g
# -Xmx2g
systemctl daemon-reload
systemctl enable opensearch
systemctl restart opensearch
```

## Installation de Graylog
```bash
wget https://packages.graylog2.org/repo/packages/graylog-6.1-repository_latest.deb
dpkg -i graylog-6.1-repository_latest.deb
apt-get update
apt-get install graylog-server
```
Générer un secret et un hash:
```bash
pwgen -N 1 -s 96
# Copier password_secret dans /etc/graylog/server/server.conf
echo -n "Rootsio2017" | shasum -a 256
# Copier le hash dans root_password_sha2
```
Configurer server.conf:
```ini
nano /etc/graylog/server/server.conf
# password_secret = VOTRE_CLE_GENEREE
# root_password_sha2 = VOTRE_HASH
# root_timezone = Europe/Paris
# http_bind_address = 0.0.0.0:9000
# elasticsearch_hosts = http://127.0.0.1:9200
```
```bash
systemctl enable --now graylog-server
```

## Accéder à Graylog
Navigateur : `http://IP:9000`  
Login: admin (mot de passe hashé plus haut)

## Vérification & gestion
- Logs: `/var/log/graylog-server/server.log` `/var/log/opensearch/opensearch.log`
- Status: `systemctl status mongod` `systemctl status opensearch` `systemctl status graylog-server`
- Pour state mémoire: `top`

## Annexes & Astuces
- En cas de souci, vérifier les dépendances et la synchronisation de l’heure avec `timedatectl`.
- Pour ajouter des sources de logs : configurer des Inputs dans Graylog (syslog, GELF…)

## Références
- IT-Connect
- Documentation officielle Graylog, OpenSearch
