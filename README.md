#Climate Monitoring (Temp/Hum) on a Raspberry Pi using InfluxDB and Grafana

Just sharing my experience getting this working. Depending on your hardware and Linux distro, you experience may be different.

1) Installed Arch on Raspberry Pi 3: https://archlinuxarm.org/platforms/armv8/broadcom/raspberry-pi-3

2) Install yaourt (aur helper): https://aur.archlinux.org/packages/yaourt/

3) Install influxdb https://docs.influxdata.com/influxdb/v0.13/introduction/installation/
In my case on Arch:
```
yaourt -S influxdb
```

4) Create climate database:

Get influx prompt
```
influx
```

Create database
```
CREATE DATABASE "climate"
```

5) Create a script to grab climate data and write to db. This script grabs temperature and humidity from a DHT22 connected to GPIO 4. Using the Adafruit python script to grab query the sensor. I'm also reading the temp from the CPU on the Raspberry Pi CPU and save to the telegraf db.

grab_n_write.sh 
```
#!/bin/bash

# Read DHT22 Temp/Hum
climate=$(python /usr/share/nginx/html/AdafruitDHT.py 2302 4)

# Split temp / hum into varibles
temp=$(echo $climate | cut -d":" -f1)
hum=$(echo $climate | cut -d":" -f2)

# Get Pi CPU Temp
picputempc=$(/opt/vc/bin/vcgencmd measure_temp | cut -d "=" -f2 | cut -d"'" -f1)

# Convert C to F
picputempf=$(echo "scale=2; $picputempc*9/5+32" | bc -l)
#picputempf=$(echo "scale=2; $(/opt/vc/bin/vcgencmd measure_temp | cut -d "=" -f2 | cut -d"'" -f1)*9/5+32" | bc -l)

# Write to influxdb
influx -database 'climate' -execute "INSERT temperature,sensor=1 value=$temp"
influx -database 'climate' -execute "INSERT humidity,sensor=1 value=$hum"
influx -database 'telegraf' -execute "INSERT cputemp value=$picputempf"
```


Create a systemd service and timer

/usr/lib/systemd/system/climatemonitor.service
```
[Unit]
Description=Climate Monitoring Service

[Service]
Type=simple
ExecStart=/bin/bash /usr/share/nginx/html/grab_n_write.sh
```

/usr/lib/systemd/system/climatemonitor.timer 
```
[Unit]
Description=Run climatemonitor.service every 30 sec

[Timer]
# Time to wait after booting before we run first time
OnBootSec=30
# Time between running each consecutive time
OnUnitActiveSec=30
Unit=climatemonitor.service
```

Start and Enable service/timer
```
systemctl start climatemonitor.timer
systemctl enable climatemonitor.timer 
systemctl status climatemonitor.timer
```

#Build Grafana from source:

http://docs.grafana.org/project/building_from_source/

Build Grafana backend
```
export GOPATH=$(pwd)
go get github.com/grafana/grafana
go get github.com/grafana/grafana
cd $GOPATH/src/github.com/grafana/grafana
go run build.go setup 
$GOPATH/bin/godep restore && go run build.go build
```
Build frontend
```
sudo pacman -S nodejs nodejs-grunt-cli npm phantomjs
npm install node-sass
npm install
grunt
```
If all goes well you should be able to start grafana server
```
./bin/grafana-server
```
Connect on port 3000

To Do:
Secure things with password changes and such

Add to script to drop differences in value of > 10 from one polling to the next. Here and there I get huge downward spikes of 20 which is impossible. Just a bad reading I think. The script should drop those.

Create service for grafana-server
