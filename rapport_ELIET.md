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
