

Install motion
--------------

```
sudo apt-get install motion
```

Auto configure motion system:

```
sudo mkdir /var/log/motion
sudo touch /var/log/motion/motion.log
sudo chown -R /var/log/motion
sudo systemctl enable motion
sudo systemctl start motion
```

In the /etc/motion/motion.

- https://motion-project.github.io/3.4.1/motion_guide.html
- https://wiki.gentoo.org/wiki/Motion
