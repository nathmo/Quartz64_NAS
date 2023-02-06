# Quartz64_NAS
repo to repurpose a Quartz64 SBC as a home server / NAS using docker and zfs on plebian OS

# What you get
this document will walk you trough the setup of a NAS using the quartz64. it will allow you to self host
a lot of service using docker compose file.
I'm currently hosting on mine : 
- zoneminder (surveillance camera recordind system)
- paperless-ngx (managed all your digital document like bill, official stuff, certificate and more)
- jellyfin (netflix like interface for your video)
- synchting ( onedrive like system that allow you to sync your file between your computers. its free and you keep your data on your device)
- piwigo (photo gallerie for family)
- radicale ( calendar server, allow me to sync my event betweens my phone and computers)

and the following service that are not installed using docker compose and are really hand to make your server more user friendly
- cockpit (a web interface to monitor the status of your server)
- portainer ( web interface for deploying docker compose file)
- ddclient (script that tell your domaine name registrar your public ip when it change if you dont have a static public IP) (useful if you want to access your network from outside using a VPN)
- pihole (DNS server that block ad from loading on your network)
- wireguard ( VPN server that allow you to remotely connect to your home network)
- log2ram ( reduce wear of the SD card / emmc)

# Hardware requirement
- Quartz64 A (8Gb RAM) (https://www.pine64.org/quartz64a/)
- emmc for the Quartz 64 (I have trust issue with SD card) (https://pine64.com/product/32gb-emmc-module/)
- heatsink for the Quartz Board
- a fan with a molex connector (or two)
- SD card (a least for copying the system to the emmc)
- standard computer PSU 
- PCIe to sata extension board (https://fr.aliexpress.com/item/1005003760757628.html)
- DC barrel jack to screw terminal (to power the Quartz 64) (https://fr.aliexpress.com/item/1005004281570915.html)
- SD card extension connector (not mandatory but handy) (https://fr.aliexpress.com/item/1005004149375152.html)
- a case ( I laser cutted one in wood)
- sata cable
- sata Hard drive
- MOLEX cable stub (https://fr.aliexpress.com/item/1005004798587038.html)

the total for the hardware is in the 200-1000 $ range ( the drive are expensive. if you have them aldready you can stay bellow the 250 $)

cut one of the molex stub yellow and black cable ( 12V = yellow, 0V = black)
and strip the insulation then screw inside the terminal.(double check the polarity if you dont want to destroy your board)


![image](https://user-images.githubusercontent.com/15912256/217092063-86f65d51-33b1-45c6-a2d3-39b7dbd8e005.png)

# Setup
follow the step carefully by coping each command in the terminal.

### assemble the case
not much to say. wire the things thogeter in a nice enclosure

### Installing the OS
you need an SD card and to download the following image from plebian : https://github.com/Plebian-Linux/quartz64-images/releases
(download the one ending with -quartz64a.img.xz)
armbian is not mature enough for this board and the peoples at plebian did an amazing job with thoeses images so i recommend using them instead of armbain 
on linux you can run this from the folder where you downloaded the image (change sdX by the drive shown with lsblk)
```
sudo apt install gunzip
gunzip XXXXX-quartz64a.img.gz
lsblk
sudo dd of=/dev/sdX if=XXXXX-quartz64a.img bs=1M status=progress
```
replace the X with the correcte drive and the XXXXX with the correct file that you downloaded


on windows you can just use 7zip to extract the archive and rufus to burn it to the SD card : [RUFUS](https://rufus.ie/fr/)

once it's done you can insert the SD card in the board and press the start button.
I assume your board is connect over ethernet and you will connect to it remotely using SSH. but you can also hook up a keyboard and an HDMI screen (beware of the US default keyboard layout)
all the following command will be run on the Quartz Board either using the keyboard or via SSH.

(to find the IP address, look in the managent page of your router or use zenmap to scan your lan)
on ubuntu :
```
ssh pleb@192.168.YYY.XXX
```
on windows you can use putty

the default password is `pleb`
it will ask you to change the password. just do it.

### RAM hardware TEST
this step is recommended to ensure there is no factory default in the RAM of your board that will cause crash and instabilitys. if you have error, send the board back for free (1 month after you buy max)

```
sudo apt-get install memtester
```

```
sudo memtester 7500 2
```
it's going to take a few minutes.
![image](https://user-images.githubusercontent.com/15912256/217096238-9a62be53-abc2-4923-bada-79fb4fce838e.png)

### copying to the emmc
if you dont have an emmc you can skip this step.

```wget https://github.com/Plebian-Linux/quartz64-images/releases/download/v2023-01-21-1/plebian-debian-bookworm-quartz64a.img.xz```
```sudo apt-get install xz-utils```
```unxz plebian-debian-bookworm-quartz64a.img.xz```
```lsblk```
```sudo dd of=/dev/mmcblk1 if=plebian-debian-bookworm-quartz64a.img  bs=1M status=progress```
ensure mmcblk1 is your EMMC ! 
change the image filelane as you want to use the lasted from the link (https://github.com/Plebian-Linux/quartz64-images/releases)(right click on the quartz64a version -> copy the link)
once its done. copying you can remove the SD card and reboot the board. its now booting from the emmc.

### set a static IP
but first : update
```
sudo  apt install nano htop tmux wget curl
sudo apt update
sudo apt upgrade
```
then edit this file : 
```sudo nano /etc/network/interfaces```
like that
```
auto end0
iface end0 inet static
 address 192.168.1.10
 netmask 255.255.255.0
 gateway 192.168.1.1
```
change the subnet and the IP so that it match your network.

```sudo unlink /etc/resolv.conf```
```sudo nano /etc/resolv.conf```
and add that in the file : (it's google and cloudflare DNS server)
(you can also the one provided by your ISP)
```
nameserver 1.1.1.1
nameserver 8.8.8.8
```
now restart the board. you will be able to reconnect via SSH on the choosen IP address (wait 1-2 min for the board to reboot)

```
sudo reboot now
```

### user and SSH setup
I recommend adding your public sh key so taht you dont need a password to login :
```
mkdir .ssh
nano .ssh/authorized_keys
```
paste the content of .ssh/id_rsa.pub of your machine to the authorized file there (from your linux machine. you can find out how to do it with putty on google if you want)

### log2ram setup
run the followings commands : 
```
echo "deb [signed-by=/usr/share/keyrings/azlux-archive-keyring.gpg] http://packages.azlux.fr/debian/ bullseye main" | sudo tee /etc/apt/sources.list.d/azlux.list
sudo wget -O /usr/share/keyrings/azlux-archive-keyring.gpg  https://azlux.fr/repo.gpg
sudo apt update
sudo apt install log2ram
```
then edit the config to ensure you will have enouh space for your log : 
```
sudo nano /etc/log2ram.conf
```
100-200 mb should be plenty
```
SIZE=200M
```

### instal cockpit
```
sudo su
apt install cockpit cockpit-pcp
```

you should now be able to open a browser tab and navigate to https://192.168.XXX.YYY:9090/metrics
and login with pleb and your password.
![image](https://user-images.githubusercontent.com/15912256/194674108-69b36efa-8e62-4a1b-9cae-8dffc15cb7f5.png)
(here is the cockpit interface)

### install ddclient
you might want to setup a dynDNS if you own a domain from some company like OVH and you dont have a static IP.
what it does is basically tell the OVH DNS server that your domain should point to this IP address. this IP address being the one your ISP gave you.
and if it change the script update it automatically.

```
sudo apt-get install ddclient
sudo apt install libio-socket-ssl-perl
```
then you can enter the following value given by your registrat (OVH for me)
![image](https://user-images.githubusercontent.com/15912256/197188539-fc47749d-6dee-4c07-af22-b387624b7fdb.png)

```
Dynamic DNS service provider:other
Dynamic DNS update protocol : dyndns2
Dynamic DNS server : www.ovh.com
Username for dynamic DNS service: OVH dynDNS username
Password for dynamic DNS service: OVH dynDNS passowrd
Re-enter password to verify: OVH dynDNS password (again)
online method to get your IP
```
you are all set. your domain name should point to your home IP address.

### install pihole and pivpn

```
curl -sSL https://install.pi-hole.net | sudo PIHOLE_SKIP_OS_CHECK=true bash
```
press next and it should work all right.

```
curl -L https://install.pivpn.io > install.sh
chmod +x install.sh
./install.sh
```
choose what you want. i personnaly went with wireguard on the default port and pihole as the DNS server.
then reboot. you can add configuration with

```
pivpn -a
```
give it a name. you can either use 
```
pivpn -qr
```
to get a QR code that you can scan or you can 
```
cat /home/ubuntu/configs/*.conf
```
and you can import that in a text file on your phone or computer and import it into wireguard vpn client.

### install ZFS

```
sudo su
sudo apt install zfs-dkms zfsutils-linux
sudo reboot now
```
you need to relogin (and if you imported your SSH public key, you dont need a password anymore)
```
ssh pleb@192.168.YYY.XXX
```
### create your ZFS pool

```
sudo fdisk -l
sudo zpool create -m /pool storagearray raidz2 /dev/sda /dev/sdb /dev/sdc /dev/sdd /dev/sde /dev/sdf
sudo zpool status
```
your pool is automatically mounted under /pool but you can change the location.
raidz2 use two drive for parity bit. you can use use raid1 for only one parity bit
(but only one drive can fail before you risk loosing data)

if you aldready have a ZFS pool from a previous install you can import it with 
```
sudo zpool import storagearray -f
```

### Install docker
docker allow you to run container, portainer is a nice web page to manage your portainer. its more user friendly than the shell at first
here we create a volume on the ZFS array for portainer and then launche the portainer container

```
sudo apt install docker-compose
sudo systemctl stop docker
mkdir /pool/image
sudo mv /var/lib/docker /pool/image 
sudo chown -R pleb:pleb /pool/image/
sudo chmod -R 755 /pool/image/
sudo ln -s /pool/image /var/lib/docker
sudo systemctl start docker
```
we installed docker and changed its root to your ZFS pool so that you dont fill up all your SD / emmc with docker image

```
sudo su pleb
cd /pool
mkdir container
cd container
mkdir portainer
curl -sSL https://get.docker.com | sh
sudo usermod -aG docker pleb
systemctl status docker
sudo docker run -d -p 8000:8000 -p 9000:9000 -p 9443:9443 --name=portainer --restart=always -v /var/run/docker.sock:/var/run/docker.sock -v /pool/container/portainer:/data portainer/portainer-ce:latest
```
the nice part of this approach is that you have all your docker compose on portainer. and portainer is stored on the zfs volume that you can reimport even if you change the host board or need to reinstall the OS.

you can now access portainer : http://192.168.1.10:9000
![image](https://user-images.githubusercontent.com/15912256/217105764-a136034a-cefb-4481-abe6-8ec7df3730ff.png)

### Deploying dockerfile
create a folder under 
```
cd /pool/container/
mkdir yourContainerVolume
```

click on add stack :
![image](https://user-images.githubusercontent.com/15912256/217105796-4a1c665f-8cf4-402a-b34c-6fc86d741669.png)
then give it a name, copy the compose file in the web edditor and deploy
![image](https://user-images.githubusercontent.com/15912256/217105881-3fbc3012-a214-478e-9f18-f6e63ae96227.png)

you can for instance deploy a synchting stack : 

![image](https://user-images.githubusercontent.com/15912256/217106401-82d4000a-1125-4a7e-91a4-08082b6c51fc.png)

```
cd /pool/container/
mkdir syncthing
cd ..
mkdir NAS
```

```
version: "2.1"
services:
  syncthing:
    image: lscr.io/linuxserver/syncthing:latest
    container_name: syncthing
    hostname: syncthing #optional
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Europe/London
    volumes:
      - /pool/container/syncthing/config:/config
      - /pool/NAS:/NAS
    ports:
      - 8384:8384
      - 22000:22000/tcp
      - 22000:22000/udp
      - 21027:21027/udp
    restart: unless-stopped
```

and thats all you can now access your synthing instance
http://192.168.1.10:8384/#

![image](https://user-images.githubusercontent.com/15912256/217106650-552952a5-e9e4-4e48-ada7-1af6f5b6038d.png)

### Recap
you have now a solid NAS setup where you can add more service using a simple dokcer compose file.

- pihole : http://192.168.1.10/admin/
- cockpit : http://192.168.1.10:9090/#
- portainer : http://192.168.1.10:9000/#
- syncthing : http://192.168.1.10:8384/#
