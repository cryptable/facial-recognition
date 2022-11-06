Description
===========

!(Face Recognition Architecture)[./Face-Recognition-Architecture.png]

Setup for facial recognition support in Home Assistant. 
We will setup a VM with the following support servers:

1) MQTT Server
2) Frigate 
3) CompreFace / Deepstack
4) Double Take

We will use the camera of laptop or later the camera of raspberry Pi.

Install a Ubuntu Server
-----------------------

Username: face
Password: support33

Install a standard Ubuntu Server with your keyboard support needed.

Login into the server using VMware and get its IP address:

```
ip a
```

When rebooted you need first to update your server. Open a terminal to the server using ssh and upgrade your server.

```
sudo apt update
sudo apt upgrade
```

Install Mosquitto
------------------

### Server

We install the MQTT Server.

```
sudo apt install -y mosquitto
```

Check if it is running

```
sudo systemctl status mosquitto
```

Which return somthing like

```
face@face-recognition-support:~$ sudo systemctl status mosquitto
[sudo] password for face: 
● mosquitto.service - Mosquitto MQTT Broker
     Loaded: loaded (/lib/systemd/system/mosquitto.service; enabled; vendor preset: enabled)
     Active: active (running) since Sun 2022-11-06 09:32:24 UTC; 2min 56s ago
       Docs: man:mosquitto.conf(5)
             man:mosquitto(8)
    Process: 15494 ExecStartPre=/bin/mkdir -m 740 -p /var/log/mosquitto (code=exited, status=0/SUCCESS)
    Process: 15497 ExecStartPre=/bin/chown mosquitto /var/log/mosquitto (code=exited, status=0/SUCCESS)
    Process: 15506 ExecStartPre=/bin/mkdir -m 740 -p /run/mosquitto (code=exited, status=0/SUCCESS)
    Process: 15508 ExecStartPre=/bin/chown mosquitto /run/mosquitto (code=exited, status=0/SUCCESS)
   Main PID: 15512 (mosquitto)
      Tasks: 1 (limit: 4534)
     Memory: 1.2M
        CPU: 50ms
     CGroup: /system.slice/mosquitto.service
             └─15512 /usr/sbin/mosquitto -c /etc/mosquitto/mosquitto.conf

Nov 06 09:32:24 face-recognition-support systemd[1]: Starting Mosquitto MQTT Broker...
Nov 06 09:32:24 face-recognition-support systemd[1]: Started Mosquitto MQTT Broker.
```

When you need to change some settings on mosquitto, we need to restart or stop and start the MQTT server.

- stop

```
sudo systemctl stop mosquitto
```

- start

```
sudo systemctl start mosquitto
```

- restart

```
sudo systemctl restart mosquitto
```

We want remote access so we need to add something to mosquitto.conf file, such that HASS can access it.

```
sudo vi /etc/mosquitto/mosquitto.conf
```

Add the followin to the configuration file:

```
listener 1883 0.0.0.0
allow_anonymous true
```

Restart the server.

### Mosquitto Client

To test the MQTT server, you need a clients. So we install the client software:

```
sudo apt install -y mosquitto-clients
```

Here you can subscribe to topics, which means every time a message is published on this topic the subscriber will receive that message.

```
mosquitto_sub -t "home/lights/sitting_room"
```

Open a second terminal to the Ubuntu Server and the test the MQTT server

```
mosquitto_pub -m "ON" -t "home/lights/sitting_room"
```

You should see 'ON' appearing in the first terminal.

Test it also from a remote location.

```
mosquitto_pub -h <IP address of MQTT server> -m "ON" -t "home/lights/sitting_room"
```

If everything works, goto next step and install Frigate on it

### Reference

- https://www.vultr.com/docs/install-mosquitto-mqtt-broker-on-ubuntu-20-04-server/
- https://community.home-assistant.io/t/mosquitto-allow-local-network-access/284833


Install rtsp-server
-------------------


```
sudo apt install ffmpeg
      sudo apt install v4l-utils
```

```
mkdir rtsp-simple-server
cd rtsp-simple-server
```

scp the downloaded file into the VM.

startup.sh script

```
#/bin/bash

export rtspServer=192.168.50.178:rtsp://192.168.50.178:8554/stream
./rtsp-simple-server &
sudo ffmpeg -f v4l2 -framerate 24 -video_size 480x480 -i /dev/video0 -f rtsp -vcodec h264 -rtsp_transport tcp rtsp://192.168.50.178:8554/stream
```

Make the startup script executable.

You need to switch of thr rtmp server support in the configuration file (yaml).

You need to switch off the rtmp-service of the server. Search for the *rtmpDisable* in the rtsp-simple-server.yml and replace it with:

```
rtmpDisable: yes
```

Otherwise the Frigate server will not start its RTMP Service.

To start the server:

```
sudo ./startup.sh
```



- https://stackoverflow.com/questions/33800086/using-ffmpeg-to-generate-rtsp-from-webcam
- https://github.com/aler9/rtsp-simple-server#installation
- https://github.com/blakeblackshear/frigate/issues/1184


Install Frigate
---------------

To use Frigate, you will need to install docker support on your Ubuntu Server (or Raspberry Pi).

### Install docker support

See the docker site to understand the following commands, which are run. At the end it just adds the Docker repo to the Ubuntu repositories and installs docker.

```
sudo apt-get install ca-certificates curl gnupg lsb-release
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-compose-plugin
```

To allow non privileged users to use Docker:

```
sudo groupadd docker
sudo usermod -aG docker $USER
```
A *docker* group is created and you add yourself (or current logged in user) to that group.

Test your docker installation.

```
sudo docker run hello-world
```

### Start the Frigate system using docker compose

Prepare your environment to run frigate.

```
sudo mkdir -p /etc/frigate
sudo mkdir -p /var/frigate/storage
```

```
mkdir frigate
cd frigate
```

Create the docker compose YAML file using vi:

```
vi docker-compose.yml
```

Copy following text into the YAML file.

```
version: "3.9"
services:
  frigate:
    container_name: frigate
    privileged: true # this may not be necessary for all setups
    restart: unless-stopped
    image: blakeblackshear/frigate:stable
    shm_size: "64mb" # update for your cameras based on calculation above
    devices:
      - /dev/bus/usb:/dev/bus/usb # passes the USB Coral, needs to be modified for other versions
      - /dev/apex_0:/dev/apex_0 # passes a PCIe Coral, follow driver instructions here https://coral.ai/docs/m2/get-started/#2a-on-linux
      - /dev/dri/renderD128 # for intel hwaccel, needs to be updated for your hardware
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /etc/frigate/frigate.yml:/config/config.yml:ro
      - /var/frigate/storage:/media/frigate
      - type: tmpfs # Optional: 1GB of memory, reduces SSD/SD Card wear
        target: /tmp/cache
        tmpfs:
          size: 1000000000
    ports:
      - "5000:5000"
      - "1935:1935" # RTMP feeds
    environment:
      FRIGATE_RTSP_PASSWORD: "password"
```

Create a minimal configuration file:

```
sudo vi /etc/frigate/frigate.yml
```

```
mqtt:
  host: 192.168.50.178
  port: 1883
cameras:
  back:
    ffmpeg:
      inputs:
        - path: rtsp://192.168.50.178:8554/stream
          roles:
            - detect
            - rtmp
      hwaccel_args: ''
    detect:
      width: 1280
      height: 720

```

Starting Frigate:

```
sudo docker compose up
```

### Reference

- https://docs.frigate.video/installation/
- https://www.reddit.com/r/homeassistant/comments/u20cx3/how_to_use_a_webcam_on_frigate/


Install CompreFace
------------------

Download the latest release: https://github.com/exadel-inc/CompreFace/releases

You have to create a directory for compreface:

```
cd
mkdir compreface
cd compreface
```

Copy the zip file to your vm.

```
sudo apt install unzip
unzip 
```

Start CompreFace with:

```
sudo docker compose up
```

1. Login into CompreFace website on location: http://192.168.50.179:8000
2. Create an account (username: face.recognition@test.com, password: support33)
3. Create an application. When you enter the application you'll find an Application Key to use in Double Take

### Reference

- https://github.com/exadel-inc/CompreFace
- https://github.com/exadel-inc/CompreFace#getting-started-with-compreface


Install Double Take
-------------------

Nothing needs to be installed, because it uses a docker container.

Create a directory in home folder and create a docker-compose.yml file to start up Double Take.

```
cd
mkdir double-take
cd double-take
```

Use vi to create the docker-compose.yml file:

```
vi docker-compose.yml
```

And enter following text for your Double Take installation:

```
version: '3.7'

volumes:
  double-take:

services:
  double-take:
    container_name: double-take
    image: jakowenko/double-take
    restart: unless-stopped
    volumes:
      - double-take:/.storage
    ports:
      - 3000:3000
```


You need configure the Double Take with:
- the MQTT system settings
- frigate server settings
- CompreFace settings

http://<IP address>:3000/config

And enter following settings:


```
mqtt:
  host: 192.168.50.178

frigate:
  url: http://192.168.50.178:5000

detectors:
  compreface:
    url: http://192.168.50.178:8000
    key: <API-key from CompreFace>
```

### References

- https://www.youtube.com/watch?v=_61-hIL1AjQ
- https://hub.docker.com/r/jakowenko/double-take

Challenges
----------

- Try the whole process on a Raspberry Pi.
- Replace CompreFace with DeepStack
- Instead of Facial recognition, try number plate recognition.