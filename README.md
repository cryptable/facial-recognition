Description
===========

![Face Recognition Architecture](./images/Face-Recgnition-Architecture.png)

Setup for facial recognition support in Home Assistant. 
We will setup a VM with the following support servers:

1) MQTT Server
2) Frigate 
3) CompreFace / Deepstack
4) Double Take

We will use the camera of laptop or later the camera of raspberry Pi.

Installation from scratch
-------------------------

See [Installation](./install.md)

Configuring the OVA-file
------------------------

The network adapter must be an "e1000" to see the "ens33" adapter in Linux.

When you have downloaded the OVA-file with the installed packages (rtsp-simple-server, Frigate, CompreFace and Double Take ), then you need to perform only the configuration of IP addresses. You need to change the IP addresses to the IP addresses of your guest OS.

To get the IP address:

```
ip a
```

Search for "ens33" to get its IP address.

Now you can ssh into the guest OS from a terminal:

```
ssh face@<IP address>
```

Password: support.

goto directory ./rtsp-simple-server and change the ip-adressen in the start.sh file. When changed, run:

```
sudo ./start.sh
```

Ssh into the guest from a second terminal and configure frigate:

```
sudo vi /etc/frigate/frigate.yml
```

Change all the IP addresses and start frigate:

```
cd frigate
sudo docker compose up
```

Ssh into the guest from a third terminal to start CompreFace:

```
cd compreface
sudo docker compose up
```

Ssh into the guest from a fourth terminal to start DoubleTake:

```
cd double-take
sudo docker compose up
```

You can now go to Double Take: http://<host ip address>:3000/config to configure it:

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

Don't forget to change the IP addresses to your Guest IP.

Next time you'll need only to start the rtsp-simple-server, because all other processes will be automatically started by docker.


Challenges
----------

- Try the whole process on a Raspberry Pi (need a Pi with minimal 4G memory). 
- Replace CompreFace with DeepStack 
- Instead of Facial recognition, try number plate recognition.

Notes
-----

- https://www.gyan.dev/ffmpeg/builds/
- https://gist.github.com/edjdavid/513cbee3f9a10cd06e9e49e8bdfa0f96

```
./ffmpeg -f dshow -rtbufsize 1024M -framerate 30 -video_size 640x480 -vcodec mjpeg -i video="Integrated Camera" -f rtsp -rtsp_transport tcp rtsp://192.168.80.128:8554/stream
./ffmpeg -f dshow -rtbufsize 1024M -framerate 30 -video_size 640x480 -vcodec mjpeg -i video="UC CAM75FS-1" -f rtsp -rtsp_transport tcp rtsp://192.168.80.128:8554/stream
```
