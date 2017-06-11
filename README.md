# docker-ccu2
Script to create a docker container with the CCU2 firmware on the raspberry

This script will download the CCU2 firmware for the Homematic, re-package it as docker image and start it on your raspi. Other ARM-based boards might also work (see Depencies section).

If you want to skip the build and just download the ready to user docker container please visit [this](https://hub.docker.com/r/angelnu/ccu2/).

## Dependencies

* ARM HW. Following combinations tested:
  * raspberry (tested on raspberry 3) and raspberrian jessy
  * Orange Pi Plus 2 and Armbian (Ubuntu 16.04 and Kernel 4.9) - Self built
* no programs using ports 80 and 2001

## How to build&install
1. ssh into your ARM device
2. 'sudo -i'
3. git clone this repository
4. (Optional) Edit the default settings in _build.sh_
5. execute _build.sh_
6. (optional) Delete the cloned repo: the docker image and its data are stored in _/var/docker_

After the above steps you can connect to <IP address of your raspberry>. The CCU2 docker image will be started automatically on every boot of the raspberry: the container is started in auto-restart mode.

## How to update to a new CCU2 firmware
Just update the CCU2_VERSION field in _build.sh_ and execute it again. Your CCU2 settings will be preserved.

## How to configure local antena
If you have added a [homematic radio module](http://www.elv.de/homematic-funkmodul-fuer-raspberry-pi-bausatz.html) to your raspberry, then you can use it with your CCU2 docker image. If not, you would need to use the external LAN gateway. You can also use a combination of both.

### Intructions
1. (raspberry 3) You need to avoid that the bluetoth module uses the HW UART. You can either disable it or let it use the miniUART. I use the second option but since I do not use Bluetoth on the Raspi I do not know if this breaks it. More info [here](http://raspberrypi.stackexchange.com/questions/45570/how-do-i-make-serial-work-on-the-raspberry-pi3).
   * Add `dtoverlay=pi3-miniuart-bt`to _/boot/config.txt_
2. (raspberry) Make sure that the serial console is not using the UART
   * Replace `console=ttyAMA0,115200` with `console=tty1` in _/boot/cmdline.txt_
   * More info [here](http://raspberrypihobbyist.blogspot.de/2012/08/raspberry-pi-serial-port.html)
4. _docker restart ccu2_

## How to import settings from an existing CCU2
1. Enable ssh in your CCU2. Instructions (in German) [here](https://www.homematic-inside.de/tecbase/homematic/generell/item/zugriff-auf-das-dateisystem-der-ccu-2)
2. ssh into your rapberry
3. `sudo -i`
4. `service ccu2 stop`
5. Copy the old rfd.conf: `cp _/var/lib/docker/volumes/ccu2_data/_data/etc/config/rfd.conf{,org}_
6. `rsync -av <your CCU2 IP>/usr/local/*  /var/lib/docker/volumes/ccu2_data/_data/`
7. Diff the orginal rfd.conf with the one you copied from the CCU2: `diff -u _/var/lib/docker/volumes/ccu2_data/_data/etc/config/rfd.conf{,org}_
8. Make sure you keep the original lines for:
   * `#Improved Coprocessor Initialization = true` - commented out
   * ` AccessFile = /dev/null` - notice the blank at the start of the line
   * ` ResetFile = /dev/ccu2-ic200` - notice the blank at the start of the line
9. `docker start ccu2`

## Swarm
You can also deploy this docker image to a docker swarm. For this you need to:
* set up a docker swarm in multiple Raspberries. They all need to have local antennas at least you connect to a remote antenna
* have a local repository to upload the image to
* have a shared folder mounted at the same location in all the members of the cluster. Examples
  * Mount a NAS folder. This is simple but then the NAS is the single point of failure
  * Cluster FS such as glusterfs. TBD: upload instructions

execute the build with additional variables:
'CCU2_REGA_PORT=8080 DOCKER_CCU2_DATA=/media/glusterfs/ccu2 DOCKER_MODE=swarm DOCKER_OPTIONS="--constraint node.labels.architecture==arm" ./build.sh'

* DOCKER_CCU2_DATA should point to a mounted folder that it is shared between the cluster members
* DOCKER_MODE=swarm to deploy to a swarm instead of a singe docker node (default)
* DOCKER_ID has to point to the local data repository
* DOCKER_OPTIONS pass additional flags such as constraints so the docker image is only deployed on valid nodes
