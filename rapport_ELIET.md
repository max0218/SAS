# Rapport : Sécurité et Administration des Systèmes   
  
## 1. Contruction d'un système d'exploitation Debian Buster minimal  

###  1.1 Installation du système de base
 
Ayant divers problèmes d'installation de Virtualbox sur ma machine, plus précisément j'avais des messages d'erreur me disant de signer les modules kernel de Virtualbox, j'ai donc tenté d'utiliser Qemu-kvm ainsi que virtual-manager, pour finalement réussir à faire fonctionner VirtualBox,  et repasser dessus..
J'ai donc ensuite installé une Debian Buster minimale à partir du mini.iso fourni, en mode rescue mode.
J'ai ensuite choisi mon password root à l'aide de la commande passwd.

### 1.2 Accès à la VM par SSH

Il est bien plus pratique de contrôler la VM par un terminal de la machine hôte, afin de pouvoir me connecter par ssh j'ai donc du modifier la ligne suivante dans le fichier **/etc/ssh/sshd_config**.

    PermitRootLogin yes

et ensuite redémarrer le service ssh

    /etc/init.d/ssh restart

et finalement sur le terminal de la machine hôte

    ssh root@192.168.122.90


## 2. LXC Containers

### 2.1 Installation des packages

    root@VMsas:/home/max# apt-get install lxc lxctl lxc-tests lxc-templates

### 2.2 Configuration réseau des containers

Afin d'avoir le réseau accessible dans nos containers, on met à jour le fichier /etc/lxc/default.conf de cette manière : 

    lxc.net.0.type = veth
    lxc.net.0.link = lxcbr0
    lxc.net.0.flags = up
    lxc.net.0.hwaddr = 00:16:3e:xx:xx:xx
    lxc.apparmor.profile = generated
    lxc.apparmor.allow_nesting = 1

et dans le fichier /etc/default/lxc on active l'utilisation du bridge par les containers : 

    USE_LXC_BRIDGE="true"  # overridden in lxc-net

On peut vérifier que tout ceci fonctionne à l'aide d'un 'ip a' qui donne : 

    3: lxcbr0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default qlen 1000

    link/ether 00:16:3e:00:00:00 brd ff:ff:ff:ff:ff:ff
    
    inet 10.0.3.1/24 scope global lxcbr0
    
       valid_lft forever preferred_lft forever

### 2.3 Création des containers

Pour créer notre premier container on éxécute :

    root@VMsas:~# lxc-create -n c1 -t download

-n, --name=NAME        NAME of the container

-t, --template=TEMPLATE       Template to use to setup container


Et on indique la distribution, release et architecture souhaitée : 

    Distribution: 
    debian

    Release: 
    buster

    Architecture: 
    amd64

### 2.4 Demarrage du premier container

On peut maintenant démarrer notre container c1   en mode daemon: 

    root@VMsas:~# lxc-start -n c1

-n, --name=NAME        NAME of the container

Ceci m'a donné comme erreur : 

    lxc-start: c1: lsm/apparmor.c: apparmor_prepare: 974 Cannot use generated profile: apparmor_parser not available

Et après de longues recherches c'était en fait juste une question de PATH.

J'ai pu ensuite vérifier que le container avait bien démarré : 

    root@VMsas:~# lxc-info -n c1
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
     Total bytes:   3.54 KiB

Et vérifier sa configuration : 

    root@VMsas:~# cat /var/lib/lxc/c1/config 
    
    # Template used to create this container: /usr/share/lxc/templates/lxc-download
    # Parameters passed to the template:
    # Template script checksum (SHA-1): 273c51343604eb85f7e294c8da0a5eb769d648f3
    # For additional config options, please look at lxc.container.conf(5)

    # Uncomment the following line to support nesting containers:
    #lxc.include = /usr/share/lxc/config/nesting.conf
    # (Be aware this has security implications)


    # Distribution configuration
    lxc.include = /usr/share/lxc/config/common.conf
    lxc.arch = linux64

    # Container specific configuration
    lxc.apparmor.profile = generated
    lxc.apparmor.allow_nesting = 1
    lxc.rootfs.path = dir:/var/lib/lxc/c1/rootfs
    lxc.uts.name = c1

    # Network configuration
    lxc.net.0.type = veth
    lxc.net.0.link = lxcbr0
    lxc.net.0.flags = up
    lxc.net.0.hwaddr = 00:16:3e:5b:8c:4a

### 2.5 Clonage du container c1 pour créer deux autres containers


    root@VMsas:~# lxc-copy -n c1 -N c2
    root@VMsas:~# lxc-ls
    c1   
  Le clonage ne fonctionne pas si c1 est dans l'état RUNNING :
  
    root@VMsas:~# lxc-stop c1
    root@VMsas:~# lxc-copy -n c1 -N c2
    root@VMsas:~# lxc-copy -n c1 -N c3
    root@VMsas:~# lxc-ls
    c1   c2   c3   

## 3. Mise en place du réseau 

Les containers sont reliés entre eux par le biais d'un bridge nommé lxcbr0.
On veut créer un bridge nommé lxc-bridge-nat pour relier nos containers au réseau de la machine hôte.

### 3.1 Configuration de la machine hôte

#### Création du bridge 

Dans le fichier /etc/network/interfaces on insère : 

    auto lxc-bridge-nat
    iface lxc-bridge-nat inet static
        bridge_ports none
        bridge_fd 0
        bridge_maxwait 0
        address 192.168.100.1
        netmask 255.255.255.0
        up iptables -t nat -A POSTROUTING -o enp0s3 -j MASQUERADE

Et dans le fichier /etc/sysctl.conf : 
   
    #Uncomment the next line to enable packet forwarding for IPv4
    net.ipv4.ip_forward=1
   
Et pour enable le forwarding on effectue cette commande : 
     
     echo 1 > /proc/sys/net/ipv4/ip_forward

Et enfin on restart le service : 
    
    /etc/init.d/networking restart  
    
Et on vérifie que le bridge a bien été créé : 
    
    root@VMsas:/home/max# ip a
    ...
    4: lxc-bridge-nat: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN group default qlen 1000
        link/ether c2:3e:7f:ed:ef:fd brd ff:ff:ff:ff:ff:ff
        inet 192.168.100.1/24 brd 192.168.100.255 scope global lxc-bridge-nat
           valid_lft forever preferred_lft forever
        inet6 fe80::c03e:7fff:feed:effd/64 scope link 
           valid_lft forever preferred_lft forever
          
 ### 3.2 Configuration réseau des containers
 
 #### Le container C1
 
 On modifie la config de notre container c1 dans le fichier /var/lib/lxc/c1/config afin qu'il utilise le bridge nouvellement créé et la passerelle à la bonne addresse : 

     #Network configuration
    lxc.net.0.type = veth
    lxc.net.0.link = lxc-bridge-nat
    lxc.net.0.ipv4.address = 192.168.100.10/24
    lxc.net.0.ipv4.gateway = 192.168.100.1
    lxc.net.0.flags = up
    lxc.net.0.hwaddr = 00:16:3e:5b:8c:4a

On peut maintenant démarrer notre container et vérifier son IP  :

    root@VMsas:/home/max# lxc-start c1
    root@VMsas:/home/max# lxc-info c1
    Name:           c1
    State:          RUNNING
    PID:            815
    IP:             192.168.100.10
    ...
    
On se connecte ensuite à la console de c1 afin de modifier son fichier de configuration du DNS /etc/resolv.conf pour qu'il soit identique à celui de la VM hôte : 

    root@VMsas:~# cat /etc/resolv.conf
    nameserver 192.168.1.1
    root@VMsas:~# lxc-attach c1
    root@c1:/# echo 'nameserver 192.168.1.1' > /etc/resolv.conf

On peut vérifier que ça fonctionne bien en pingant le serveur DNS de Google : 

    root@c1:/# apt-get install iputils-ping
    root@c1:~# ping www.google.fr 
    PING www.google.fr (216.58.215.35) 56(84) bytes of data.
    64 bytes from par21s17-in-f3.1e100.net (216.58.215.35): icmp_seq=1 ttl=54 time=5.80 ms
    64 bytes from par21s17-in-f3.1e100.net (216.58.215.35): icmp_seq=2 ttl=54 time=7.27 ms
    ^C
    --- www.google.fr ping statistics ---
    2 packets transmitted, 2 received, 0% packet loss, time 3ms
    rtt min/avg/max/mdev = 5.796/6.533/7.270/0.737 ms


##  4. Installation du serveur DHCP

### 4.1 Configurer le serveur DHCP sur le container C1

On commence par installer le package isc-server-dhcp :

    root@c1:/# apt-get install isc-dhcp-server

Ensuite on modifie le fichier de configuration */etc/dhcp/dhcpd.conf* :

    # option definitions common to all supported networks...
    option domain-name "sas";
    option domain-name-servers 192.168.1.1;

    # Subnet configuration
    subnet 192.168.100.0 netmask 255.255.255.0 {
        range 192.168.100.128 192.168.100.192;
        option subnet-mask 255.255.255.0;
        option broadcast-address 192.168.100.255;
        option routers 192.168.100.10;
    }
    
Et on précise dans */etc/default/isc-dhcp-server*

    INTERFACESv4="eth0"
    
Et dans */etc/network/interfaces*

    # The loopback network interface
    auto lo
    iface lo inet loopback

    auto eth0
    iface eth0 inet static
        address 192.168.100.10
        netmask 255.255.255.0
        broadcast 192.168.100.255
        gateway 192.168.100.1

    source /etc/network/interfaces.d/*.cfg
    
 On peut maintenant vérifier que le service DHCP fonctionne bien : 
 
    root@c1:/# service isc-dhcp-server restart 
    root@c1:/# service isc-dhcp-server status
    ● isc-dhcp-server.service - LSB: DHCP server
       Loaded: loaded (/etc/init.d/isc-dhcp-server; generated)
       Active: active (running) since Sat 2020-05-09 08:09:52 UTC; 7s ago
         Docs: man:systemd-sysv-generator(8)
      Process: 702 ExecStart=/etc/init.d/isc-dhcp-server start (code=exited, status=0/SUCCESS)
        Tasks: 1 (limit: 4698)
       Memory: 4.5M
       CGroup: /system.slice/isc-dhcp-server.service
               └─714 /usr/sbin/dhcpd -4 -q -cf /etc/dhcp/dhcpd.conf eth0

    May 09 08:09:50 c1 systemd[1]: Starting LSB: DHCP server...
    May 09 08:09:50 c1 isc-dhcp-server[702]: Launching IPv4 server only.
    May 09 08:09:50 c1 dhcpd[714]: Wrote 1 leases to leases file.
    May 09 08:09:50 c1 dhcpd[714]: Server starting service.
    May 09 08:09:52 c1 isc-dhcp-server[702]: Starting ISC DHCPv4 server: dhcpd.
    May 09 08:09:52 c1 systemd[1]: Started LSB: DHCP server.


















