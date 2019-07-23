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

## 2) Installation

### 2.1) Serveur de temps

Un des pré requis pour avoir un OpenStack fonctionnel est d'avoir des machines ayant la même heure et pour ça on va installer un serveur de temps (ntp) sur `controller` et définir les autres machines comme clients.

Sur toutes les machines il faut installer le paquet `ntp` puis il faut remplacer dans `/etc/ntp.conf` `server 0.ubuntu.pool.ntp.org` par `server controller` sur toutes sauf le `controller` qui lui reste inchangé.

```bash
# sudo apt install ntp
```

### 2.2) Installation des dépôts OpenStack

Sur toutes les machines il faut installer les dépôts OpenStack

```bash
# apt install software-properties-common
# add-apt-repository cloud-archive:stein
# apt update && apt dist-upgrade
# apt install python-openstackclient
```

### 2.3) Installation des BDD sur `controller`

Maintenant il faut installer les bases de données sur `controller` qui seront nécessaire au bon fonctionnement de l'installation.

On commence par le serveur `mariadb`

```bash
# apt install mariadb-server python-pymysql
```

On édite le fichier de configuration `/etc/mysql/conf.d/mysqld_openstack.cnf` pour y mettre :

```conf
[mysqld]
## Set to Management IP
bind-address = 10.10.10.10
default-storage-engine = innodb
innodb_file_per_table
collation-server = utf8_general_ci
init-connect = 'SET NAMES utf8'
character-set-server = utf8
```

Puis on redémarre le service

```bash
# systemctl restart mysql
```

Ensuite on passe à installation de `mongodb`

```bash
# apt install mongodb-server mongodb-clients python-pymongo
```

Et on édit son fichier de configuration `/etc/mongodb.conf`

```conf
dbpath=/var/lib/mongodb
logpath=/var/log/mongodb/mongodb.log
logappend=true
bind_ip = 10.10.10.10
journal=true
smallfiles = true
```

On redémarre le service

```bash
# systemctl restart mongodb
```

Ensuite on finit par l'installation de RabbitMQ

```bash
# apt-get install rabbitmq-server
```

Et on ajoute l’utilisateur openstack et on positionne ses droits:

```bash
# rabbitmqctl add_user openstack MOT_DE_PASSE
# rabbitmqctl set_permissions openstack ".*" ".*" ".*"
```

### 2.4) Installation de `memcached`
