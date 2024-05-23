# HomeLab/DIY_Dropbox-Project

## Table of Contents
- [Hardware Setup](#hardware-setup)
- [Down To The Beans](#down-to-the-beans)
  - [CT Templates](#ct-templates)
  - [Structure](#structure)


## Hardware Setup
- HP Microserver Gen 8
- 4x Seagate ST3000NM0035 3TB 7.2k 12Gb/s 3.5” SAS Hard Drive
- LSI HBA SAS 9207-8i
### Additional Things
- [Ventoy](https://sourceforge.net/projects/ventoy/) (Optional)
- [Proxmox iso](https://www.proxmox.com/en/downloads/proxmox-virtual-environment/iso)
- [iLO liscense](https://support.hpe.com/hpesc/public/docDisplay?docId=sd00001039en_us&docLocale=en_US&page=GUID-821C2C6D-BCBF-408B-93F9-641EFFB16E29.html) (Optional)

The server that I am using has something called HP iLO which basically allows me to not have to connect a monitor to the server and be able to get a 'video output' through a web interface. With iLO, I could access the bios of the server and set configurations from a browser. 
I connected the SAS backplane from the server into the LSI raid card and used it in HBA mode to have individual access to each drive and to also do a ZFS RAIDZ setup. Raid card was also advertized to be used 'for zfs freeNAS' lol.
I was also introduced to which allows for multiple iso's to be on a single flash drive. But I really just needed to have the Proxmox iso on a flash drive and install it on the server.

## Down To The Beans
Used default setup settings for Proxmox, and named my root node `beans` cause it's funny.

### CT Templates
To get started with making containers, templates were an easy way to get things started.
- `pveam available` lists out all the available templates
- `pveam update` updates templates duh
Templates can then be eassily downloaded in `local` storage.

### Structure
`beans` is composed of 2 LXC containers and 1 VM
- [CT 101 (vpn-ddns)](#vpn-and-ddns-service)
- [CT 103 (docker)](#ct-103-(docker))
- [VM 100 (TrueNAS-scale)](#VM-100-(TrueNAS-scale))

  ### VPN and DDNS service
  Purpose of a VPN is to have access to the Proxmox server when not at home, and DDNS is to make sure my ip address...
  
  I made a default LXC container with a template of ubuntu 20.04 - cause I couldn't find a template of 22.04 - and then the usual `sudo apt update`, `sudo apt upgrade`, and then `sudo apt dist-upgrade`
  After that, I installed wireguard using `wget https://git.io/wireguard -O wireguard-install.sh && bash wireguard-install.sh` from [here](https://github.com/Nyr/wireguard-install)
  I got the qr code and stuff on my phone and exported it to a file and now I have a profie I can import anywhere to use on whatever device.

## CT 103 (docker)
Used the [script](https://helper-scripts.com/scripts?id=Docker) `bash -c "$(wget -qLO - https://github.com/tteck/Proxmox/raw/main/ct/docker.sh)"` in node shell to install an instance of Docker in a proxmox LXC. 
Notable setup options was running in `priviliged` mode to allow mounting network drives.
Once the Docker instance was set up, I used the preinstalled container of Portainer which is a web interface to then set up the rest of my containers. 
- Cloudflared: Used the preset to install. All I needed to do was enter my tunnel token and it establishes a connection once the container is running.
- File Browser: Modified a docker-compose script from [here](https://hub.docker.com/r/hurlenko/filebrowser) to allow for a network storage mount. Since I was using Portainer as an interface and had my own docker-compose file, I used the stack feature thing to make a custom container and deployed it through there.

This is the `docker-compse.yml` I used to set up File Browser up with a NAS which I got from [here](https://www.youtube.com/watch?v=UfKUtllcmxI&t=359s)
```
version: "3"
services:
  filebrowser:
    image: hurlenko/filebrowser
    user: "${UID}:${GID}"
    ports:
      - 8080:8080
    volumes:
      - truenas:/data
      - ./config:/config
    environment:
      - FB_BASEURL=/filebrowser
    restart: always

volumes:
  truenas:
  driver_opts:
    type: cifs
    device: //192.168.1.161/data
    o: "username=<USERNAME>,password=<PASSWORD>,vers=3.0"
```
## VM 100 (TrueNAS-scale)
Created a VM with 8Gb of ram and 2 and 4 CPUs. I needed to configue the VM to use the connected hardrives, so from the node shell I ran `nano /etc/pve/qemu-server/100.conf`, 100 being the ID of the VM, and then added the lines:
`
scsi1: /dev/disk/by-id/scsi-35000c500a6517c77
scsi2: /dev/disk/by-id/scsi-35000c500a65354ef
scsi3: /dev/disk/by-id/scsi-35000c500a6536c2b
scsi4: /dev/disk/by-id/scsi-35000c500a655c1d3
`
ID for the drives were found by doing `ls -l /dev/disk/by-id/`
For stoarage I created a pool with the 4 hardrives and did a RAIDZ3 setup. 
Created the dataset `tank` to be the main or the root dataset. Subdatasets were:
- `data`: which holds all the user data
- `proxmox-backup`: self-explanitory

Made the credentials:
- user: For File Browser to be able to login and mount the network storage
- server: to link a network storage to Proxmox and use it as a backup dump

