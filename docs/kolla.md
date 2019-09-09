# Déploiement d'OpenStack avec Kolla-ansible

Pour suivre cette documentation il est nécessaire d'avoir quelques connaissances basiques d'[Ansible](https://docs.ansible.com/) et de [Docker](https://docs.docker.com/).\
Kolla va effectuer le déploiement d'OpenStack sur des conteneurs Docker.\
Kolla peut être exécuter directement sur le système ou dans un environnement virtuel python.\
Il y a aussi une version multinode et une autre all-in-one.

## Configuration matériel

La documentation offcielle est rédigée pour CentOS, RHEL et Ubuntu.\
Configuration minimale :

- 2 cartes réseaux
- 8GB de RAM
- 40GB d'espace disque

Cette documentation est réalisée sur une VM Ubuntu 18.04 LTS avec 2 cartes réseaux, 64GB de RAM et 300GB de dique.\
L'installation sera une version all-in-one directement exécuté sur le système.

## Installation des dépendances

- Mise à jour du système :

```bash
sudo apt update
```

- Installation de `python` et de ses dépendances :

```bash
sudo apt install python-dev libffi-dev gcc libssl-dev python-selinux python-setuptools
```

- Installation de `pip` :

```bash
sudo apt install python-pip
```

- Mise à jour de `pip` :

```bash
sudo pip install -U pip
```

- Installation d'`Ansible` :

```bash
sudo apt install ansible
```

## Installation de Kolla-ansible

- Installation de `kolla-ansible` avec `pip` :

```bash
sudo pip install kolla-ansible
```

- Création du dossier `/etc/kolla` :

```bash
sudo mkdir -p /etc/kolla
sudo chown $USER:$USER /etc/kolla
```

- Copie de `globals.yml` et `passwords.yml` dans le dossier `/etc/kolla` :

```bash
cp -r /usr/local/share/kolla-ansible/etc_examples/kolla/* /etc/kolla
```

- Copie des fichiers d'inventaire `all-in-one` et `multinode` dans le dossier courant :

```bash
cp /usr/local/share/kolla-ansible/ansible/inventory/* .
```

## Configuration d'`Ansible`

Il faut modifier quelques données dans le fichier `/etc/ansible/ansible.cfg` :

```bash
[defaults]
host_key_checking=False
pipelining=True
forks=100
```

## Préparation de la configuration initiale

### Génération des mots de passes

Kolla a une commande pour générer les mots de passes necéssaires au déploiement :

```bash
kolla-genpwd
```

### Kolla globals.yml

Le fichier `/etc/kolla/globals.yml` permet de configurer le déploiement :

- Distribution pour conteneurs :
Il faut choisir quelle distribution sera utilisée comme base pour `OpenStack`.

```bash
kolla_base_distro: "centos"
```

- Source de l'installation :
L'installation peut être faite soit via les dépots apt ou yum en fonction de la distribution ou via les dépots source.

```bash
kolla_install_type: "binary"
```

- Version d'`OpenStack` :
Il faut choisir la version d'`OpenStack` a installé. `master` permet de choisir la dernière version publié.

```bash
openstack_release: "master"
```

- Réseau :
  - IP vip pour l'administration :
    Cette IP ne doit pas déjà être utilisée et se trouvé dans le réseau interne dédié à l'administration.

    ```bash
    kolla_internal_vip_address: "192.168.77.250"
    ```

    - Interface pour le réseau interne :

    ```bash
    network_interface: "ens32"
    ```

    - Interface pour le réseau externe :

    ```bash
    neutron_external_interface: "ens33"
    ```

- Activation de services supplémentaires :
Pour activer des services supplémentaires il suffit de modifier la valeur associé dans ce fichier mais il faut prendre en compte les dépendances.

```bash
enable_cinder: "yes"
```

## Déploiement

- Préparation du serveur :

```bash
kolla-ansible -i ./all-in-one bootstrap-servers
```

- Vérification avant le déploiement :

```bash
kolla-ansible -i ./all-in-one prechecks
```

- Déploiement :

```bash
kolla-ansible -i ./all-in-one deploy
```

## Utilisation d'`OpenStack`

- Installation du client CLI :

```bash
pip install python-openstackclient
```

- Génération des identifiants administrateur :

```bash
kolla-ansible post-deploy
. /etc/kolla/admin-openrc.sh
```

- Création des exemples :

```bash
/usr/local/share/kolla-ansible/init-runonce
```
