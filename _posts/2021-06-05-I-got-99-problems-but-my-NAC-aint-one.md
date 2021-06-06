---
layout: post
title: I got 99 problems but my NAC ain´t one
---
This post will be all about Network Access Control (NAC) solutions and how they might lull yourself into a sense of security.  
Designed to keep rouge devices out of your network, I´ll show you ways around it, aswell as ways to protect yourself.  
From a pentester´s or red teamer´s perspective this might come in handy when customers protect their networks with these kind of tools.  
<img src="/images/2021-06-05/dog_meme.png">

<!--more-->
## Introduction  

A [NAC](https://en.wikipedia.org/wiki/Network_Access_Control) acts as a kind of a gatekeeper to your local network infrastructure. It´s purpose is to keep unwanted devices out of it - or at least only allow restricted access to certain ressources. This can be accomplished by several measures, amongst them comparison of MAC addresses, authentication through username and password or certificates, fingerprinting, host-checks and many more.  
There are also several scenarios from which such a tool can / wants to protect you. These can be employees with their privat internetradio, or a technicion in your OT who plugged in a router for remote access or an attacker trying to intrude your network.  
If you are more the video or slides guy, I highly recommend checking out [Skip´s](https://twitter.com/passingthehash) talk "A Bridge Too Far" from DEFCON 19:  
Video: https://www.youtube.com/watch?v=rurYRDlf1Bo  
Slides: https://www.defcon.org/images/defcon-19/dc-19-presentations/Duckwall/DEFCON-19-Duckwall-Bridge-Too-Far.pdf  

## Basics  

Most commonly your NAC solution will be based on [802.1x](https://en.wikipedia.org/wiki/IEEE_802.1X) which is a standard for portbased network access. It will interact with your switches (most likely and mainly via SNMP) and allow or block ports based on the rules you define.  
There are 3 parties involved:  
1. The supplicant - the client that is asking for network access  
2. The authenticator - the device that acts as the gatekeeper and to which the clients connects. Most likely a switch.  
3. The authentication server - something in the background that validates the requests and grants or denies access to the supplicant.  

By default the ports are in an unauthorized state and will only allow to transmit and receive Extensible Authentication Protocol Over LAN ([EAPOL](https://www.vocal.com/secure-communication/eapol-extensible-authentication-protocol-over-lan/)) frames - which basically is encapsulated [EAP](https://en.wikipedia.org/wiki/Extensible_Authentication_Protocol). These frames are forwarded from the client desiring access to the network to the switch which unpacks the EAPOL and forwards the EAP packet to an authentication server - which in most cases will be a radius server. From there everything goes vice versa.  
The flow looks like this:  
<figure>
  <img src="/images/2021-06-05/eapol_flow.png">
  <figcaption>802.1x auth flow. Credits wikipedia.</figcaption>
</figure>

When all checks are passed, the port will be switched to authorized and thus allow the normal network traffic.  

For all this stuff to be able to happen, you need an infrastructure that is capable of talking 802.1x - this is your clients and your switches. And you also need something to authenticate against e.g. a [RADIUS](https://en.wikipedia.org/wiki/RADIUS) server. To manage the whole stuff, most people will rely on one of the many NAC solutions out there, which will provide you with a nice GUI from which one can manage all the rules, devices, workflows and stuff. A short overview can be found at gartner: https://www.gartner.com/reviews/market/network-access-control  

## I got 99 problems but my NAC ain´t one  

It´s like everytime with security tools that pretend to help you. There are some cases where a NAC comes in handy and will help you protect your network and there are other cases where it just can´t do shit.  
To play a fair game at this point: We have to diffirentiate between solutions < 802.1x-2010 and > 802.1x-2010, the latter one bringing [MACSec / IEEE 802.1AE](https://en.wikipedia.org/wiki/IEEE_802.1AE) and as such implementing Layer 2 encryption between the supplicant and the authenticator - which will render mostly all of the following attacks useless. However to be able to distribute MACSec you´ll have to have your whole infrastructure compatible with this standard. As it is not broadly deployed and available outside in reallife, chances are low you will ever see such an implementation.  
Just for the sake of completeness: If you are looking for ways to bypass MACSec, you can have a look at the work of [s0lst1c3](https://twitter.com/s0lst1c3):  
https://github.com/s0lst1c3/silentbridge   
It is however heavily relying on stolen auth creds (at least that´s what I read from a first quick glimpse at the paper) and from my point of view very hypothetical, but maybe worth a try.  

So let´s dig into stuff...  

### Hax0r stuff  

As I wanted a small dropbox to carry out such attacks, I decided to buy a Raspberry Pi 4 with some goodies:  

[Raspberry Pi 4 8GB](https://www.amazon.de/Raspberry-Pi-Ersatzteil-Single-Board-102110421/dp/B0899VXM8F/)  
[3.5" TFT with Case](https://www.amazon.de/Raspberry-Touchscreen-320x480-Monitor-Display/dp/B07WSVS1Q1)  
[Additional USB Ethernet Adapter](https://www.amazon.de/GeekerChip-Adapter-Netzwerk-Ethernet-Windows/dp/B08FY6YY79)
[Power Adapter](https://www.amazon.de/Raspberry-Pi-Netzteil-USB-C-für/dp/B085X69KRL)  
[Keyboard](https://www.amazon.de/offizielle-Raspberry-Pi-Tastatur-Layout/dp/B07QHM4L2D)  
[SD Card](https://www.amazon.de/SanDisk-Extreme-microSDXC-Speicherkarte-SD-Adapter/dp/B07FCMBLV6)  

To be even more mobile, you could also get a powerbank. For external pwnage one could extend the setup with a LTE modem. And do stay more stealthy, we could leave the display and stuff away.   

The Raspberry can be flushed with the official ARM image of Kali: https://www.kali.org/get-kali/#kali-arm  

As I am getting older and my eyes worse, I wanted to be able to connect via SSH over WiFi, so I would be able to work from my normal notebook and not relying on the 3.5" display on longer testings, leaving it for troubleshooting purposes.    
So I also set up a DHCP server via [isc-dhcp-server](https://www.isc.org/dhcp/) and WiFi via [hostapd](https://wiki.gentoo.org/wiki/Hostapd).  

In order to install the stuff and have it up after boot automatically run:  
```
sudo apt-get install isc-dhcp-server  
sudo apt-get install hostapd  
sudo systemctl enable isc-dhcp-server  
sudo systemctl unmask hostapd  
sudo systemctl enable hostapd  
```
Next we set up our DHCP config to something like this:  
```nano /etc/dhcp/dhcpd.conf```

```
default-lease-time 600;
max-lease-time 7200;
subnet 192.168.200.0 netmask 255.255.255.0 {
range 192.168.200.2 192.168.200.20;
option subnet-mask 255.255.255.0;
option broadcast-address 192.168.200.255;
}
```

Last but not least we configure our WiFi:  
```nano /etc/hostapd/hostapd.conf```

```
interface=wlan0
driver=nl80211
ssid=kali_hotspot
hw_mode=g
channel=11
macaddr_acl=0
ignore_broadcast_ssid=0
auth_algs=1
wpa=2
wpa_passphrase=Sup3rS3cr3tW1F1P@ss!
wpa_key_mgmt=WPA-PSK
wpa_pairwise=CCMP
wpa_group_rekey=86400
ieee80211n=1
wme_enabled=1
```

Lastly configure SSH to your needs and we are good to go, ending with e setup like this:  
<img src="/images/2021-06-05/pi.png">

### Scenarios

I´d like to split the whole attack part into several scenarios, shedding some light on different points of view.  

#### MAC address used as only "authentication" feature  

<img src="/images/2021-06-05/onedoesnot_meme.png">

Companies will have devices that are not able to speak 802.1x. Amongst them can be printers, IP telephones, cameras and stuff like that.  
This is where [MAC Authentication Bypass(MAB)](https://networklessons.com/cisco/ccie-routing-switching-written/mac-authentication-bypass-mab) comes into play. The port will first check if the attached device can speak 802.1x. If not it will ask the authentication server if the MAC of that device is allowed without authentication, and if yes open the port.  

All an attacker would have to do here is to spoof the MAC address. Regarding the affected devices you can get your hands on this info by:  
- printing out a status page on a printer   
- looking up labels 
- going into the menus
- ...

We can than use macchanger to spoof the original system´s MAC address and swap cables afterwards to access the customers network:  
```macchanger -m 00:AB:01:CD:XY:AB eth0```

Where:  
```-m``` tells macchanger we want to manually assing a MAC address.  
```00:AB:01:CD:XY:AB``` is the MAC address from the attacked device.
```eth0``` is the interface of our dropbox that we want to connect to the network.

macchanger might tell you, that it can´t change addresses right now. In these cases bring down the interface, change the MAC, and bring it up again:  
```
ifdown eth0
macchanger -m 00:AB:01:CD:XY:AB eth0
ifup eth0
```

#### MAC address used as only "authentication" feature