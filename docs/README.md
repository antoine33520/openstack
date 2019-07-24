# Projet OpenStack

Pour ma période de stage de l'année 2018 - 2019 un de mes objectifs est de réaliser et documenter une installation complète d'**OpenStack**.
Cette page constitue donc la documentation à réaliser.

_Toute la documentation a été réalisé avec les droit `root`, toutes les commandes commençant `#` doivent être exécutées soit en utilisant la commande `sudo` (`man sudo` pour plus d'explication) soit depuis l'utilisateur `root`(méthode à éviter pour raison de sécurité)._

## 1) Topologie

Cette installation va être réalisé sur 6 machines virtuelles. Une qui sera le controlleur, deux pour l'execution des instances, une pour le stockage de block et deux autre pour le stockage d'objets.

|   Hostname   | RAM | vCPU | interface 1 |  interface 2   | Disque 1 | Disque 2 | Disque 3 |
|:------------:|:---:|:----:|:-----------:|:--------------:|:--------:|:--------:|:--------:|
| `controller` |  8  |  2   | 10.10.10.10 | 192.168.20.181 |    50    |          |          |
|  `compute`   |  4  |  2   | 10.10.10.20 | 192.168.20.182 |    50    |          |          |
|  `compute2`  |  4  |  2   | 10.10.10.22 | 192.168.20.183 |    50    |          |          |
|  `storage1`  |  2  |  1   | 10.10.10.30 | 192.168.20.184 |    50    |   100    |          |
|  `object1`   |  2  |  1   | 10.10.10.40 | 192.168.20.185 |    50    |    50    |    50    |
|  `object2`   |  2  |  1   | 10.10.10.42 | 192.168.20.186 |    50    |    50    |    50    |

![Photo Topologie](./topo.svg)

Le fichier `/etc/hosts` est complété sur chaque machine.
Pour le bon fonctionnement du réseau un pfsense fait le lien entre le réseau LAN et le réseau réservé à openstack.

## 2) Installation

### 2.1) Environnement

#### 2.1.1) Serveur de temps

Un des pré requis pour avoir un OpenStack fonctionnel est d'avoir des machines ayant la même heure et pour ça on va installer un serveur de temps (ntp) sur `controller` et définir les autres machines comme clients.

Sur toutes les machines il faut installer le paquet `ntp` puis il faut remplacer dans `/etc/ntp.conf` `server 0.ubuntu.pool.ntp.org` par `server controller` sur toutes sauf le `controller` qui lui reste inchangé.

```bash
# sudo apt install ntp
```

#### 2.1.2) Installation des dépôts OpenStack

Sur toutes les machines il faut installer les dépôts OpenStack

```bash
# apt install software-properties-common
# add-apt-repository cloud-archive:stein
# apt update && apt dist-upgrade
# apt install python3-openstackclient
```

#### 2.1.3) Installation de la BDD sur `controller`

Maintenant il faut installer le service de bases de données sur `controller` qui seront nécessaire au bon fonctionnement de l'installation.

On commence par le serveur `mariadb`

```bash
# apt install mariadb-server python-pymysql
```

On édite le fichier de configuration `/etc/mysql/conf.d/mysqld_openstack.cnf` pour y mettre :

```conf
[mysqld]
bind-address = 10.10.10.10
default-storage-engine = innodb
innodb_file_per_table = on
max_connections = 4096
collation-server = utf8_general_ci
character-set-server = utf8
```

Puis on redémarre le service

```bash
# systemctl restart mysql
# systemctl enable mysql
```

#### 2.1.4) Installation de `RabbitMQ` sur `controller`

Ensuite on install RabbitMQ

```bash
# apt-get install rabbitmq-server
```

Et on ajoute l’utilisateur openstack et on positionne ses droits:

```bash
# rabbitmqctl add_user openstack MOT_DE_PASSE
# rabbitmqctl set_permissions openstack ".*" ".*" ".*"
```

#### 2.1.5) Installation de `memcached` sur `controller`

Le service de d'identité utilise memcached afin de mettre en cache les jetons, il faut donc installer ce dernier.

```bash
# apt install memcached python-memcache
```

Pour que les autres machines puissent accéder au service il faut modifier l'adresse IP d'écoute dans le fichier `/etc/memcached.conf`

```conf
-l 10.10.10.10
```

Et on redémarre le service

```bash
# systemctl enable memcached
# systemctl restart memcached

```

#### 2.1.6) Installation de `etcd` sur `controller`

Certains services utilisent etcd pour un stockage clé-valeur sécurisé, il faut donc l'installer.

```bash
# apt install etcd
```

Ensuite on édite le fichier `/etc/default/etcd` avec ces informations :

```code
ETCD_NAME="controller"
ETCD_DATA_DIR="/var/lib/etcd"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster-01"
ETCD_INITIAL_CLUSTER="controller=http://10.10.10.10:2380"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://10.10.10.10:2380"
ETCD_ADVERTISE_CLIENT_URLS="http://10.10.10.10:2379"
ETCD_LISTEN_PEER_URLS="http://0.0.0.0:2380"
ETCD_LISTEN_CLIENT_URLS="http://10.10.10.10:2379"
```

```bash
# systemctl enable etcd
# systemctl start etcd
```

### 2.2) Installation de Keystone

Maintenant on va installer et configurer le service d'identité Keystone.

#### 2.2.1) SQL

```bash
# mysql
```

```mysql
MariaDB [(none)]> CREATE DATABASE keystone;
MariaDB [(none)]> GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' \
IDENTIFIED BY 'MOT_DE_PASSE';
MariaDB [(none)]> GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' \
IDENTIFIED BY 'MOT_DE_PASSE';
MariaDB [(none)]> exit
```

#### 2.2.2) Installation du service

```bash
# apt install keystone
```

Ensuite il faut donner au service la possibilité de se connecter à la base de donnée en éditant le fichier `/etc/keystone/keystone.conf`

```config
[database]
 ...
connection = mysql+pymysql://keystone:MOT_DE_PASSE@controller/keystone

[token]
 ...
provider = fernet
```

Maintenant il faut initialiser le service

```bash
# su -s /bin/sh -c "keystone-manage db_sync" keystone
# keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
# keystone-manage credential_setup --keystone-user keystone --keystone-group keystone
# keystone-manage bootstrap --bootstrap-password ADMIN_PASS \
  --bootstrap-admin-url http://controller:5000/v3/ \
  --bootstrap-internal-url http://controller:5000/v3/ \
  --bootstrap-public-url http://controller:5000/v3/ \
  --bootstrap-region-id RegionOne
```

#### 2.2.3) Configuration d'Apache

Pour le bon fonctionnement du service http il faut configurer le nom du serveur dans le fichier `/etc/apache2/apache2.conf`

```config
ServerName controller
```

#### 2.2.4) Finalisation de l'installation

Redémarrage du service

```bash
# service apache2 restart
```

Exportation des variables du compte administrateur

```bash
$ export OS_USERNAME=admin
$ export OS_PASSWORD=ADMIN_PASS
$ export OS_PROJECT_NAME=admin
$ export OS_USER_DOMAIN_NAME=Default
$ export OS_PROJECT_DOMAIN_NAME=Default
$ export OS_AUTH_URL=http://controller:5000/v3
$ export OS_IDENTITY_API_VERSION=3
```

#### 2.2.5) Création du domaine, projets, utilisateurs et rôles

Pour assurer l'identification le service d'authentification utilise une combinaison de domaines, projets, utilisateurs et rôles.

Le domaine :

```bash
$ openstack domain create --description "Domaine Exemple" exemple
```

Le projet :

```bash
$ openstack project create --domain Default \
  --description "Service Project" service
```

Il faut également créer un projet pour les tâches sans droits administrateurs :

```bash
$ openstack project create --domain Default \
  --description "Demo Project" myproject
```

Un utilisateur :

```bash
$ openstack user create --domain Default \
  --password-prompt myuser
```

Et un rôle :

```bash
$ openstack role create myrole
```

On termine en ajoutant le role à l'utilisateur et au projet :

```bash
$ openstack role add --project myproject --user myuser myrole
```

#### 2.2.6) Création des scripts client OpenStack

Pour simplififer la connexion nous allons créer deux scripts pour une connexion avec et sans les droits administrateur 

`admin-openrc` :

```conf
export OS_PROJECT_DOMAIN_NAME=Default
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=ADMIN_PASS
export OS_AUTH_URL=http://controller:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
```

`demo-openrc` :

```conf
export OS_PROJECT_DOMAIN_NAME=Default
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_NAME=myproject
export OS_USERNAME=myuser
export OS_PASSWORD=26@CHAnce19
export OS_AUTH_URL=http://controller:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
```

Pour utiliser un des scripts pour se connecter il suffit d'exécuter cette commande :

```bash
$ . admin-openrc
```

Et ensuite de demander un token au service :

```bash
$ openstack token issue
```

### 2.3) Installation de Glance sur `controller`

L'étape suivante est l'installation du service d'image (Glance) sur `controller`.
