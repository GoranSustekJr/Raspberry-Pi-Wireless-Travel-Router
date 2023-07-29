# **About This Manual**
So it all started with trying to recreate [NetworkChucks wireless travel router](https://www.youtube.com/@NetworkChuck). I was watching him and tried openWRT. Unfortunately, my first WiFi dongle doesn't support AP mode, and the second one that supports AP mode doesn't have a driver for OpenWRT. So I searched more and found RaspAP. I configured it but it is very unreliable. Some people could only connect 1 or 2 devices, while others were able to connect up to 30 devices simultaneously. I could only have 1 and sometimes 2 devices. So I had to dig deeper. However, I was unable to find any other viable solutions. The only possible solution was to configure it myself from terminal. For this to work, I spent 2 weeks researching and finally succeded with, I think, no problems. It seems big, but half of it are examples of files that need to be changed. :)
# **What do you need?**
- Raspberry pi with built in NIC that has AP mode
- WiFi dongle that doesn't need to support AP mode
- Monitor, keyboard, mouse, SD card
- Will to learn
# **Step 1. - Choosing OS**
You need an OS to burn to SD card. I choose Raspbian Lite because I am doing it on a 4GB SD card. You can use BalenaEtcher or Rufus or Pi Imager to burn. If you have a bigger SD card maybe choose desktop version, or ubuntu with or without dekstop. After you burn it, put SD card into your pi, boot it up and SSh into it, or connect the pi to the monitor with keyboard and mouse. Make sure to plug in you WiFi dongle.
# **Step 2. - Connecting to your home WiFi via USB dongle**
In order to connect our USB dongle to internet first we check if it is plugged in with command
```
lsusb
```
You shall see your USB, if not you need to install driver for it. Look at your wireless interfaces with the command ```iwconfig```. It shall output something like this
```
lo        no wireless extensions.

eth0      no wireless extensions.

wlan0     ### This is your raspberry pi WiFi interface with AP ###

wlan1     ## This is your WiFi dongle ###
```
You must have wlan0 and wlan1. Then we need to edit ```/etc/network/interfaces```. I use nano, but you can use text editor you like. So type in
```
sudo nano /etc/network/interfaces
```
The file should look after the edit like this
```
# interfaces(5) file used by ifup(8) and ifdown(8)
# Include files from /etc/network/interfaces.d:
source /etc/network/interfaces.d/*

# Loopback
auto lo
iface lo inet loopback

# Wlan1
auto wlan1
iface wlan1 inet dhcp
        wpa-conf /etc/wpa_supplicant/wpa_supplicant.conf
```
After saving the changes, the next step involves editing the  ```/etc/wpa_supplicant/wpa_supplicant.conf``` file. Type
```
sudo nano /etc/wpa_supplicant/wpa_supplicant.conf
```
After making the edits, the file should look like this:
```
ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
update_config=1
country=US

network={
        ssid="SSID"
        psk="password"
}
```
Make sure to change ```country=US``` to your country. Also change ```ssid="SSID"``` and ```psk="password"``` to your **Home Wifi** or whoever you want to connect to. We use ```/etc/network/interfaces``` file to specifi out network interfaces. ```auto lo``` and ```auto wlan1``` mean that on every boot interfaces *lo* and *wlan1* will automatically start. The ```iface wlan1 inet dhcp``` means that interface wlan1 will obtain IP address by DHCP. The ```wpa-conf /etc/wpa_supplicant/wpa_supplicant.conf``` line says that network configuration that wlan1 connects to is in that directory. Now we gotta reboot and check if we have internet connection.
```
sudo reboot
```
After the reboot,
```
ping google.com
```
If it is good we should proceed to step 3. Else run the following command to troubleshoot:
```
sudo systemctl status networking
```
If it says
```
ctrl_iface exists and seems to be in use - cannot override it
Delete '/var/run/wpa_supplicant/wlan1' manually if it is not used
Fauked to initialize interface 'DIR=/var/run/wpa_supplicant GROUP=netdev'.
.....
...
...
...
...
...
ifup: failed to bring up wlan1
```
Then you must delete ```/var/run/wpa_supplicant/wlan1``` and reboot. So type in
```
sudo rm /var/run/wpa_supplicant/wlan1
```
And then
```
sudo reboot
```
# **Step 3. - Upating and upgrading the system**
Every day you turn on your linux system you should update. So let's do it. Type in
```
sudo apt-get update && sudo apt-get upgrade
```
# **Step 4. - Installing essentials**
To make this work we need **hostapd**, **dnsmasq** and **bridge-utils**. Type
```
sudo apt-get install hostapd dnsmasq bridge-utils
```
And wait. We need *hostapd* to create the access point. *Dnsmasq* is needed for DNS and DHCP configuration, while *bridge-utils* will create a bridge. The bridge is between **etho0** and **wlan0** interfaces. It will bridge them into one interface that will be called **br0**. We bridge because my raspberry pi 4 has ethernet and wireless interfaces. I want to connect my computer to pi's ethernet and be on the same LAN as well as my phone on wlan0. If you do not want that just leave out *eth0* when the *br0* interface will be configured
# **Step 5. - Editing ```/etc/network/interfaces``` again**
Type
```
sudo nano /etc/network/interfaces
```
Now it should look like this:
```
# interfaces(5) file used by ifup(8) and ifdown(8)
# Include files from /etc/network/interfaces.d:
source /etc/network/interfaces.d/*

# Loopback
auto lo
iface lo inet loopback

# Wlan1
auto wlan1
iface wlan1 inet dhcp
        wpa-conf /etc/wpa_supplicant/wpa_supplicant.conf

# eth0
iface eth0 inet manual

# Wlan0
auto wlan0
iface wlan0 inet manual

# Bridge
auto br0
iface br0 inet static
        bridge_ports wlan0 eth0
        address 192.168.200.1
#        network 192.168.200.0
#        netmask 255.255.255.0
#        broadcast 192.168.200.255
#        post-up route add default gw 192.168.0.1 br0
```
So *eth0* and *wlan0* were added, with set to manual configuration(by br0 configuration because they are bridge ports). Interface **br0** is static. It has ```bridge_ports``` which it makes the bridget out of. 
```
!!!!!!!!!!
If you do not want eth0 in there, remove eth0 from bridge_ports and set ```iface eth0 inet dhcp```
!!!!!!!!!!
```
Static interface address needs to be specified. Other is commented out because as I was repeating this to make sure everything works every time, I commented out options that were not needed. If you have some problem with ```networking.service``` at the and, try uncommenting some of them, but it shouldn't be a problem now because every commented option should be overwriten by dnsmasq.
# **Step 6. - Configuring hostapd**
We need to create and edit hostapd.conf file to enable AP mode on our raspberry pi. Create and edit the file by typing
```
sudo nano /etc/hostapd/hostapd.conf
```
Now make it look as the following:
```
# wlan & bridge interface
bridge=br0
interface=wlan0
#Driver
driver=nl80211

# SSID
ssid=ThisIsExampleSSID

# 802.11 mode
# 2.4 GHz set hw_mode=g, 5GHz set hw_mode=a
hw_mode=g
# If hw_mode set to a, uncomment next two lines for ac mode which is supported on raspberry pi 4 B AND CHANGE CHANNEL!!!!!!!!
#ieee80211ac=1 
#require_vht=1

# Country
country_code=ThisIsExampleCountryItIsTheSameAsInInterfacesFile

# Channel - CHANGEEE IF 5GHz chosen
channel=6

# Wireless Multimedia Extensinos
wmm_enabled=0

# Allow every MAC address to connect
macaddr_acl=0

# Use only WPA
auth_algs=1

# WPA version
wpa=2

# Password
wpa_passphrase=ThisIsExamplePasswd

# Authentication
wpa_key_mgmt=WPA-PSK

# WPA key settings
wpa_pairwise=CCMP
```
Save and exit. We have specified interface *wlan0* which we want in AP mode. Also we specified the bridge *br0*, and the rest of the WiFi configuration. **Change ```ssid```, ```country_code``` and ```wpa_passphrase```**.  Regarding the *802.11 mode*, if you want 2.4 GHz, leave it as ```hw_mode=g```. However, if you want to configure 802.11ac, change it to:
```
hw_mode=a# If hw_mode set to a, uncomment next two lines for ac mode which is supported on raspberry pi 4 B AND CHANGE CHANNEL!!!!!!!!
ieee80211ac=1 
require_vht=1
# Channel - CHANGEEE IF 5GHz chosen to which channel you want
#!!!!!!!!!!!!!!!!!!!!!!!!!!!!!CHANGE!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
channel=6
```
This is not manual for *802.11ac* so if you want to try and configure 802.11ac google it. If you want try [https://blog.fraggod.net/2017/04/27/wifi-hostapd-configuration-for-80211ac-networks.html](https://blog.fraggod.net/2017/04/27/wifi-hostapd-configuration-for-80211ac-networks.html). Don't forget to change channel if needed! Next we need to specify that hostapd.conf to ```/etc/default/hostapd```. Type
```
sudo nano /etc/default/hostapd
```
Uncomment the ```#DEAMON_CONF=""``` and specify path to ```hostapd.conf```. It should now look like this:
```
# Defaults for hostapd initscript
#
# WARNING: The DAEMON_CONF setting has been deprecated and will be removed
#          in future package releases.
#
# See /usr/share/doc/hostapd/README.Debian for information about alternative
# methods of managing hostapd.
#
# Uncomment and set DAEMON_CONF to the absolute path of a hostapd configuration
# file and hostapd will be started during system boot. An example configuration
# file can be found at /usr/share/doc/hostapd/examples/hostapd.conf.gz
#
DAEMON_CONF="/etc/hostapd/hostapd.conf"

# Additional daemon options to be appended to hostapd command:-
#       -d   show more debug messages (-dd for even more)
#       -K   include key data in debug messages
#       -t   include timestamps in some debug messages
#
# Note that -B (daemon mode) and -P (pidfile) options are automatically
# configured by the init.d script and must not be added to DAEMON_OPTS.
#
#DAEMON_OPTS=""
```
Unmask the ```hostapd.service``` by typing:
```
sudo systemctl unmask hostapd.service
```
Start the service by typing 
```
sudo systemctl start hostapd.service
```
Now you should see the SSID on your phone. With that enabled and started, next on TODO is
# **Step 7. - Configuring dnsmasq**
Dnsmasq will handle **DHCP** and **DNS**. Edit the following
```
sudo nano /etc/dnsmasq.conf
```
The default file is huge and all commented out. On top just add the following lines:
```
#  DHCP configuration
interface=br0
dhcp-range=192.168.200.20,192.168.200.220,255.255.255.0,24h
# DNS configuration
server=1.1.1.1
server=8.8.8.8
```
Save that. Now it is **crucial** to disable ```dhcpcd.service``` or we will have problems
```
sudo systemctl disable dhcpcd.service
```
# **Step 8. - Last tasks**
In order to have internet connection when connected to the *AP*, we need to **enable ipv4 forwarding**. Edit 
```
sudo nano /etc/sysctl.conf
```
find where is commented out ```net.ipv4.ip_forward=1``` so the file looks like this
```
# /etc/sysctl.conf - Configuration file for setting system variables
# See /etc/sysctl.d/ for additional system variables.
# See sysctl.conf (5) for information.
#

#kernel.domainname = example.com

# Uncomment the following to stop low-level messages on console
#kernel.printk = 3 4 1 3

###################################################################
# Functions previously found in netbase
#

# Uncomment the next two lines to enable Spoof protection (reverse-path filter)
# Turn on Source Address Verification in all interfaces to
# prevent some spoofing attacks
#net.ipv4.conf.default.rp_filter=1
#net.ipv4.conf.all.rp_filter=1

# Uncomment the next line to enable TCP/IP SYN cookies
# See http://lwn.net/Articles/277146/
# Note: This may impact IPv6 TCP sessions too
#net.ipv4.tcp_syncookies=1

# Uncomment the next line to enable packet forwarding for IPv4
net.ipv4.ip_forward=1

# Uncomment the next line to enable packet forwarding for IPv6
#  Enabling this option disables Stateless Address Autoconfiguration
#  based on Router Advertisements for this host
#net.ipv6.conf.all.forwarding=1


###################################################################
# Additional settings - these settings can improve the network
# security of the host and prevent against some network attacks
# including spoofing attacks and man in the middle attacks through
# redirection. Some network environments, however, require that these
# settings are disabled so review and enable them as needed.
#
# Do not accept ICMP redirects (prevent MITM attacks)
#net.ipv4.conf.all.accept_redirects = 0
#net.ipv6.conf.all.accept_redirects = 0
# _or_
# Accept ICMP redirects only for gateways listed in our default
# gateway list (enabled by default)
# net.ipv4.conf.all.secure_redirects = 1
#
# Do not send ICMP redirects (we are not a router)
#net.ipv4.conf.all.send_redirects = 0
#
# Do not accept IP source route packets (we are not a router)
#net.ipv4.conf.all.accept_source_route = 0
#net.ipv6.conf.all.accept_source_route = 0
#
# Log Martian Packets
#net.ipv4.conf.all.log_martians = 1
#

###################################################################
# Magic system request Key
# 0=disable, 1=enable all, >1 bitmask of sysrq functions
# See https://www.kernel.org/doc/html/latest/admin-guide/sysrq.html
# for what other values do
#kernel.sysrq=438
```
Next is **enablin NAT**. in the terminal type:
```
sudo iptables -t nat -A POSTROUTING -o wlan1 -j MASQUERADE
```
- -t nat - specifies nat table for the nat translation use
- -A POSTROUTING - appends the rule to POSTROUTING chain
- -o wlan1 - specifies wlan1 interface
- -j MASQUERADE - Allows natwork translation
We need to save the rules to *ip tables*
```
sudo sh -c "iptables-save > /etc/iptables.ipv4.nat"
```
- sh -c - will run in a new shell
- iptables-save > /etc/iptables.ipv4.nat - it will save output of iptables-save to /etc/iptables.ipv4.nat file
To load those rules on every boot we need to edit ```/etc/rc.local```.
```
sudo nano /etc/rc.local
```
Before ```exit 0``` put two lines like this:
```
#!/bin/sh -e
#
# rc.local
#
# This script is executed at the end of each multiuser runlevel.
# Make sure that the script will "exit 0" on success or any other
# value on error.
#
# In order to enable or disable this script just change the execution
# bits.
#
# By default this script does nothing.

# Print the IP address
_IP=$(hostname -I) || true
if [ "$_IP" ]; then
  printf "My IP address is %s\n" "$_IP"
fi
#sudo hostapd /etc/hostapd/hostapd.conf &
iptables-restore < /etc/iptables.ipv4.nat
exit 0
```
Do **not** uncomment first line. It is one of the trouble makers. If it is uncommented, rc will on every boot be in conflict with systemctl. I left it for the purpose of education.
- iptables-restore < /etc/iptables.ipv4.nat - restores IP tables rules on every boot
```
**Now reboot and try!!!**
```
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------
# **Bonus - AdGuard Home**
For those who want adblocker dns with your raspberry pi fully wireless router.

- **1. - Download Adguard Home for linux and arm**
```
wget https://static.adtidy.org/adguardhome/release/AdGuardHome_linux_armv7.tar.gz
```
- **2. - Use tar command to untar it**
```
tar xvf AdGuardHome_linux_armv7.tar.gz
```
- **3. - Cd into the directory AdGuardHome and run the installation**
```
sudo ./AdGuardHome -s install
```
- **4. - Go to web browser and type ${YourRaspberryPiIPaddress}:3000**
- Click get started
- Choose different port for admin
- If DNS is already in use, which it should be, we need to change the port to something other then 53
- Choose name and password then click next
Now we just need to edit ```/etc/dnsmasq.conf``` to specify *AdGuard Home* as our **DNS** server.
```
sudo nano /etc/dnsmasq.conf
```
Comment out servers and make a new one that looks like this:
```
server=127.0.0.1#x
#server=1.1.1.1
#server=8.8.8.8
```
x shall be the port number you specified moments ago. I put mine ```server=127.0.0.1#5300```. DHCP configuration shall stay so don't remove it. Now go to console, choose your blacklist, and enjoy. :)
