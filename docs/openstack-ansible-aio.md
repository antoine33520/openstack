# Déploiement "All In One" d'OpenStack avec Ansible

Le déploiement "All In One" permet d'effectuer une installation d'OpenStack sur une seule machine physique ou virtuelle. Ceci est réalisable en créant des conteneur LXC qui hébergeront les defférents services d'OpenStack.

_Toute cette documentation a été réalisée avec l'utilisateur `root` ce qui est vivement déconseillé pour des raisons de sécurité_
_Comme l'exécution de certaines commandes peuvent être longues je recommende d'utiliser un outils comme `tmux` ou `screen` afni de parer aux problèmes liés au possibles déconnexion accidentelles_
## I) Topologie

Pour ce projet je n'utiliserai qu'une seule VM CentOS 7 avec un total de 8 coeurs, 20Go de RAM, un disque de 100Go et un autre de 1To avec l'accélération processeur pour la virtualisation par virtualisation imbriquée.

## II) Préparation de l'hôte

* Il faut commencer par mettre à jour le système :

```bash
[root@aio ~]# yum update -y
```

* Ensuite installer `git` :

```bash
[root@aio ~]# yum install git
```

* Désactivation de `selinux` :
Pour l'instant `openstack-ansible` ne supporte pas que `selinux` soit activé.

```bash
[root@aio ~]# sed -i 's/enforcing/permissive/g' /etc/selinux/config
```

* Désactivation du pare-feu :
Pour éviter que le pare-feu bloque la communication entre l'hôte et les conteneurs je recommande de le désactiver temporairement.

```bash
[root@aio ~]# systemctl stop firewalld
[root@aio ~]# systemctl disable firewalld
Removed symlink /etc/systemd/system/multi-user.target.wants/firewalld.service.
Removed symlink /etc/systemd/system/dbus-org.fedoraproject.FirewallD1.service.
```

* Redémarrage de l'hôte :
Afin de valider les changement il faut redémarrer la machine

```bash
[root@aio ~]# reboot
```

