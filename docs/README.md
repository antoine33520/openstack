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
# rabbitmqctl add_user openstack RABBIT_PASS
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
export OS_PASSWORD=USER_PASS
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

#### 2.3.1) Pré requis

##### 2.3.1.1) Préparation de la base de données

Il faut commencer par créer la base de données :

```bash
# mysql
```

```mysql
MariaDB [(none)]> CREATE DATABASE glance;
MariaDB [(none)]> GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost' \
  IDENTIFIED BY 'GLANCE_DBPASS';
MariaDB [(none)]> GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' \
  IDENTIFIED BY 'GLANCE_DBPASS';
```

##### 2.3.1.2) Création du compte pour le service

Ensuite on se connect avec les droits administrateurs à l'aide du script.

```bash
$ . admin-openrc
```

Création de l'utilisateur :

```bash
$ openstack user create --domain default --password-prompt glance
```

Ajout du rôle administrateur à l'utilisateur glance et projet service :

```bash
$ openstack role add --project service --user glance admin
```

Création du service :

```bash
$ openstack service create --name glance \
  --description "OpenStack Image" image
```

##### 2.3.1.3) création des points de terminaison de l'API

```bash
$ openstack endpoint create --region RegionOne \
  image public http://controller:9292
$ openstack endpoint create --region RegionOne \
  image internal http://controller:9292
$ openstack endpoint create --region RegionOne \
  image admin http://controller:9292
```

#### 2.3.2) Installation et configuration du composant

Installation du paquet :

```bash
# apt install glance
```

Ensuite il faut éditer le fichier `/etc/glance/glance-api.conf` :

* Dans la section [database], on configure la connexion à la BDD :

```conf
[database]
# ...
connection = mysql+pymysql://glance:GLANCE_DBPASS@controller/glance
```

* Et dans les sections [keystone_authtoken] et [paste_deploy], on configure l'accès au service d'identité :

```conf
[keystone_authtoken]
# ...
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = glance
password = GLANCE_PASS

[paste_deploy]
# ...
flavor = keystone
```

* Enfin dans la section [glance_store], on configure le stockage des fichiers d'image :

```conf
[glance_store]
# ...
stores = file,http
default_store = file
filesystem_store_datadir = /var/lib/glance/images/
```

Il faut également modifier le fichier `/etc/glance/glance-registry.conf` :

* Dans la section [database], on configure l'accès à la BDD :

```conf
[database]
# ...
connection = mysql+pymysql://glance:GLANCE_DBPASS@controller/glance
```

* Et dans les sections [keystone_authtoken] et [paste_deploy], on configure l'accès au service d'identité :

```conf
[keystone_authtoken]
# ...
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = glance
password = GLANCE_PASS

[paste_deploy]
# ...
flavor = keystone
```

Après il faut initialiser la BDD :

```bash
# su -s /bin/sh -c "glance-manage db_sync" glance
```

#### 2.3.3) Finalisation de l'installation

```bash
# service glance-registry restart
# service glance-api restart
```

### 2.4 Installation de Placement sur `controller`

#### 2.4.1) Pré requis

##### 2.4.1.1) Préparation de la base de données

Il faut commencer par créer la base de données :

```bash
# mysql
```

```mysql
MariaDB [(none)]> CREATE DATABASE placement;
MariaDB [(none)]> GRANT ALL PRIVILEGES ON glance.* TO 'placement'@'localhost' \
  IDENTIFIED BY 'PLACEMENT_DBPASS';
MariaDB [(none)]> GRANT ALL PRIVILEGES ON glance.* TO 'placement'@'%' \
  IDENTIFIED BY 'PLACEMENT_DBPASS';
```

##### 2.4.1.2) Création du compte pour le service

Ensuite on se connect avec les droits administrateurs à l'aide du script.

```bash
$ . admin-openrc
```

Création de l'utilisateur :

```bash
$ openstack user create --domain default --password-prompt placement
```

Ajout du rôle administrateur à l'utilisateur glance et projet service :

```bash
$ openstack role add --project service --user placement admin
```

Création du service :

```bash
$ openstack service create --name placement \
  --description "Placement API" placement
```

##### 2.4.1.3) création des points de terminaison de l'API

```bash
$ openstack endpoint create --region RegionOne \
  placement public http://controller:8778
$ openstack endpoint create --region RegionOne \
  placement internal http://controller:8778
$ openstack endpoint create --region RegionOne \
  placement admin http://controller:8778
```

#### 2.4.2) Installation et configuration du composant

Installation du paquet :

```bash
# apt install placement-api
```

Ensuite on édit le fichier `/etc/placement/placement.conf` :

* Dans la section [placement_database], on configure la connexion à la BDD :

```conf
[placement_database]
# ...
connection = mysql+pymysql://placement:PLACEMENT_DBPASS@controller/placement
```

* Et dans les sections [api] et [keystone_authtoken], la connexion au service d'identité :

```conf
[api]
# ...
auth_strategy = keystone

[keystone_authtoken]
# ...
auth_url = http://controller:5000/v3
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = placement
password = PLACEMENT_PASS
```

Il faut après initialiser la base de données :

```bash
# su -s /bin/sh -c "placement-manage db sync" placement
```

#### 2.4.3) Finalisation de l'installation

```bash
# service apache2 restart
```

### 2.5) Installation de Nova

#### 2.5.1) Partie sur `controller`

##### 2.5.1.1) Pré requis

###### 2.5.1.1.1) Préparation de la base de données

Il faut commencer par créer la base de données :

```bash
# mysql
```

```mysql
MariaDB [(none)]> CREATE DATABASE nova_api;
MariaDB [(none)]> CREATE DATABASE nova;
MariaDB [(none)]> CREATE DATABASE nova_cell0;

MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'localhost' \
  IDENTIFIED BY 'NOVA_DBPASS';
MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'%' \
  IDENTIFIED BY 'NOVA_DBPASS';

MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'localhost' \
  IDENTIFIED BY 'NOVA_DBPASS';
MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' \
  IDENTIFIED BY 'NOVA_DBPASS';

MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'localhost' \
  IDENTIFIED BY 'NOVA_DBPASS';
MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'%' \
  IDENTIFIED BY 'NOVA_DBPASS';
```

##### 2.5.1.1.2) Création du compte pour le service

Ensuite on se connect avec les droits administrateurs à l'aide du script.

```bash
$ . admin-openrc
```

Création de l'utilisateur :

```bash
$ openstack user create --domain default --password-prompt nova
```

Ajout du rôle administrateur à l'utilisateur glance et projet service :

```bash
$ openstack role add --project service --user nova admin
```

Création du service :

```bash
$ openstack service create --name nova \
  --description "OpenStack Compute" compute
```

##### 2.5.1.1.3) création des points de terminaison de l'API

```bash
$ openstack endpoint create --region RegionOne \
  compute public http://controller:8774/v2.1

$ openstack endpoint create --region RegionOne \
  compute internal http://controller:8774/v2.1

$ openstack endpoint create --region RegionOne \
  compute admin http://controller:8774/v2.1
```

#### 2.5.1.2) Installation et configuration du composant

Installation des paquets :

```bash
# apt install nova-api nova-conductor \
  nova-novncproxy nova-scheduler
```

Il faut éditer le fichier `/etc/nova/nova.conf` :

* Dans les sections [api_database] et [database], on configure l'accès à la base de données :

```conf
[api_database]
# ...
connection = mysql+pymysql://nova:NOVA_DBPASS@controller/nova_api

[database]
# ...
connection = mysql+pymysql://nova:NOVA_DBPASS@controller/nova
```

* Dans la section [DEFAULT], on configure la connexion à RabbitMQ

```conf
[DEFAULT]
# ...
transport_url = rabbit://openstack:RABBIT_PASS@controller
```

* Dans les sections [api] et [keystone_authtoken], on configure l'accès au service d'identité :

```conf
[api]
# ...
auth_strategy = keystone

[keystone_authtoken]
# ...
auth_url = http://controller:5000/v3
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = nova
password = NOVA_PASS
```

* A nouveau dans la section [DEFAULT], on indique l'IP de `controller` :

```conf
[DEFAULT]
# ...
my_ip = 10.10.10.10
```

* Toujous dans la section [DEFAULT], on active le suppot du réseau :

```conf
[DEFAULT]
# ...
use_neutron = true
firewall_driver = nova.virt.firewall.NoopFirewallDriver
```

* Dans la section [neutron], on va configurer d'avance la connexion au service neutron qui n'est pas encore installé

```conf
[neutron]
# ...
url = http://controller:9696
auth_url = http://controller:5000
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = neutron
password = NEUTRON_PASS
service_metadata_proxy = true
metadata_proxy_shared_secret = METADATA_SECRET
```

* Dans la section [vnc], on configure le proxy du vnc :

```conf
[vnc]
enabled = true
# ...
server_listen = $my_ip
server_proxyclient_address = $my_ip
```

* Dans la section [glance], on configure la connexion à l'api de glance :

```conf
[glance]
# ...
api_servers = http://controller:9292
```

* Dans la section [oslo_concurrency], on configure le lock_path :

```conf
[oslo_concurrency]
# ...
lock_path = /var/lib/nova/tmp
```

* De nouveau dans la section [DEFAULT]
Comme indiqué dans la documentation officielle il faut retiré le `log_dir` du fichier de configuration.

* Dans la section [placement], on configure l'accès au service Placement

```conf
[placement]
# ...
region_name = RegionOne
project_domain_name = Default
project_name = service
auth_type = password
user_domain_name = Default
auth_url = http://controller:5000/v3
username = placement
password = PLACEMENT_PASS
```

Ensuite on initialise les bases de données :

```bash
# su -s /bin/sh -c "nova-manage api_db sync" nova
# su -s /bin/sh -c "nova-manage cell_v2 map_cell0" nova
# su -s /bin/sh -c "nova-manage cell_v2 create_cell --name=cell1 --verbose" nova
# su -s /bin/sh -c "nova-manage db sync" nova
# su -s /bin/sh -c "nova-manage cell_v2 list_cells" nova
```

##### 2.5.1.3) Finalisation de l'installation

Pour terminer l'installation il faut redémarrer les services :

```bash
# service nova-api restart
# service nova-scheduler restart
# service nova-conductor restart
# service nova-novncproxy restart
```

#### 2.5.2) Partie sur `compute` et `compute2`

#### 2.5.2.1) Installation du composant

Cette partie doit être réalisée sur toutes les machines étant destinées au rôle compute.

Installation du paquet :

```bash
# apt install nova-compute
```

Il faut éditer le fichier `/etc/nova/nova.conf` :

* Dans la section [DEFAULT], on configure RabbitMQ

```conf
[DEFAULT]
# ...
transport_url = rabbit://openstack:RABBIT_PASS@controller
```

* Dans les sections [api] et [keystone_authtoken], on configure l'accès au service d'identité

```conf
[api]
# ...
auth_strategy = keystone

[keystone_authtoken]
# ...
auth_url = http://controller:5000/v3
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = nova
password = NOVA_PASS
```

* A nouveau dans la section [DEFAULT], on configure l'adresse ip :

```conf
[DEFAULT]
# ...
my_ip = MANAGEMENT_INTERFACE_IP_ADDRESS
```

* Toujours dans la section [DEFAULT], on configure le support du réseau :

```conf
[DEFAULT]
# ...
use_neutron = true
firewall_driver = nova.virt.firewall.NoopFirewallDriver
```

* Dans la section [neutron], on configure la connexion au service neutron qu'on mettra en place par la suite :

```conf
[neutron]
# ...
url = http://controller:9696
auth_url = http://controller:5000
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = neutron
password = NEUTRON_PASS
```

* Dans la section [vnc], on configure l'accès vnc à distance :

```conf
[vnc]
# ...
enabled = true
server_listen = 0.0.0.0
server_proxyclient_address = $my_ip
novncproxy_base_url = http://controller:6080/vnc_auto.html
```

* Dans la section [glance], on configure l'accès à l'api glance :

```conf
[glance]
# ...
api_servers = http://controller:9292
```

* Dans la section [oslo_concurrency], on configure le lock_path :

```conf
[oslo_concurrency]
# ...
lock_path = /var/lib/nova/tmp
```

* Dans la section [placement], on configure l'accès à l'api de Placement :

```conf
[placement]
# ...
region_name = RegionOne
project_domain_name = Default
project_name = service
auth_type = password
user_domain_name = Default
auth_url = http://controller:5000/v3
username = placement
password = PLACEMENT_PASS
```

#### 2.5.2.2) Finalisation de l'installation

Il faut vérifier si la/les machine(s) peuvent utiliser l'accélération processeur pour la virtualisation :

```bash
# egrep -c '(vmx|svm)' /proc/cpuinfo
```

**ATTENTION** si le résultat est différent de `0` l'étape suivante n'est pas nécessaire et pourrait ralentir l'exécution des instances.
Si le résultat de la commande précédente est `0` il faut modifier le fichier `/etc/nova/nova-compute.conf` :

* Dans la section [libvirt]

```conf
[libvirt]
# ...
virt_type = qemu
```

Il faut ensuite redémarrer le service :

```bash
service nova-compute restart
```

#### 2.5.3) A nouveau `controller`

#### 2.5.3.1) Ajout des `compute`

Il faut maintenant ajouter les machines dans la base de données :

```bash
$ . admin-openrc

# su -s /bin/sh -c "nova-manage cell_v2 discover_hosts --verbose" nova
Found 2 cell mappings.
Skipping cell0 since it does not contain hosts.
Getting computes from cell 'cell1': f741e0b3-b740-45a4-823f-a9fbbb4e6e81
Checking host mapping for compute host 'compute': 260ba75d-2c7e-40c1-b2fa-01cecb501d3a
Creating host mapping for compute host 'compute': 260ba75d-2c7e-40c1-b2fa-01cecb501d3a
Checking host mapping for compute host 'compute2': 1918c90d-93e0-416b-a724-2393f4b2d348
Creating host mapping for compute host 'compute2': 1918c90d-93e0-416b-a724-2393f4b2d348
Found 2 unmapped computes in cell: f741e0b3-b740-45a4-823f-a9fbbb4e6e81

```

### 2.6) Installation de Neutron

Neutron est le service qui s'occupe de la gestion de la partie réseau d'openstack.
2 options sont disponibles pour la configuration réseau d'OpenStack, ici j'utiliserai la première, pour plus d'information rapportez-vous à la [documentattion officielle](https://docs.openstack.org/neutron/stein/install/overview.html).

#### 2.6.1) Partie sur `controller`

