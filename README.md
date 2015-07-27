#Supernode
##Batman-adv
```
sudo modprobe batman-adv
sudo batctl gwl
```
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
sudo nano dummy
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
user "fastd";

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
cd /etc/dhcp
sudo nano dhcpd.conf
```

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
