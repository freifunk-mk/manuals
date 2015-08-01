#Anleitung zum Aufsetzen eines Freifunk Supernodes auf Ubuntu Server 14.04 LTS
Die Anleitung ist in Arbeit.
#Supernode
##Installation
Nichts besonderes, bei der Gelegenheit einfach gleich den OpenSSH Server mitinstallieren.
##ssh Key hinterlegen
```
nano ~/.ssh/authorized_key
```
Key einfügen, Datei speichern, nano beenden und server rebooten.
```
sudo reboot
```
per keyfile am Server anmelden und wenn das klappt den login per passwort abschalten
```
sudo nano /etc/ssh/ssh_config
```
Folgende Zeilen hinzufügen und speichern und server erneut rebooten.
```
PasswordAuthentication no
UsePAM no
```
```
sudo reboot
```
##System Aktualisieren
```
sudo apt-get update
sudo apt-get dist upgrade
```
##Fastd einrichten
sudo nano /etc/apt/sources.list
```
deb http://repo.universe-factory.net/debian/ sid main
```
```
sudo apt-get update
sudo apt-get install fastd
```
```
cd /etc/fastd
sudo mkdir backbone
cd backbone
fastd --generate-key
sudo nano secret.conf
secret "xxx";
```
```
bind any:10001 default ipv4;
include "secret.conf";
include peers from "server";
interface "tap1";
log level info;
mode tap;
method "null";
method "salsa2012+umac";
mtu 1364;
secure handshakes yes;
log to syslog level verbose;
```
```
sudo mkdir server
cd server
sudo nano fichtenfunk01
```
```
# Knotenname: node01
key "xxx";
remote ipv4 "fichtenfunk01.freifunk.ruhr" port 10001;

```
##Batman-adv
http://www.open-mesh.org/projects/open-mesh/wiki/Download
cd ~
wget http://downloads.open-mesh.org/batman/stable/sources/batman-adv/batman-adv-2015.0.tar.gz
tar -xf batman-adv-2015.0.tar.gz
cd batman-adv-2015.0
sudo apt-get install build-essential
make
sudo make install
sudo apt-get install batctl

```
sudo modprobe batman-adv
sudo batctl gwl
```
Error - mesh has not been enabled yet
Activate your mesh by adding interfaces to batman-adv
batman-adv in die /etc/modules eintragen
```
sudo nano /etc/modules
```
https://wiki.ubuntuusers.de/Kernelmodule#start

##Client Fastd
```
cd /etc/fastd
sudo mkdir fichtenfunk
cd fichtenfunk
```
```
sudo mkdir dummy
```
```
sudo nano fastd.conf
```
```
bind any:10000 default ipv4;
include "secret.conf";
include peers from "dummy";
interface "tap0";
log level info;
mode tap;
method "salsa2012+umac";
method "null";
peer limit 200;
hide ip addresses yes;
mtu 1280;
secure handshakes yes;
log to syslog level verbose;
status socket "/tmp/fastd.sock";

on up "
  ip link set address 04:EE:EF:CA:FE:3A dev tap0
  ip link set up dev tap0
  batctl -m bat0 if add $INTERFACE
  ip link set address 02:EE:EF:CA:FF:3A dev bat0
  ip link set up dev bat0
  brctl addif br0 bat0
";

on verify "
  /etc/fastd/fastd-blacklist.sh $PEER_KEY
";
```
```
fastd --generate-key
```
```
sudo nano secret.conf
```
```
secret "xxx";
```
##DHCP Server
```
sudo apt-get install isc-dhcp-server 
cd /etc/dhcp
sudo nano dhcpd.conf
```
```
authoritative;
subnet 10.224.0.0 netmask 255.255.0.0 {
        range 10.224.25.0 10.224.32.254;
        default-lease-time 300;
        max-lease-time 600;
        option domain-name "ffis01.freifunk-iserlohn.de";
        option domain-name-servers 10.224.24.1;
        option broadcast-address 10.224.24.255;
        option subnet-mask 255.255.0.0;
        option routers 10.224.24.1;
        interface br0
}
```
```
sudo reboot
```
##Munin
```
sudo apt-get install munin-node
```
```
sudo nano /etc/munin/munin-node.conf
```
```
allow ^89\.163\.150\.82$
```
port 4949 im host frei machen

#Mapserver
##Backend
aliases.json
```
chown ffmap /var/run/alfred.sock
cd /home/chrisno/ffmap-backend && python3 backend.py --with-rrd -d /var/www/json/ -a /home/chrisno/ffmap-backend/aliases.json
```

##Frontend meshviewer
```
grunt
```
nach /var/www/meshviewer/

Cache leeren
```
cd /var/www/json
sudo rm nodes.json
```
