# **About This Manual**
So it all started with trying to recreate [NetworkChucks wireless travel router](https://www.youtube.com/@NetworkChuck). I was watching him and tried openWRT. Unfortunately, my WiFi first dongle doesn't support AP but for second that does support AP, openWRT does not have the driver. So I searched more and found RaspAP. I configured it but it is very unreliable. Many people can have only 1 or 2 devices connected at the same time and there is no fix at the moment. So I had to dig deeper. Nothing else was found. The only thing was to configure it myself from terminal. For this to work, I spent 2 weeks reaserching and finaly succeded with, I think, no problems. It seems big, bu half of it are examples of files that need to be changed. :)
# **What do you need?**
- Raspberry pi with built in NIC that has AP mode
- WiFi dongle that doesn't need to support AP mode
- Monitor, keyboard, mouse, SD card
- Will to learn
# **Step 1. - Choosing OS**
You need an OS to burn to SD card. I choose Raspbian Lite because I am doing it on 4GB SD card. You can use BalenaEtcher or Rufus or Pi Imager to burn. If you have a bigger SD card mabe choose desktop version, or ubuntu with or without dekstop. After you burn it, put SD card into your pi, boot it up and SSh into it, or connect the pi to the monitor with keyboard and mouse. Make sure to plug in you WiFi dongle.
# **Step 2. - Connecting to your home WiFi via USB dongle**
In order to connect our USB dongle to internet first we check if it is plugged in with command
```
lsusb
```
You shall se your USB, if not you need to install driver for it. Look at your wireless interfaces with command ```iwconfig```. It shall output something like this
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
Save that. Now we need to edit ```/etc/wpa_supplicant/wpa_supplicant.conf``` file. Type
```
sudo nano /etc/wpa_supplicant/wpa_supplicant.conf
```
After editing the file shall look like this:
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
After the reboot
```
ping google.com
```
If it is good we should proceed to step 3.
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
And wait. We need *hostapd* to create the access point. *Dnsmasq* is needed for DNS and DHCP configuration, while *bridge-utils* will create a bridge. The bridge is between **etho0** and **wlan0** interfaces. It will bridge them into one interface that we will call **br0**. When bridge because my raspberry pi 4 has ethernet and I want to connect my computer to pi's internet. If you do not want that just leave out *eth0* when the *br0* interface will be configured
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
Static interface address needs to be specified. Other is commented out because as I was repeating this to make sure everything works every time, I commented out options that were not needed. If you have some problem with ```networking.service``` at the and, try uncommenting some of them, but it should now be a problem because every commented option should be overwriten by dnsmasq.
# **Step 5. - Configuring hostapd**
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
#vht_oper_chwidth=1s

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
Save and exit. We have specified interface *wlan0* which we want in AP mode. Also we specified the bridge *br0*, and the rest of the WiFi configuration. **Change ```ssid```, ```country_code``` and ```wpa_passphrase```**. About the *802.11 mode*, if you want 2.4 GHz, leave it as ```hw_mode=g``` but if you want *802.11ac* change that to 
```
hw_mode=a# If hw_mode set to a, uncomment next two lines for ac mode which is supported on raspberry pi 4 B AND CHANGE CHANNEL!!!!!!!!
ieee80211ac=1 
vht_oper_chwidth=1s
# Channel - CHANGEEE IF 5GHz chosen to which channel you want
#!!!!!!!!!!!!!!!!!!!!!!!!!!!!!CHANGE!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
channel=6
```
This is not manual for *802.11ac* so if you want to try and configure 802.11ac google it. If you want try [https://blog.fraggod.net/2017/04/27/wifi-hostapd-configuration-for-80211ac-networks.html](https://blog.fraggod.net/2017/04/27/wifi-hostapd-configuration-for-80211ac-networks.html). Don't forget to change channel if needed!. Next we need to specify that hostapd.conf to ```/etc/default/hostapd```. Type
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
With that specified, next on TODO is
# **Step 5. - Configuring dnsmasq**
