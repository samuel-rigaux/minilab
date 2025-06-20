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

    
### Etape 2 : Configuration des serveurs srv-dhcp1 et srv-dhcp2

#### 2.1 IP statique
Fichier : `/etc/network/interfaces`:

    # srv-dhcp1
    iface ens33 inet static
    address 192.168.15.253
    netmask 255.255.255.0
    gateway 192.168.15.254

Idem pour srv-dhcp2 avec IP 192.168.15.252

    # srv-dhcp2
        iface ens33 inet static
        address 192.168.15.252
        netmask 255.255.255.0
        gateway 192.168.15.254


#### 2.2 DHCP + Failover

    apt install isc-dhcp-server

Fichier : `/etc/dhcp/dhcpd.conf` (srv-dhcp1) :

    default-lease-time 600;
    max-lease-time 7200;
    option domain-name "linuxisgood.local";
    option domain-name-servers 192.168.15.250;
    
    failover peer "dhcp" {
    primary;
    address 192.168.15.253;
    port 647;
    peer address 192.168.15.252;
    peer port 647;
    max-response-delay 60;
    split 128;
    mclt 3600;
    }
    
    subnet 192.168.15.0 netmask 255.255.255.0 {
    range 192.168.15.100 192.168.15.150;
    option routers 192.168.15.254;
    option broadcast-address 192.168.15.255;
    option domain-name-servers 192.168.15.250;
    }

Adapter l’autre serveur en secondary avec l’adresse inversée :

    default-lease-time 600;
    max-lease-time 7200;
    
    option domain-name "linuxisgood.local";
    option domain-name-servers 192.168.15.250;
    
    failover peer "dhcp" {
      secondary;
      address 192.168.15.252;
      port 647;
      peer address 192.168.15.253;
      peer port 647;
      max-response-delay 60;
      load balance max seconds 3;
    }
    
    subnet 192.168.15.0 netmask 255.255.255.0 {
      range 192.168.15.100 192.168.15.150;
      option routers 192.168.15.254;
      option broadcast-address 192.168.15.255;
      option domain-name-servers 192.168.15.250;
    
      # Lier cette plage à la config de failover
      peer "dhcp";
    }


#### 2.3 DNS (bind9)

    apt install bind9

Fichier : `/etc/bind/named.conf.local` (srv-dhcp1) :

    zone "linuxisgood.local" {
    type master;
    file "/etc/bind/db.linuxisgood";
    allow-transfer { 192.168.15.252; };
    };

Fichier : `/etc/bind/db.linuxisgood` (Copier le fichier db.local comme base)

    $TTL 604800
    @ IN SOA linuxisgood.local. root.linuxisgood.local. (
    
    2 ; Serial
    604800 ; Refresh
    86400 ; Retry
    2419200 ; Expire
    604800 ) ; Negative Cache TTL
    
    ;
    @ IN NS srv-dhcp1.linuxisgood.local.
    srv-core IN A 192.168.15.254
    srv-dhcp1 IN A 192.168.15.253
    srv-dhcp2 IN A 192.168.15.252


#### 2.4 LDAP slave (syncrepl)
##### Configurer le serveur master : 

Fichier : `/etc/ldap/slapd.d/cn=config/olcDatabase={1} mdb.ldif` 
Ajoutez une entrée `olcSyncRepl` avec la config du slave.

Sur le slave, configurer le `olcServerID` et `olcSyncRepl` aussi, en inversant master/slave.

### Etape 3 : Configuration des clients Debian MATE

#### 3.1 PAM-LDAP

    apt install libnss-ldap libpam-ldap nscd

Renseigner : 

    - LDAP URI: ldap://192.168.15.250/ 
    - Base DN: dc=linuxisgood,dc=local

Fichier : `/etc/nsswitch.conf`

    passwd: files ldap
    group: files ldap
    shadow: files ldap

#### 3.2 Monter /home via NFS

Fichier : `/etc/fstab`:

    192.168.15.254:/srv/nfs /home nfs defaults 0 0

Test :

    mount -a


### Etape 4 : Vérifications

- Créer un utilisateur LDAP sur `srv-core`

- Se connecter avec ce compte sur n'importe quel client

- Vérifier que le `/home` vient du NFS

- Simuler une panne `DHCP1`, voir si `DHCP2` prend le relais

- Tester les résolutions DNS

- Vérifier RAID5 avec `cat /proc/mdstat`

- Vérifier VPN si déployé (OpenVPN ou autre)