# Cluster node energy monitoring with IPMI, influxDB and grafana

## On every compute node

### 1. Configure read-only IPMI user

```
sudo ipmitool user set name 15 pwrmon

sudo ipmitool user set password 15 "IPMIpwdPOWER"

sudo ipmitool user enable 15

sudo ipmitool channel setaccess 1 15 callin=on ipmi=on link=on privilege=2

sudo ipmitool lan set 1 access on
```

### 2. Get IPMI IP address

```
sudo ipmitool lan print 1 | grep "IP Address  "
```

In the following, we will assume 4 nodes `172.16.1.1` - `172.16.1.4`

## On the master node

(instruction for ubuntu 22.04 desktop, following https://www.howtoforge.com/how-to-install-tig-stack-telegraf-influxdb-and-grafana-on-ubuntu-22-04/)

### 1. Check IPMI connection

```
sudo apt install ipmitool

ipmitool -I lanplus -H 172.16.1.1 -U pwrmon -P IPMIpwdPOWER -L USER sensor | grep Pwr
```

Alternative command (DCMI):
```
ipmitool -I lanplus -H 172.16.1.1 -U pwrmon -P IPMIpwdPOWER -L USER dcmi power reading
```
NOTE: you can stop here and just use the above commands to get power measurements into your existing monitoring system (e.g. checkmk)

### 2. Configure firewall

```
sudo ufw enable
#add ssh
sudo ufw allow 22
#add influxDB
sudo ufw allow 8086
#add grafana
sudo ufw allow 3000
```

### 3. Install influxdb2

#### 3.1 Add custom repo

```
wget -q https://repos.influxdata.com/influxdb.key

echo '23a1c8836f0afc5ed24e0486339d7cc8f6790b83886c4c96995b88a061c5bb5d influxdb.key' | sha256sum -c && cat influxdb.key | gpg --dearmor | sudo tee /etc/apt/trusted.gpg.d/influxdb.gpg > /dev/null

echo 'deb [signed-by=/etc/apt/trusted.gpg.d/influxdb.gpg] https://repos.influxdata.com/debian stable main' | sudo tee /etc/apt/sources.list.d/influxdata.list
```

#### 3.2. Install package

```
sudo apt update
sudo apt install influxdb2
```

#### 3.3. Start service
```
sudo systemctl start influxdb
sudo systemctl status influxdb
```

#### 3.4. Configuration

Run
```
influx setup
```
and fill in:
```
username: influxDB
passwd: influxDB_pwd
organization: MYORG
period: 0 infinite

bucket: power
```

#### 3.5. Web login

USE CHROMIUM and not firefox to connect to 
```
http://localhost:8086
```
Click on the Generate Token button and select the Read/Write Token option to launch a new overlay popup. Give a name to the Token (telegraf) and select the default bucket we created under both Read and Write sections. 

Sample token:
```
75Od8v-XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX-wYw==
```

### 4. Install telegraf

#### 4.1. Install package
```
sudo apt install telegraf
```

#### 4.2. Configure telegraf

```
sudo vi /etc/telegraf/telegraf.conf
```
Configure influxdb output:
```
[[outputs.influxdb_v2]]
urls = ["udp://127.0.0.1:8086"]
token = "75Od8v-XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX-wYw=="
organization = "MYORG"
bucket = "power"
```
Configure IPMI input:
```
[[inputs.ipmi_sensor]]
servers = ["pwrmon:IPMIpwdPOWER@lanplus(172.16.1.1)", "pwrmon:IPMIpwdPOWER@lanplus(172.16.1.2)", "pwrmon:IPMIpwdPOWER@lanplus(172.16.1.3)", "pwrmon:IPMIpwdPOWER@lanplus(172.16.1.4)"]
interval = "30s"
timeout = "20s"
privilege = "USER"
metric_version = 2
```

#### 4.3. Restart to apply changes:
```
sudo systemctl restart telegraf
```
#### 4.4. Validate that data is loaded to influxDB:

```
localhost:8086 -> login -> data explorer -> from: power -> _measurement: ipmi_sensors -> name: pwr_consumption -> SUBMIT
```

### 5. Install grafana

#### 5.1. Add custom repo

```
sudo wget -q -O /usr/share/keyrings/grafana.key https://packages.grafana.com/gpg.key

echo "deb [signed-by=/usr/share/keyrings/grafana.key] https://packages.grafana.com/oss/deb stable main" | sudo tee -a /etc/apt/sources.list.d/grafana.list
```

#### 5.2. Install package

```
sudo apt update
sudo apt install grafana
```

#### 5.3. Enable systemd service:

```
sudo systemctl enable grafana-server --now
sudo systemctl status grafana-server
```

#### 5.4. Test connection:

```
http://<serverIP>:3000
admin:admin
```

Set new password

#### 5.5. Connect to influxDB

Configuration -> Datasources -> InfluxDB

```
Query Language: Flux
URL: http://localhost:8086
Basic auth: ON
User: influxDB
Password: influxDB_pwd
Organization: MYORG
Default bucket: power
```

#### 5.6. Create plots

Sample power graph:
```
from(bucket: "power")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r["_measurement"] == "ipmi_sensor")
  |> filter(fn: (r) => r["_field"] == "value")
  |> filter(fn: (r) => r["name"] == "pwr_consumption")
  |> filter(fn: (r) => r["entity_id"] == "7.1")
  |> filter(fn: (r) => r["server"] == "172.16.1.1")
  |> aggregateWindow(every: v.windowPeriod, fn: mean, createEmpty: true)
  |> yield(name: "mean")
```

Sample energy over period graph:
```
from(bucket: "power")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r["_measurement"] == "ipmi_sensor")
  |> filter(fn: (r) => r["name"] == "pwr_consumption")
  |> filter(fn: (r) => r["_field"] == "value")
  |> filter(fn: (r) => r["entity_id"] == "7.1")
  |> filter(fn: (r) => r["server"] == "172.16.1.1")
  |> aggregateWindow(every: 2m, fn: mean, createEmpty: true)
  |> integral(unit: 1h, interpolate: "")
```
