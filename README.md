# Projet minilab

  

## Installation

  

### 🔧 Etape 1 : Configuration de srv-core

  

#### 1.1 Configuration réseau

  

Fichier : ``etc/network/interfaces`` :

  

    # WAN (ens33)
    auto ens33
    iface ens33inet dhcp
    # LAN (ens34)
    auto ens34
    iface ens34inet static
    address 192.168.15.254
    netmask 255.255.255.0

  

Puis :

    systemctl restart networking

#### 1.2 Activer le routage IP et masquerading NAT

Fichier : `/etc/sysctl.conf`:

    net.ipv4.ip_forward=1

Commande :

    sysctl -p
    
Configurer le NAT avec iptables :

    iptables -t nat -A POSTROUTING -o ens33 -j MASQUERADE

    apt install iptables-persistent

  

#### 1.3 Mise en place du RAID5

Installer :

    apt install mdadm -y

Créer le RAID :

    mdadm --create /dev/md0 --level=5 --raid-devices=4 /dev/sd[b-e]

    mkfs.ext4 /dev/md0

    mkdir /srv/nfs

    mount /dev/md0 /srv/nfs

Fichier : `/etc/fstab`:

    /dev/md0 /srv/nfs ext4 defaults 0 0

Sauvegarder la config RAID :

    mdadm --detail --scan >> /etc/mdadm/mdadm.conf

#### 1.4 NFS Server

    apt install nfs-kernel-server -y

Fichier : `/etc/exports`:

    /srv/nfs 192.168.15.0/24(rw,sync,no_subtree_check)

Activer :

    exportfs -ra
    systemctl restart nfs-server

  

#### 1.5 LDAP + PAM

    apt install slapd ldap-utils libnss-ldap libpam-ldap nslcd

Répondre : 

    - Domaine : linuxisgood.local 
    - Base DN : dc=linuxisgood,dc=local 
    - Admin DN : cn=admin,dc=linuxisgood,dc=local

Configurer la création des utilisateurs et groupes LDAP avec ldif

Fichier : `/etc/nsswitch.conf`:

    passwd: files ldap
    group: files ldap
    shadow: files ldap

Tester :

    getent passwd
