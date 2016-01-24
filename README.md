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
sudo apt install libnl-3-dev
cd ~
wget http://downloads.open-mesh.org/batman/stable/sources/batman-adv/batman-adv-2016.0.tar.gz
tar -xf batman-adv-2016.0.tar.gz
cd batman-adv-2016.0
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
```
# /etc/modules: kernel modules to load at boot time.
#
# This file contains the names of kernel modules that should be loaded
# at boot time, one per line. Lines beginning with "#" are ignored.
batman-adv
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
##Netzwerkstuff
```
sudo nano /etc/network/interfaces
```
Einmal ein Interface mit der FFRL Public Nat IPv4 anlegen
```
auto tun-ffrl-uplink
iface tun-ffrl-uplink inet static
        address 185.66.195.52
        netmask 255.255.255.255
        pre-up ip link add $IFACE type dummy
        post-down ip link del $IFACE
```
Und ein Bridge Interface für das FF Netz
```
auto br0
iface br0 inet static
        address 172.17.0.1
        netmask 255.255.0.0
        bridge_ports none
        bridge_stp no
        post-up ip route add 172.17.0.0/16 dev br0 table 42

iface br0 inet6 static
        address 2a03:2260:120:xxx::1
        netmask 48
        post-up ip -6 route add 2a03:2260:120:xxx::/56 dev br0 table 42
```
Und Gre Tunnel für jeden Backbone standort:
```
# bb-a.ak.ber
auto  tun-ffrl-ber-a
iface tun-ffrl-ber-a inet tunnel
        mode            gre
        netmask         255.255.255.254
        address         100.64.2.xxx
        dstaddr         100.64.2.xxx
        endpoint        185.66.195.0
        local          xx.xxx.xx.xx
        ttl             255
        mtu             1400
        post-up ip -6 addr add 2a03:2260:0:xxx::2/64 dev $IFACE

```
##Routingstuff
```
sudo nano /etc/rc.local
```
```
#!/bin/sh -e
# rc.local

ip -4 rule add prio 1000 from 172.xx.0.0/16 table internet
ip -6 rule add prio 1000 from 2a03:2260:120:xxx::/56 table internet

ip -4 rule add prio 1000 fwmark 0x1 table internet
ip -6 rule add prio 1000 fwmark 0x1 table internet

FFRL_IFS="tun-ffrl-dus-a tun-ffrl-dus-b tun-ffrl-ber-a tun-ffrl-ber-b"
for interface in $FFRL_IFS; do
    ip -4 rule add prio 1001 iif $interface table internet
    ip -6 rule add prio 1001 iif $interface table internet
done

# unreachable default route, damit ihr freifunk pakete nie über die eth0 defaultroute schickt
ip -4 rule add prio 2000 from 172.xx.0.0/16 table unreachable
ip -4 route add default unreachable table unreachable

#ip route add 172.xx.0.0/16 dev br0 table 42

exit 0
```
Und Ferm installieren
```
sudo apt-get install ferm
```
und eine config erstellen
```
sudo nano /etc/ferm/ferm.conf
```
```
# -*- shell-script -*-
#
#  Configuration file for ferm(1).
#

domain (ip ip6) {
    table filter {
        chain INPUT {
            policy ACCEPT;

            proto gre ACCEPT;

            # connection tracking
            mod state state INVALID DROP;
            mod state state (ESTABLISHED RELATED) ACCEPT;

            # allow local packet
            interface lo ACCEPT;

            # respond to ping
            proto icmp ACCEPT;

            # allow IPsec
            proto udp dport 500 ACCEPT;
            proto (esp) ACCEPT;

            # allow SSH connections
            proto tcp dport ssh ACCEPT;
        }
        chain OUTPUT {
            policy ACCEPT;

            # connection tracking
            #mod state state INVALID DROP;
            mod state state (ESTABLISHED RELATED) ACCEPT;
        }
        chain FORWARD {
            policy ACCEPT;

            # connection tracking
            mod state state INVALID DROP;
            mod state state (ESTABLISHED RELATED) ACCEPT;
        }
    }

    table mangle {
        chain PREROUTING {
            interface tun-ffrl-+ {
                # sockmark, hier ohne bedingung, ist die wichtig? also port 53 und so (zum direkt rauswerfen? dann brauchts noch ein nat gegen die ip auf et$
                # erstmal so funktionierend)
                MARK set-mark 1;
            }
        }

        chain POSTROUTING {
            # mss clamping
            outerface tun-ffrl-+ proto tcp tcp-flags (SYN RST) SYN TCPMSS clamp-mss-to-pmtu;
        }
    }

    table nat {
        chain POSTROUTING {
            # nat translation
            outerface tun-ffrl-+ saddr 172.xx.0.0/16 SNAT to 185.66.195.xx;
            policy ACCEPT;
            outerface tun-ffrl-+ {
                MASQUERADE;
            }
        }
    }
}
```
Eine Runde Bird installieren
```
sudo apt-get install bird
```
Und configs anlegen
```
sudo nano /etc/bird/bird.conf
```
```
#table ffrl;
router id 185.66.195.xx;

protocol direct announce {
        table master; # implizit
        import where net ~ [185.66.195.xx/32];
        interface "tun-ffrl-uplink";
};

protocol kernel {
        table master;
        device routes;
        import none; # ihr wollt nichts aus der kernel routing tabelle lernen
        export filter {
                #  setze src addr beim route-export in kernel tabelle
                krt_prefsrc = 185.66.195.xx;
                accept;
        };
        kernel table 42;
};

protocol device {
        scan time 15;
};

function is_default() {
        return (net ~ [0.0.0.0/0]);
};

template bgp uplink {
        local as 65xxx;
        import where is_default();
        export where proto = "announce";
};

protocol bgp ffrl_ber_a from uplink {
        source address 100.64.2.xxx;
        neighbor 100.64.2.xxx as 201701;
};

```
Den letzten Block für alle backbone standorte analog zu den gre tunneln wiederholen

Nun nochmal das ganze in IPv6
```
sudo nano /etc/bird/bird6.conf
```
```
#table ffrl;
router id 185.66.195.xx;

protocol direct announce {
        table master; # implizit
        import where net ~ [2a03:2260:120:xxx::/56];
        interface "tun-ffrl-uplink";
};

protocol kernel {
        table master;
        device routes;
        import none; # ihr wollt nichts aus der kernel routing tabelle lernen
        export filter {
                #  setze src addr beim route-export in kernel tabelle
                krt_prefsrc = 2a03:2260:120:xxx::1;
                accept;
        };
        kernel table 42;
};

protocol device {
        scan time 15;
};

function is_default() {
        return (net ~ [::/0]);
};

template bgp uplink {
        local as 65xxx;
        import where is_default();
        export where proto = "announce";
};

protocol bgp ffrl_ber_a from uplink {
        source address 2a03:2260:0:xxx::2;
        neighbor 2a03:2260:0:xxx::1 as 201701;
};
```
Und den letzten Block wieder entsprechend vervielfältigen

##Testen
einmal neustarten
```
sudo reboot
```
Und jetzt mal die bgp4 session prüfen
```
sudo birdc s p
```
Sollte nun so aussehen
```
BIRD 1.4.0 ready.
name     proto    table    state  since       info
announce Direct   master   up     2016-01-19  
kernel1  Kernel   master   up     2016-01-19  
device1  Device   master   up     2016-01-19  
ffrl_ber_a BGP      master   up     2016-01-19  Established   
ffrl_ber_b BGP      master   up     2016-01-19  Established   
ffrl_dus_a BGP      master   up     2016-01-19  Established   
ffrl_dus_b BGP      master   up     2016-01-19  Established 
```
Und für IPv6
```
sudo birdc6 s p
```
Sollte so aussehen
```
BIRD 1.4.0 ready.
name     proto    table    state  since       info
announce Direct   master   up     2016-01-19  
kernel1  Kernel   master   up     2016-01-19  
device1  Device   master   up     2016-01-19  
ffrl_ber_a BGP      master   up     2016-01-19  Established   
ffrl_ber_b BGP      master   up     2016-01-19  Established   
ffrl_dus_a BGP      master   up     2016-01-19  Established   
ffrl_dus_b BGP      master   up     2016-01-19  Established   
```
Wenn es nicht direkt läuft einfach mal 2-5 Minuten warten, bgp kann dauern

Dann mal sehen ob batman nun läuft



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
