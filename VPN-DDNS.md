Set up LXC container with default settings besides Hard Disk size set to 6GiB and Network settings set to DHCP.
Main node's shell to edit lxc config with 'nano /etc/pve/lxc/101.conf' and added the lines
' lxc.cgroup2.devices.allow: c 10:200 rwm
 lxc.mount.entry: /dev/net dev/net none bind,create=dir'
 followed from: https://pve.proxmox.com/wiki/OpenVPN_in_LXC

then did this 'chown 100000:100000 /dev/net/tun'


 from: https://github.com/Nyr/wireguard-install
 ran the script 'wget https://git.io/wireguard -O wireguard-install.sh && bash wireguard-install.sh'
 Followed default options, for DNS resolve I did Google

 
 
