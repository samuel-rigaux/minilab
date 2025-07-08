# MiniLab - Documentation Complète & Guide Utilisateur

---

## Sommaire

* [1. Introduction et Contexte](#1-introduction-et-contexte)
* [2. Architecture et Topologie Réseau](#2-architecture-et-topologie-réseau)
* [3. Préparation des VM et Configuration Réseau](#3-préparation-des-vm-et-configuration-réseau)
* [4. Installation et Configuration des Services](#4-installation-et-configuration-des-services)

  * [4.1 Serveur Principal](#41-serveur-principal)
  * [4.2 Serveurs Secondaires DHCP/DNS/LDAP Master-Slave](#42-serveurs-secondaires-dhcpdnsldap-master-slave)
  * [4.3 Clients Debian 12 MATE](#43-clients-debian-12-mate)
* [5. Sécurisation de l’Infrastructure](#5-sécurisation-de-linfrastructure)
* [6. Script de Mise à Jour Centralisée](#6-script-de-mise-à-jour-centralisée)
* [7. Interface de Gestion Centralisée (Idée)](#7-interface-de-gestion-centralisée-idée)
* [8. PRA / PCA](#8-pra--pca)
* [9. Guide Utilisateur](#9-guide-utilisateur)
* [10. Perspectives d’Évolution](#10-perspectives-dévolution)

---

## 1. Introduction et Contexte

Tu es admin système pour une association qui utilise des PC reconditionnés sous Debian 12. Tous les utilisateurs partagent aujourd’hui un même compte, ce qui pose problème pour retrouver leurs profils sur n’importe quelle machine. Le but : configurer une infrastructure réseau avec profils itinérants via LDAP et NFS, DHCP/DNS/LDAP en haute dispo master/slave, failover IP avec Keepalived, VPN sécurisé avec OpenVPN, pare-feu, et un système facile à maintenir.

---

## 2. Architecture et Topologie Réseau

| VM                | Rôle                                          | IP Fixe LAN                  | Interfaces                     |
| ----------------- | --------------------------------------------- | ---------------------------- | ------------------------------ |
| Serveur Principal | Passerelle Internet, NFS RAID5, LDAP, OpenVPN | 192.168.15.254               | eth0 : WAN, eth1 : LAN         |
| Serveur DHCP1     | DHCP/DNS/LDAP Master                          | 192.168.15.253               | eth0 : LAN                     |
| Serveur DHCP2     | DHCP/DNS/LDAP Slave                           | 192.168.15.252               | eth0 : LAN                     |
| VIP Failover      | IP Flottante Failover Keepalived              | 192.168.15.250               | Assignée via VRRP (Keepalived) |
| Clients Debian    | Poste utilisateur avec Debian 12 MATE         | DHCP de 192.168.15.100 à 150 | eth0 : LAN                     |

---

## 3. Préparation des VM et Configuration Réseau

### 3.1 Création des machines virtuelles

* **Serveur Principal** : 2 CPU, 1 Go RAM, 8 Go disque système + 4 disques de 8 Go pour RAID5
* **Serveurs DHCP1 & DHCP2** : 1 CPU, 1 Go RAM, 8 Go disque
* **Clients Debian 12 MATE** : 1 CPU, 1 Go RAM, 16 Go disque

### 3.2 Configuration réseau des VM

---

### 3.2.1 Serveur Principal (2 interfaces)

Fichier `/etc/network/interfaces` (ou utiliser `netplan` si Debian 12 avec netplan):

```bash
auto eth0
iface eth0 inet dhcp  # Interface WAN (accès Internet)

auto eth1
iface eth1 inet static
    address 192.168.15.254
    netmask 255.255.255.0
    network 192.168.15.0
    broadcast 192.168.15.255
```

---

### 3.2.2 Serveur DHCP1 (Master)

```bash
auto eth0
iface eth0 inet static
    address 192.168.15.253
    netmask 255.255.255.0
    network 192.168.15.0
    broadcast 192.168.15.255
```

---

### 3.2.3 Serveur DHCP2 (Slave)

```bash
auto eth0
iface eth0 inet static
    address 192.168.15.252
    netmask 255.255.255.0
    network 192.168.15.0
    broadcast 192.168.15.255
```

---

### 3.2.4 Clients Debian 12 MATE

```bash
auto eth0
iface eth0 inet dhcp
```

---

### 3.3 Activer le forwarding IP sur Serveur Principal

```bash
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

---

## 4. Installation et Configuration des Services

---

### 4.1 Serveur Principal

---

#### 4.1.1 RAID5 et Serveur NFS

1. Installer mdadm et nfs-kernel-server :

```bash
sudo apt update
sudo apt install mdadm nfs-kernel-server -y
```

2. Créer RAID5 sur 4 disques (supposons `/dev/sdb`, `/dev/sdc`, `/dev/sdd`, `/dev/sde`) :

```bash
sudo mdadm --create --verbose /dev/md0 --level=5 --raid-devices=4 /dev/sdb /dev/sdc /dev/sdd /dev/sde
```

3. Formater en ext4 :

```bash
sudo mkfs.ext4 /dev/md0
```

4. Monter le RAID :

```bash
sudo mkdir -p /srv/nfs
sudo mount /dev/md0 /srv/nfs
```

5. Ajouter dans `/etc/fstab` pour montage automatique :

```
/dev/md0 /srv/nfs ext4 defaults 0 2
```

6. Configurer exports NFS dans `/etc/exports` :

```
/srv/nfs 192.168.15.0/24(rw,sync,no_subtree_check,no_root_squash)
```

7. Appliquer les exports et redémarrer NFS :

```bash
sudo exportfs -a
sudo systemctl restart nfs-kernel-server
```

8. Tester avec :

```bash
showmount -e
```

---

#### 4.1.2 Installation et Configuration OpenLDAP

1. Installer slapd et ldap-utils :

```bash
sudo apt install slapd ldap-utils -y
```

2. Configurer slapd (sera demandé pendant l’installation ou via `dpkg-reconfigure slapd`):

* Nom de domaine DNS : `linuxisgood.local`
* Organisation : `linuxisgood`
* Mot de passe admin LDAP : choisi par toi
* Ne pas déplacer la base existante
* Ne pas configurer comme client LDAP

3. Créer unité organisationnelle (OU) dans LDAP :

Créer un fichier `base.ldif` :

```ldif
dn: ou=People,dc=linuxisgood,dc=local
objectClass: organizationalUnit
ou: People

dn: ou=Groups,dc=linuxisgood,dc=local
objectClass: organizationalUnit
ou: Groups
```

4. Ajouter dans LDAP :

```bash
ldapadd -x -D cn=admin,dc=linuxisgood,dc=local -W -f base.ldif
```

---

#### 4.1.3 Configuration PAM LDAP client sur Serveur Principal

1. Installer modules PAM LDAP :

```bash
sudo apt install libpam-ldap libnss-ldap nscd -y
```

2. Modifier `/etc/nsswitch.conf` :

```
passwd:         files ldap
group:          files ldap
shadow:         files ldap
```

3. Modifier `/etc/pam.d/common-auth` pour l’authentification LDAP :

```bash
auth    required    pam_unix.so nullok_secure
auth    sufficient  pam_ldap.so use_first_pass
auth    required    pam_deny.so
```

4. Redémarrer le service nscd :

```bash
sudo systemctl restart nscd
```

---

#### 4.1.4 Installation et configuration OpenVPN

---

##### 1. Installer OpenVPN et Easy-RSA

```bash
sudo apt install openvpn easy-rsa -y
```

##### 2. Initialiser la PKI (Public Key Infrastructure)

```bash
make-cadir ~/openvpn-ca
cd ~/openvpn-ca
```

Éditer le fichier `vars` et personnaliser les champs (pays, ville, organisation, etc.).

```bash
nano vars
```

Puis :

```bash
source vars
./clean-all
./build-ca
```

##### 3. Générer le certificat serveur

```bash
./build-key-server server
```

##### 4. Générer les paramètres Diffie-Hellman

```bash
./build-dh
```

##### 5. Générer la clé TLS Auth

```bash
openvpn --genkey --secret keys/ta.key
```

##### 6. Générer les certificats clients

```bash
./build-key client1
```

Fais pareil pour chaque client.

##### 7. Copier les certificats dans `/etc/openvpn` :

```bash
sudo cp keys/ca.crt keys/server.crt keys/server.key keys/ta.key keys/dh.pem /etc/openvpn/
```

##### 8. Configurer OpenVPN serveur

Copier exemple config :

```bash
gunzip -c /usr/share/doc/openvpn/examples/sample-config-files/server.conf.gz | sudo tee /etc/openvpn/server.conf
```

Modifier `/etc/openvpn/server.conf` :

* `ca ca.crt`
* `cert server.crt`
* `key server.key`
* `dh dh.pem`
* `tls-auth ta.key 0`
* `cipher AES-256-CBC`
* `auth SHA256`
* `user nobody`
* `group nogroup`
* `server 10.8.0.0 255.255.255.0`
* `push "redirect-gateway def1 bypass-dhcp"`
* `push "dhcp-option DNS 192.168.15.250"`

##### 9. Activer forwarding IP :

```bash
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

##### 10. Configurer iptables pour OpenVPN

```bash
sudo iptables -t nat -A POSTROUTING -s 10.8.0.0/24 -o eth0 -j MASQUERADE
sudo iptables -A INPUT -i lo -j ACCEPT
sudo iptables -A INPUT -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
sudo iptables -A INPUT -p udp --dport 1194 -j ACCEPT
sudo iptables -A FORWARD -s 10.8.0.0/24 -j ACCEPT
sudo iptables -A FORWARD -d 10.8.0.0/24 -j ACCEPT
```

Sauvegarder les règles iptables :

```bash
sudo apt install iptables-persistent -y
sudo netfilter-persistent save
```

##### 11. Démarrer et activer OpenVPN

```bash
sudo systemctl enable openvpn@server
sudo systemctl start openvpn@server
```

---

### 4.2 Serveurs Secondaires DHCP/DNS/LDAP Master-Slave

---

#### 4.2.1 Keepalived Failover IP (192.168.15.250)

Installer :

```bash
sudo apt install keepalived -y
```

---

**Sur DHCP1 (Master) :** `/etc/keepalived/keepalived.conf`

```conf
vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 51
    priority 101
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1234
    }
    virtual_ipaddress {
        192.168.15.250
    }
}
```

---

**Sur DHCP2 (Backup) :** `/etc/keepalived/keepalived.conf`

```conf
vrrp_instance VI_1 {
    state BACKUP
    interface eth0
    virtual_router_id 51
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1234
    }
    virtual_ipaddress {
        192.168.15.250
    }
}
```

Démarrer keepalived sur les deux serveurs :

```bash
sudo systemctl enable keepalived
sudo systemctl start keepalived
```

---

#### 4.2.2 Installation et configuration ISC DHCP Server

Installer :

```bash
sudo apt install isc-dhcp-server -y
```

---

**DHCP1 (Master) :** `/etc/dhcp/dhcpd.conf`

```conf
authoritative;

subnet 192.168.15.0 netmask 255.255.255.0 {
    range 192.168.15.100 192.168.15.150;
    option routers 192.168.15.254;
    option domain-name-servers 192.168.15.250;
    option domain-name "linuxisgood.local";
    option broadcast-address 192.168.15.255;
    default-lease-time 600;
    max-lease-time 7200;

    failover peer "dhcp-failover" {
        primary;
        address 192.168.15.253;
        port 647;
        peer address 192.168.15.252;
        peer port 647;
        max-response-delay 60;
        max-unacked-updates 10;
        mclt 3600;
        split 128;
        load balance max seconds 3;
    }
}
```

---

**DHCP2 (Slave) :** `/etc/dhcp/dhcpd.conf`

```conf
failover peer "dhcp-failover" {
    secondary;
    address 192.168.15.252;
    port 647;
    peer address 192.168.15.253;
    peer port 647;
    max-response-delay 60;
    max-unacked-updates 10;
    load balance max seconds 3;
}
```

Démarrer DHCP sur les deux :

```bash
sudo systemctl enable isc-dhcp-server
sudo systemctl start isc-dhcp-server
```

---

#### 4.2.3 Installation et configuration Bind9 DNS Master/Slave

Installer :

```bash
sudo apt install bind9 -y
```

---

**DNS Master (DHCP1) :** `/etc/bind/named.conf.local`

```conf
zone "linuxisgood.local" {
    type master;
    file "/etc/bind/db.linuxisgood.local";
    allow-transfer { 192.168.15.252; };
    notify yes;
};
```

Créer zone `/etc/bind/db.linuxisgood.local` :

```dns
$TTL 86400
@   IN  SOA ns1.linuxisgood.local. admin.linuxisgood.local. (
            2023070301 ; Serial
            3600       ; Refresh
            1800       ; Retry
            604800     ; Expire
            86400 )    ; Negative Cache TTL

@       IN  NS      ns1.linuxisgood.local.
@       IN  NS      ns2.linuxisgood.local.

ns1     IN  A       192.168.15.253
ns2     IN  A       192.168.15.252
```

---

**DNS Slave (DHCP2) :** `/etc/bind/named.conf.local`

```conf
zone "linuxisgood.local" {
    type slave;
    file "/var/cache/bind/db.linuxisgood.local";
    masters { 192.168.15.253; };
};
```

---

Redémarrer bind9 sur les deux serveurs :

```bash
sudo systemctl restart bind9
```

Tester avec :

```bash
dig @192.168.15.253 linuxisgood.local
dig @192.168.15.252 linuxisgood.local
```

---

#### 4.2.4 Réplication LDAP Master-Slave

**Master (DHCP1) :** ajouter dans la config slapd un `syncrepl` :

Exemple LDIF `syncrepl.ldif` :

```ldif
dn: olcDatabase={1}mdb,cn=config
changetype: modify
add: olcSyncRepl
olcSyncRepl: rid=001
  provider=ldap://192.168.15.253
  bindmethod=simple
  binddn="cn=admin,dc=linuxisgood,dc=local"
  credentials=motdepasse_admin
  searchbase="dc=linuxisgood,dc=local"
  schemachecking=on
  type=refreshAndPersist
  retry="60 +"
  timeout=1
```

Appliquer :

```bash
ldapmodify -Y EXTERNAL -H ldapi:/// -f syncrepl.ldif
```

---

**Slave (DHCP2)** : configurer en `olcMirrorMode: TRUE` pour recevoir réplication.

---

### 4.3 Clients Debian 12 MATE

---

#### 4.3.1 Installer PAM LDAP client

```bash
sudo apt install libpam-ldap libnss-ldap nscd -y
```

Configurer `/etc/nsswitch.conf` :

```
passwd:         files ldap
group:          files ldap
shadow:         files ldap
```

Configurer PAM (`/etc/pam.d/common-auth`) :

```bash
auth    required    pam_unix.so nullok_secure
auth    sufficient  pam_ldap.so use_first_pass
auth    required    pam_deny.so
```

Redémarrer `nscd` :

```bash
sudo systemctl restart nscd
```

---

#### 4.3.2 Monter `/home` depuis NFS

Ajouter dans `/etc/fstab` :

```
192.168.15.254:/srv/nfs /home nfs defaults 0 0
```

Monter le partage :

```bash
sudo mount -a
```

---

## 5. Sécurisation de l’Infrastructure


* Firewall iptables sur tous les serveurs, règles strictes, uniquement ports nécessaires
* OpenVPN avec TLS Auth, chiffrement fort AES-256, authentification SHA256
* Keepalived pour haute disponibilité IP
* Mots de passe LDAP forts
* Surveillance logs et alertes

---

## 6. Script de Mise à Jour Centralisée

Exemple script `update-all.sh` :

```bash
#!/bin/bash

SERVERS=("192.168.15.253" "192.168.15.252" "192.168.15.254")

for server in "${SERVERS[@]}"; do
  echo "Mise à jour sur $server"
  ssh admin@$server "sudo apt update && sudo apt upgrade -y"
done
```

---

## 7. Interface de Gestion Centralisée (Idée)

* Installer Cockpit ou Webmin sur serveur principal
* Permet gestion facile des utilisateurs, services, journaux

---

## 8. PRA / PCA

* Sauvegardes régulières LDAP, NFS, configurations
* Tests de basculement Keepalived, DHCP failover
* Scripts d’automatisation restauration

---

## 9. Guide Utilisateur


### 9.1 Connexion au VPN

* Installer OpenVPN client
* Récupérer le fichier `.ovpn` personnalisé (généré via Easy-RSA)
* Lancer connexion :

```bash
sudo openvpn --config client1.ovpn
```

---

### 9.2 Connexion au poste Debian

* Se connecter avec son utilisateur LDAP :

```bash
ssh utilisateur@ip_client
```

ou directement à la console

* Le home est monté automatiquement via NFS

---

### 9.3 Comment changer son mot de passe

```bash
passwd
```

---

### 9.4 Dépannage

* Pas d’accès au VPN → vérifier internet + config client
* Pas d’authentification LDAP → vérifier mot de passe + état serveur LDAP
* Home non monté → vérifier montage NFS avec `mount`

---

## 10. Perspectives d’Évolution

* Automatisation complète avec Ansible
* Mise en place MDM pour gestion poste utilisateur
* Passage à Kerberos + LDAP pour sécurité accrue
* Mise en place d’une solution de supervision (Zabbix, Prometheus)

---
