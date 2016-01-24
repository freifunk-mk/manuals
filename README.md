#Anleitung zum Aufsetzen eines Freifunk Supernodes auf Ubuntu Server 16.04 LTS
Die Anleitung ist in Arbeit.
Bitte in den Branches schauen für andere / aktuellere Versionen dieser Anleitung!
##Einleitung
Wir bauen hier ein Active/Passive Failover Setup mit einem Aktiven Supernode für die Domäne und einem zweiten Standby node für die Ausfallsicherheit.
#Supernode
##Installation
Nichts besonderes, bei der Gelegenheit einfach gleich den OpenSSH Server mitinstallieren.
##ssh Key hinterlegen
```
nano ~/.ssh/authorized_keys
```
Key einfügen, Datei speichern, nano beenden und server rebooten.
```
sudo reboot
```
per keyfile am Server anmelden und wenn das klappt den login per passwort abschalten
```
sudo nano /etc/ssh/sshd_config
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
sudo apt-get dist-upgrade
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
##Batman-adv
http://www.open-mesh.org/projects/open-mesh/wiki/Download
```
cd ~
wget http://downloads.open-mesh.org/batman/stable/sources/batman-adv/batman-adv-2015.1.2.tar.gz
tar -xf batman-adv-2015.1.2.tar.gz
cd batman-adv-2015.1.2
sudo apt-get install build-essential
make
sudo make install
sudo apt-get install batctl
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
sudo mkdir client
cd client
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
peer limit 200;
hide ip addresses yes;
mtu 1280;
secure handshakes yes;
log to syslog level verbose;

on up "
        ip link set address 04:EE:EF:CA:FE:3A dev tap0
        ip link set up tap0
        batctl -m bat0 if add $INTERFACE
        ip link set address 02:EE:EF:CA:FE:FF:3A dev bat0
        ip link set up dev bat0
        brctl addif br0 bat0
        batctl -m bat0 it 5000
        batctl -m bat0 bl 0
        batctl -m bat0 gw server 48mbit/48mbit
        batctl -m bat0 vm server
";

on verify "
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
subnet 172.16.0.0 netmask 255.255.0.0 {
        range 172.16.1.1 172.16.10.254;
        default-lease-time 300;
        max-lease-time 600;
        option domain-name-servers 8.8.8.8;
        option routers 172.16.0.1;
        interface br0;
}
```
```
sudo reboot
```

Hier klafft eine große lücke

#Monitoring
##Check_MK
```
sudo apt-get install xinetd gdebi
```
```
sudo gdebi check-mk-agent_1.2.6p15-1_all.deb
```
##vnstat
```
sudo apt-get install vnstat vnstati lighttpd
cd /var/www
sudo mkdir vnstats
cd vnstats
mkdir eth0
cd ..
sudo rm *.html
sudo nano index.html
```
```
<img src="vnstats/eth0/summary.png"><br>
<img src="vnstats/eth0/hours.png"><br>
<img src="vnstats/eth0/days.png"><br>
<img src="vnstats/eth0/months.png"><br>
```
```
sudo crontab -e
```
Vier Zeilen hinzufügen
```
*/5 * * * * vnstati -i eth0 -o /var/www/vnstats/eth0/hours.png -h
*/5 * * * * vnstati -i eth0 -o /var/www/vnstats/eth0/days.png -d
*/5 * * * * vnstati -i eth0 -o /var/www/vnstats/eth0/months.png -m
*/5 * * * * vnstati -i eth0 -o /var/www/vnstats/eth0/summary.png -s
```
