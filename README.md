#Climate Monitoring (Temp/Hum) on a Raspberry Pi using InfluxDB and Grafana


Install influxdb https://docs.influxdata.com/influxdb/v0.13/introduction/installation/

#Create climate database:

Get influx prompt
```
influx
```

Create database
```
CREATE DATABASE "climate"
```

Create script to grab climate data and write to db

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
