# Pi-Network-Monitor
Network Monitor using Raspberry Pi, Docker, Grafana and Telegraf

## Description

This project uses a Raspberry Pi 3B and a 32 GByte SD card. It is connected as in a headless configuration with only a wired Ethernet connection. WireGuard is used to provide the VPN services.

## Raspian Installation

Install Raspian Buster Lite on the SD Card.

1. Download the latest version of Raspbian Buster Lite from [Raspberry Pi Downloads](https://www.raspberrypi.org/downloads/raspbian/).
1. Format the SD Card.
   1. Windows or Mac - Use SD Card Formatter from [SD Association](https://www.sdcard.org/downloads/formatter/).
1. Burn the Raspbian Buster Lite image to the SD Card.
   1. Windows or Mac - Use balaenaEtcher from [balena](https://www.balena.io/etcher/).
   1. There is **NO** need to unzip the Raspbian Buster Lite file as Etcher will handle it.
1. Create an empty file in the root partition of the SD Card with the name **ssh**.
1. Put the SD Card in the Raspberry Pi.
1. Connect up a wired Internet connection.
1. Power up the Raspberry Pi.
1. Find the IP Address for the Raspberry Pi. Will be the IP Address with port 22 (ssh) open.
   1. Windows - Use [Angry IP Scanner](https://angryip.org/).
   1. Mac - Use Terminal.
      1. In Finder open Terminal which is found in the Applications -> Utilities Folder.
      1. Enter the command `arp -a`.
         1. Raspberry Pi's will have MAC addresses with the format **B8:27:EB:xx:xx:xx** or **DC:A6:32:xx:xx:xx**.
1. SSH into the Raspberry Pi by connecting to the IP Address that was found above.
   1. Windows - Use [PuTTY](https://www.putty.org/).
   1. Mac - Use Terminal.
1. Use Userid: **pi** and Password: **raspberry**.
   ```script
   login as: pi
   pi@IP-ADDRESS's password: raspberry
   ```
1. Mount the /tmp, /var/tmp, /var/log directories into RAM (optional, but will reduce the ware on the micro SD card, but the log files will be lost on reboot and the available RAM will be reduced by 230 MB).
   ```shell
   sudo nano /etc/fstab
   ```
   1. Once editing the file enter the following at the bottom of the file.
   ```shell
   /tmpfs /tmp     tmpfs defaults,noatime,nosuid,size=100m           0 0
   /tmpfs /var/tmp tmpfs defaults,noatime,nosuid,size=30m            0 0
   /tmpfs /var/log tmpfs defaults,noatime,nosuid,size=100m,mode=0755 0 0
   ```
   1. Press **^o** then **Enter** then **^x** to save and exit.

1. Configure the Raspberry Pi
   ```
   $ sudo raspi-config
   ```
   1. Change User Password - Change password for the 'pi' user
   1. Network Options - Configure network settings
      1. Hostname - Set the visible name for this Pi on a network
      1. Wireless LAN - Enter SSID and passphrase (**OPTIONAL**)
   1. Localization Options - Set up language and regional settings to match your location
      1. Change Locale - Set up language and regional settings to match your location
      1. Change Time Zone - Set up timezone to match your location
      1. Change Keyboard Layout - Set the keyboard layout to match your keyboard
   1. Finish to leave the configuration, but **DON'T** reboot just yet.
1. Edit the pi user .bashrc file.
   ```shell
   nano ~/.bashrc
   ```
   1. Enter the above aliases to avoid making mistakes.
      ```shell
      alias rm='rm -i'
      alias cp='cp -i'
      alias mv='mv -i'
      ```
1. **Note** Optionally set a Static IP address
   ```script
   sudo nano /etc/dhcpcd.conf
   ```
   1. Scroll down to the commented out section Example static IP configuration and modify it accordingly
    
1. Update the Raspberry Pi Software.
   ```
   $ sudo apt update
   $ sudo apt upgrade -y
   ```
1. Reboot the pi.
```
$ sudo shutdown -r now
```

## Install Docker

### Create temporary work directory, change into it and download the installation script
```
$ mkdir work
$ cd work
$ curl -fsSL https://get.docker.com -o get-docker.sh
```

### Install docker
```
$ sudo sh ./get-docker.sh
```

### Add pi user to the Docker group
Reload the groups by entering a new shell, then execute the `groups` command to ensure that the pi user is a member of the groups
```
$ sudo usermod -aG docker pi
$ su -l pi
Password: PI-PASSWORD
$ groups
```

### Check the version of docker to ensure it is running
```
$ docker --version
```

### Run the Hello World
This will ensure that docker has been set up correctly. If the Hello World image is not on the system it will first will download the image, install and run the hello-world program
```
$ docker run hello-world
```

### Check docker images
```
$ docker image ls
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
hello-world         latest              851163c78e4a        9 months ago        4.85kB
$
```

### Remove hello-world
```
$ docker container ls -a
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS                      PORTS               NAMES
4d072d07b557        hello-world         "/hello"            55 seconds ago      Exited (0) 54 seconds ago                       wizardly_mccarthy
$ docker container rm 4d072d07b557
$ docker image rm hello-world
Untagged: hello-world:latest
Untagged: hello-world@sha256:8c5aeeb6a5f3ba4883347d3747a7249f491766ca1caa47e5da5dfcf6b9b717c0
Deleted: sha256:851163c78e4ad68e6fe5391f0894aafd164d40c4d4d0a56b4291f0dc2c75cc2c
Deleted: sha256:2536d8d4e4b1baa6515d44eb77a1402d6be0a533e7d191c51cb8428ba5ece3f4
$
```

## Upgrade Docker if required
```
$ sudo apt upgrade
```

## Install InfluxDB

Create the user for InfluxDB
```
$ sudo useradd -rs /bin/false influxdb
```

Create the configuration directory for InfluxDB
```
$ sudo mkdir -p /etc/influxdb
```

Create InfluxDB configuration file using Docker and reassign the permissions so we can use it
```
$ docker run --rm influxdb influxd config | sudo tee /etc/influxdb/influxdb.conf > /dev/null
$ sudo chown -R influxdb:influxdb /etc/influxdb/*
```

Create InfluxDB data directory and reassign the permissions so we can use it
```
$ sudo mkdir -p /var/lib/influxdb
$ sudo chown -R influxdb:influxdb /var/lib/influxdb
```

### InfluxDB Initialization Script

Create scripts directory
```
$ sudo mkdir -p /etc/influxdb/scripts
```

Create a script in the directory
```
$ cd /etc/influxdb/scripts
$ sudo nano influxdb-init.iql
```

Add the following 2 lines to influxdb-init.iql
```
CREATE DATABASE weather;
CREATE RETENTION POLICY one_week ON weather DURATION 168h REPLICATION 1 DEFAULT;
```

Change the permissions
```
$ sudo chown -R influxdb:influxdb /var/lib/influxdb
$ sudo chown -R influxdb:influxdb /etc/influxdb
```

Update meta database
```
$ docker run --rm -e INFLUXDB_HTTP_AUTH_ENABLED=true -e INFLUXDB_ADMIN_USER=admin -e INFLUXDB_ADMIN_PASSWORD=PUT-ADMIN-PASSWORD-HERE -v /var/lib/influxdb:/var/lib/influxdb -v /etc/influxdb/scripts:/docker-entrypoint-initdb.d influxdb /init-influxdb.sh
```

Ensure that the logs are printed out and if the intitialization script was entered ensure that there is a logline printed out of it `/init-influxdb.sh: running /docker-entrypoint-initdb.d/influxdb-init.iql`

Check that the meta.db file has the changes that were written correctly
```
$ cat /var/lib/influxdb/meta/meta.db | grep one_week
Binary file (standard input) matches
```

Verify that the correct information is in the configuration file
```
$ less influxdb.conf

[meta]
  dir = "/var/lib/influxdb/meta"

[data]
  dir = "/var/lib/influxdb/data"
  wal-dir = "/var/lib/influxdb/wal"

[http]
  bind-address = ":8086" 
```

### Running InfluxDB container on Docker

Ensure nothing is running on port 8086
```
sudo netstat -tulpn | grep 8086
```

Find the InfluxDB user ID
```
$ cat /etc/passwd | grep influxdb
influxdb:x:999:994::/home/influxdb:/bin/false
```

From the above line the InfluxDB user ID is 999 with a group ID of 994

Start up InfluxDB (will get the container id if successful)
```
$ docker run -d -p 8086:8086 --user 999:999 --name=influxdb -v /etc/influxdb/influxdb.conf:/etc/influxdb/influxdb.conf -v /var/lib/influxdb:/var/lib/influxdb influxdb -config /etc/influxdb/influxdb.conf
abf758e0179dd3dc3c6642e52c91d53a3fe241f04ae6cebda331f1639f71d2cb
```

The container can be checked and should see the same id
```
$ docker container ls
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                    NAMES
abf758e0179d        influxdb            "/entrypoint.sh -con…"   19 seconds ago      Up 16 seconds       0.0.0.0:8086->8086/tcp   influxdb
```

### Test the InfluxDB Container

Check the HTTP API is correctly enabled
```
$ curl -G http://localhost:8086/query --data-urlencode "q=SHOW DATABASES"
{"results":[{"statement_id":0,"series":[{"name":"databases","columns":["name"],"values":[["weather"]]}]}]}
```

Check that InfluxDB server is correctly listening on port 8086
```
$ netstat -tulpn | grep 8086
tcp6  0   0 :::8086    :::*    LISTEN   -
```

Check the databases (enter ^D to exit InfluxDB)
```
$ docker exec -it influxdb influx
Connected to http://localhost:8086 version 1.8.3
InfluxDB shell version: 1.8.3
> SHOW USERS
user  admin
----  -----
admin true
> SHOW DATABASES
name: databases
name
---
weather
> exit
```

### Enable Authentication on InfluxDB (if the admin user is not already set)

Get the container id
```
docker container ls
$ docker container ls
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                    NAMES
abf758e0179d        influxdb            "/entrypoint.sh -con…"   2 minutes ago       Up 2 minutes        0.0.0.0:8086->8086/tcp   influxdb
```

Log into the InfluxDB bash shell using the container ID
```
$ docker exec -it abf758e0179d /bin/bash
influxdb@abf758e0179d:/$
```

Enter influx (^D to exit)
```
$ influx -username admin -password <ADMIN_PASSWORD>
Connected to http://localhost:8086 version 1.8.3
InfluxDB shell version: 1.8.3
> 
> USE telegraf
> CREATE USER telegraf WITH PASSWORD 'PUT-TELEGRAF-PASSWORD-HERE' WITH ALL PRIVILEGES
> SHOW USERS
user     admin
----     -----
admin    true
telegraf true
> influxdb@67679b5fe9e5:/$ exit
```

### Enable HTTP Authentication in the config file by setting auth-enabled=true
```
$ sudo nano /etc/influxdb/influxdb.conf

[http]
  enabled = true
  bind-address = ":8086"
  auth-enabled = true
```

Restart the container ID
```
$ docker container restart influxdb
```

Ensure that the HTTP API fails now authentication is enabled
```
 $ curl -G http://localhost:8086/query --data-urlencode "q=SHOW DATABASES"
{"error":"unable to parse authentication credentials"}
```

Check that is succeeds when the password is provided
```
 $ curl -G -u admin:<ADMIN-PASSWORD> http://localhost:8086/query --data-urlencode "q=SHOW DATABASES"
{"results":[{"statement_id":0,"series":[{"name":"databases","columns":["name"],"values":[["weather"]]}]}]}
```

## Setup Telegraf

Add Telegraf user
```
$ sudo useradd -rs /bin/false telegraf
```

Create the Telegraf configuration directory and file
```
$ sudo mkdir -p /etc/telegraf
$ docker run --rm telegraf telegraf config | sudo tee /etc/telegraf/telegraf.conf > /dev/null
```

Change permissions on the telegraf directory
```
$ sudo chown telegraf:telegraf /etc/telegraf/*
```

Modify the telegraf configuration file giving the basic authentication for HTTP (use credentials from InfluxDB section above)
```
$ sudo nano /etc/telegraf/telegraf.conf

## HTTP Basic Auth
username = "telegraf"
password = "PUT-TELEGRAF-PASSWORD-HERE"
```

### Run Telegraf Container under Docker

Get the InfluxDB container ID (abf758e0179d) from below
```
$ docker container ls
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                    NAMES
abf758e0179d        influxdb            "/entrypoint.sh -con…"   39 minutes ago      Up 12 minutes       0.0.0.0:8086->8086/tcp   influxdb
```

Get the telegraf user ID (998) from below
```
$ getent passwd | grep telegraf
telegraf:x:998:993::/home/telegraf:/bin/false
```

Run the telegraf image using the InfluxDB container ID and user ID from above
```
$ docker run -d --user 998:998 --name=telegraf --net=container:abf758e0179d -e HOST_PROC=/host/proc -v /proc:/host/proc:ro -v /etc/telegraf/telegraf.conf:/etc/telegraf/telegraf.conf:ro telegraf
```

Ensure that Telegraf is running correctly, view the logs
```
$ docker container logs -f --since 10m telegraf
```

If there are no errors check the correctness by inspecting the InfluxDB database to see that metrics are being stored in the cpu measurement.
```
$ docker exec -it influxdb influx -username telegraf -password PUT-TELEGRAF-PASSWORD-HERE
Connected to http://localhost:8086 version 1.8.0
InfluxDB shell version: 1.8.0
> SHOW DATABASES
name: databases
name
----
_internal
telegraf
> USE telegraf
Using database telegraf
> SELECT * FROM cpu WHERE time > now() - 1m
> SELECT * FROM cpu WHERE time > now() - 1m
name: cpu
time                cpu       host         usage_guest usage_guest_nice usage_idle        usage_iowait        usage_irq usage_nice usage_softirq        usage_steal usage_system        usage_user
----                ---       ----         ----------- ---------------- ----------        ------------        --------- ---------- -------------        ----------- ------------        ----------
1603925210000000000 cpu-total abf758e0179d 0           0                99.29859719438942 0.02505010020040467 0         0          0.025050100200400778 0           0.175350701402806   0.47595190380760866
1603925210000000000 cpu0      abf758e0179d 0           0                99.19839679358789 0.10020040080161206 0         0          0.1002004008016054   0           0.3006012024048184  0.3006012024048184
1603925210000000000 cpu1      abf758e0179d 0           0                99.69909729187363 0                   0         0          0                    0           0                   0.3009027081243679
1603925210000000000 cpu2      abf758e0179d 0           0                99.29929929929993 0                   0         0          0                    0           0.3003003003003025  0.40040040040040037
1603925210000000000 cpu3      abf758e0179d 0           0                98.9979959919826  0                   0         0          0                    0           0.10020040080160089 0.9018036072144258
1603925220000000000 cpu-total abf758e0179d 0           0                98.12077173640691 0.0250563768479029  0         0          0                    0           0.8519168128288677  1.0022550739163207
...
> exit
```

## Setup Grafana

### Install Grafana on Docker

```
$ docker run -d --name=grafana -p 3000:3000 grafana/grafana
```

A Grafana server container should now be up and running on your host. To confirm that, run the following command.
```
$ docker container ls | grep grafana
121a2051ddf4        grafana/grafana     "/run.sh"                40 seconds ago      Up 35 seconds       0.0.0.0:3000->3000/tcp   grafana
```

You can also make sure that it is correctly listening on port 3000.
```
$ netstat -tulpn | grep 3000
tcp6       0      0 :::3000                 :::*                    LISTEN      -
```

### Configure Grafana

Using a web browser head over to [http://IP-ADDRESS-OF-PI:3000]([http://IP-ADDRESS-OF-PI:3000]). You will be directed to a web page where you can enter with the credentials admin/admin. Change the password immediately.

#### Grafana Web UI Configuration

#### Add data source
1. Select InfluxDB
1. In the terminal get the InfluxDB public IP on our bridge network (IPv4)
```
$ docker network inspect bridge | grep influxdb -A 5
"Name": "influxdb",
"EndpointID": "5586ff63a5932287005e9f4112fef85573d9a2c4c640149867463b0ce1b498b3",
"MacAddress": "02:42:ac:11:00:02",
"IPv4Address": "172.17.0.2/16",
"IPv6Address": ""
}

```
iii. Enter the IPv4 address with the port in the URL entry 
```
http://172.17.0.2:8086
```
iv. Under Auth click Basic auth

v. Under Basic AUth Details
   1. User (admin)
   1. Password (PUT-ADMIN-PASSWORD-HERE)

vi. Enter the InfluxDB Details 
   1. Database (telegraf)
   1. User (telegraf)
   1. Password (PUT-TELEGRAF-PASSWORD-HERE)
   1. HTTP method (GET)
   
vii. Click Save & Test

#### Import A Grafana Dashboard

1. Hover over the '+' icon on the top left and select import
   1. Enter 1443 as the Dashboard ID and select LOAD
   1. Select General as a Folder
   1. Select InfluxDB as the InfluxDB source.
   1. Click Import
   1. The Telegraf Host Metrics will be displayed

### Configure additional Telegraf statistics

Edit the /etc/telegraf/telegraf.conf file and make the following changes

1. `sudo nano /etc/telegraf/telegraf.conf`
   1. In `[[inputs.ping]]`
   1. `urls = ["10.0.0.54", "google.com", "orchardrecovery.com", "ox.orchardrecoveryonline.com", ...]`
1. In `[[inputs.net_response]]`
   1. `address = "https://fyidb2.com/orchard/index2.php"`
1. In `[[inputs.file]]`
   1. `files = ["/sys/devices/virtual/thermal/thermal_zone0/temp"]`
   1. `data_format = "value"`
   1. `data_type = "integer"`
   1. `name_override = "pitemp"`

Restart telegraf
```
$ docker container restart telegraf
```

### Create A Grafana Dashboard

1. In the top left of Grafana hover over the + and select Create Dashboard
   1. Select Add new Panel
      1. Under Settings change the Panel Title to Ping Internal
   1. Select Visualization
   1. Select Graph
   1. Select Query
      1. `FROM default ping WHERE url = <HOST-NAME-OR-IP-ADDRESS> SELECT field(average_response_ms) distinct()
   1. Change Alias By to the actual host name
   1. Repeat until all the hosts are added
   
1. Select Add new Panel
   1. Under Settings change the Panel Title to EZDB
1. Select Visualization
1. Select Graph
1. Select Query
   1. SELECT distinct("response_time")  * 1000 FROM "http_response" WHERE ("server" = 'https://fyidb2.com/orchard/index2.php')
1. Change Alias By to EZDB Response Time

1. Select Add new Panel
   1. Under Settings change the Panel Title to Pi Temp
1. Select Visualization
1. Select Graph
1. Select Query
   1. SELECT distinct("value")  / 1000 FROM "pitemp" WHERE ("host" = 'abf758e0179d')
1. Change Alias By to Temperature

1. Select Add new Panel
   1. Under Settings change the Panel Title to Uptime
1. Select Visualization
1. Select Stat
1. Select Query
   1. SELECT distinct("uptime") FROM "system" WHERE ("host" = 'abf758e0179d')
1. Select Display
   1. Select Calculation - Last
   1. Select Graph mode - None
1. Under Field
   1. Choose Unit - duration (d hh:mm:ss)
   1. Remove all but the base threshold

1. Select Add new Panel
   1. Under Settings change the Panel Title to Disk (Used)
1. Select Visualization
1. Select Stat
1. Select Query
   1. SELECT distinct("used_percent") FROM "disk" WHERE ("host" = 'abf758e0179d')
1. Select Display
   1. Select Calculation - Last
   1. Select Graph mode - None
1. Under Field
      1. Choose Unit - Percent (0-100)
      1. Choose Decimals - 2
      1. Under threshold
         1. Add Red threshold of 75
         1. Add Yellow threshold of 50
      
1. Select Add new Panel
   1. Under Settings change the Panel Title to RAM (Free)
1. Select Visualization
1. Select Stat
1. Select Query
   1. SELECT distinct("used_percent") FROM "disk" WHERE ("host" = 'abf758e0179d')
1. Select Display
   1. Select Calculation - Last
   1. Select Graph mode - None
1. Under Field
      1. Choose Unit - Percent (0-100)
      1. Choose Decimals - 2
      1. Under threshold
         1. Add Green threshold of 50
         1. Add Yellow threshold of 25
         1. Change Base to Red

1. Select Add new Panel
   1. Under Settings change the Panel Title to Swap (Used)
1. Select Visualization
1. Select Stat
1. Select Query
   1. SELECT distinct("used_percent") FROM "swap" WHERE ("host" = 'abf758e0179d')
1. Select Display
   1. Select Calculation - Last
   1. Select Graph mode - None
1. Under Field
      1. Choose Unit - Percent (0-100)
      1. Choose Decimals - 2
      1. Under threshold
         1. Add Red threshold of 75
         1. Add Yellow threshold of 50
