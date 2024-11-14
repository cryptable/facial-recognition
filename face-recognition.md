Face Recognition
================

Install Debian
--------------

Simple minimal install without desktop

VMWare image USB settings
-------------------------

In the USB-hub settings of VMware you need to set it as *USB 3.1* and enable *Show all USB input devices*.

Install sudo
------------

```
su
apt-get install sudo
/sbin/usermod -aG sudo face
exit
exit
```

Install docker
--------------

Remove docker

```
for pkg in docker.io docker-doc docker-compose podman-docker containerd runc; do sudo apt-get remove $pkg; done
sudo apt autoremove
```

Install the docker resources

```
# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
```

Install docker

```
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

```
sudo groupadd docker
sudo usermod -aG docker $USER
```

Install mediamtx
----------------

This turns your webcam into a rtsp-based IP camera

```
sudo apt-get install v4l-utils
sudo apt install ffmpeg
wget https://github.com/bluenviron/mediamtx/releases/download/v1.3.1/mediamtx_v1.3.1_linux_amd64.tar.gz
tar xvzf mediamtx_v1.3.1_linux_amd64.tar.gz
mediamtx &
sudo ffmpeg -f v4l2 -framerate 30 -video_size 640x480 -i /dev/video0 -f rtsp rtsp://192.168.80.131:8554/mystream
```

You need to switch off the rtmp-service of the server. Search for the *rtmpDisable* in the rtsp-simple-server.yml and replace it with:

```
rtmpDisable: yes
```

Install Frigate
---------------

```
sudo mkdir -p /etc/frigate
sudo mkdir -p /var/frigate/storage
```

```
mkdir frigate
cd frigate
```

```
vi docker-compose.yml
```

```
version: "3.9"
services:
  frigate:
    container_name: frigate
    privileged: true # this may not be necessary for all setups
    restart: unless-stopped
    image: blakeblackshear/frigate:0.11.1
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

Install CompreFace
------------------

```
sudo apt install unzip
cd
mkdir compreface
cd compreface
```

Copy the zip file to your vm.

```
wget -q -O tmp.zip 'https://github.com/exadel-inc/CompreFace/releases/download/v1.2.0/CompreFace_1.2.0.zip' && unzip tmp.zip && rm tmp.zip
```

```
sudo docker compose up
```

1. Login into CompreFace website on location: http://192.168.50.179:8000
2. Create an account (username: face.recognition@test.com, password: support33)
3. Create an application. When you enter the application you'll find an Application Key to use in Double Take

