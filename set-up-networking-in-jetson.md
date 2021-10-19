# Set up Networking in Jetson Board

Created by Jack Liang

>Preface: The file `/etc/network/interfaces.d/eth0` in Jetson board will be referred to as eth0 interface file. The guide also pertains specifically to overcoming the internet blockade problem in China.

There are three ways of giving internet to the Jetson board: 
## Direct Ethernet by Cable
Connect the ends of an ethernet cable to an ethernet port and a Jetson board respectively. Then set the eth0 interface file: 
```
auto eth0
iface eth0 inet dhcp
```
`sudo /etc/init.d/networking restart` or restart Jetson board by power then do `ifconfig` to see the DHCP-assigned eth0 inet IP (do `sudo dhclient -r eth0 && sudo dhclient eth0` if no “eth0” shows up). 
In Mac, one should be able to ping that IP and SSH into it by ssh user@x.x.x.x/password.

I encountered many download errors due to no connection to download URL with this approach. This can be amended by installing the likes of Shadowsocks and V2Ray, setting up Dockerhub China mirror, https://www.serverlab.ca/tutorials/containers/docker/how-to-set-the-proxy-for-docker-on-ubuntu/, along with apt and K8s related mirrors. I found this approach tedious and finicky and didn’t worth my while, maybe for you it’d be different. 

>If the Jetson board eth0 IP is not ping-able and the Mac is connected to an adaptor/dongle (like for external monitor), do `ifconfig` and see if you have an “bridge100” or the like and do `sudo ifconfig bridge100 down`. This happens when your Mac has previously connected to devices (like a Jetson board) via an ethernet cable hooked up to an adaptor/dongle with ethernet port (see below) and the Jetson board eth0 inet IP is in the same subnet as the bridge100 inet IP.
 
## DHCP from Mac WIFI Sharing
Connect the ends of an ethernet cable to a Mac (via adaptor/dongle) and a Jetson board respectively. Then in Mac’s “Preferences” → “Network”, choose the adaptor/dongle settings in the left pane labeled in the likes of “USB10/100/100LAN“, set the “Configure IPv4” to “Using DHCP” and let it auto configure then save. Then in “Preferences” → “Sharing” → enable “Internet Sharing” (left pane) and share WIFI with the likes of “USB10/100/100LAN“. Then in the Jetson board, repeat the same step as above and set the eth0 interface file: 
```
auto eth0
iface eth0 inet dhcp
```
then `sudo /etc/init.d/networking restart` or restart Jetson board by power then do `ifconfig` to see the DHCP-assigned eth0 inet IP (do sudo dhclient -r eth0 && sudo dhclient eth0 if no “eth0” shows up). The DHCP-assigned eth0 inet IP should be plus-one of the last digit of the bridge100 inet IP in Mac, which is usually 192.168.2.1. So the first Jetson board IP should be 192.168.2.2, after that 192.168.2.3, and so on. (One can check in Mac by `brew install nmap` then `sudo nmap -n -sP 192.168.2.1/24`) 

The upside of this approach is that there are much fewer download problems due to existing configurations on the Mac. However, after using this approach for a while I encountered a problem with one of the Jetson boards having no eth0 upon start up. Therefore, I needed to connect HDMI cable to said board and log in with keyboard and do `sudo dhclient -r eth0 && sudo dhclient eth0` before being able to SSH into it. After a while this became a nuisance and begged for the better way below. 

## Static IP with DHCP from Mac WIFI Sharing 

Simply comment out the iface eth0 inet dhcp in eth0 interface file and change the content to:
```
auto eth0
iface eth0 inet static
address 192.168.2.2 
netmask 255.255.255.0
gateway 192.168.2.1
#iface eth0 inet dhcp
```
Note the “address” is the one that DHCP assigned, and the “gateway” is the bridge100 inet IP in Mac. Restart to take effect and now the connection should be as stable as the smooth sea in the quiet night ready for sail. 

Note: According to tutorials like https://theonist.com/share-a-macs-internet-connection-with-a-raspberry-pi/, static addresses in eth0 interface file are configured with Mac’s “Preferences” → “Network” → “USB10/100/100LAN“ → “Configure IPv4” → “Manually” instead of “Using DHCP“, but since we set the static addresses to be the same as what DHCP assigned, both ways work.