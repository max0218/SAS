# Rapport : Sécurité et Administration des Systèmes   
  
## 1. Contruction d'un système d'exploitation Debian Buster minimal  

###  1.1 Installation du système de base
 
Ayant divers problèmes d'installation de Virtualbox sur ma machine perosnnelle, j'ai décidé de plutôt utiliser Qemu-kvm,
ainsi que virtual-manager afin de me faciliter la tâche.
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

*...
3: lxcbr0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default qlen 1000
    link/ether 00:16:3e:00:00:00 brd ff:ff:ff:ff:ff:ff
    inet 10.0.3.1/24 scope global lxcbr0
       valid_lft forever preferred_lft forever
*
