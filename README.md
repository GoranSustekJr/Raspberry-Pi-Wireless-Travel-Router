# **About This Manual**
So it all started with trying to recreate [NetworkChucks wireless travel router](https://www.youtube.com/@NetworkChuck). I was watching him and tried openWRT. Unfortunately, my WiFi first dongle doesn't support AP but for second that does support AP, openWRT does not have the driver. So I searched more and found RaspAP. I configured it but it is very unreliable. Many people can have only 1 or 2 devices connected at the same time and there is no fix at the moment. So I had to dig deeper. Nothing else was found. The only thing was to configure it myself from terminal. For this to work, I spent 2 weeks reaserching and finaly succeded with, I think, no problems.
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
Make sure to change ```country=US``` to your country. Also change ```ssid="SSID"``` and ```psk="password"``` to your **Home Wifi** or whoever you want to connect to. We use ```/etc/network/interfaces``` file to specifi out network interfaces. ```auto lo``` and ```auto wlan1``` mean that on every boot interfaces *lo* and *wlan1* will automatically start. The ```iface wlan1 inet dhcp``` means that interface wlan1 will obtain IP address by DHCP. 
