# Rapport : Sécurité et Administration des Systèmes   
  
## 1. Contruction d'un système d'exploitation Debian Buster minimal  

###  1.1 Installation du système de base
 
Ayant divers problèmes d'installation de Virtualbox sur ma machine perosnnelle, j'ai donc tenté d'utiliser Qemu-kvm,
ainsi que virtual-manager, pour finalement réussir à faire fonctionner VirtualBox et repasser dessus..
J'ai donc ensuite installé une Debian Buster minimale à partir du mini.iso fourni, en mode rescue mode.
J'ai ensuite choisi mon password root à l'aide de la commande passwd.

### 1.2 Accès à la VM par SSH

Il est bien plus pratique de contrôler la VM par un terminal de la machine hôte, afin de pouvoir me connecter par ssh j'ai donc du modifier la ligne suivante dans le fichier **/etc/ssh/sshd_config** 

*PermitRootLogin yes*

et ensuite redémarrer le service ssh

*/etc/init.d/ssh restart*

et finalement sur le terminal de la machine hôte

*ssh root@192.168.122.90*


## 2. LXC Containers

### 2.1 Installation des packages

*root@VMsas:/home/max# apt-get install lxc lxctl lxc-tests lxc-templates*

### 2.2 Configuration réseau des containers

Afin d'avoir le réseau accessible dans nos containers, on met à jour le fichier /etc/lxc/default.conf de cette manière : 

*lxc.net.0.type = veth
lxc.net.0.link = lxcbr0
lxc.net.0.flags = up
lxc.net.0.hwaddr = 00:16:3e:xx:xx:xx
lxc.apparmor.profile = generated
lxc.apparmor.allow_nesting = 1*

et dans le fichier /etc/default/lxc on active l'utilisation du bridge par les containers : 

*USE_LXC_BRIDGE="true"  # overridden in lxc-net*

On peut vérifier que tout ceci fonctionne à l'aide d'un 'ip a' qui donne : 

*3: lxcbr0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default qlen 1000
    link/ether 00:16:3e:00:00:00 brd ff:ff:ff:ff:ff:ff
    inet 10.0.3.1/24 scope global lxcbr0
       valid_lft forever preferred_lft forever*

### 2.3 Création des containers

Pour créer notre premier container on éxécute :
*root@VMsas:~# lxc-create -n c1 -t download
*
-n, --name=NAME        NAME of the container
-t, --template=TEMPLATE       Template to use to setup container


Et on indique la distribution, release et architecture souhaitée : 

*Distribution: 
debian
Release: 
buster
Architecture: 
amd64*

On peut maintenant démarrer notre container c1   en mode daemon: 

*root@VMsas:~# lxc-start -n c1*
-n, --name=NAME        NAME of the container

Ceci m'a donné comme erreur : 
*lxc-start: c1: lsm/apparmor.c: apparmor_prepare: 974 Cannot use generated profile: apparmor_parser not available*

Et après de longues recherches c'était en fait juste une question de PATH.

J'ai pu ensuite vérifier que le container avait bien démarré : 

*root@VMsas:~# lxc-info -n c1
Name:           c1
State:          RUNNING
PID:            1097
IP:             10.0.3.168
CPU use:        0.27 seconds
BlkIO use:      88.00 KiB
Memory use:     17.84 MiB
KMem use:       3.71 MiB
Link:           vethKDJXWC
 TX bytes:      1.55 KiB
 RX bytes:      1.98 KiB
 Total bytes:   3.54 KiB*
 
 
