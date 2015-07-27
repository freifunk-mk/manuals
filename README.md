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
