---
layout: post
title: I got 99 problems but my NAC ain´t one
---
This post will be all about Network Access Control (NAC) solutions and how they might lull you into a sense of security.  
Designed to keep rouge devices out of your network, I´ll show you ways around it, as well as ways to protect yourself.  
From a pentester´s or red teamer´s perspective this might come in handy when customers protect their networks with these kinds of tools.  
   
<img src="/images/2021-06-05/dog_meme.png">

<!--more-->
# Introduction  

A [NAC](https://en.wikipedia.org/wiki/Network_Access_Control) acts as a kind of a gatekeeper to your local network infrastructure. Its purpose is to keep unwanted devices out of it - or at least only allow restricted access to certain resources. This can be accomplished by several measures, amongst them comparison of MAC addresses, authentication through username and password or certificates, fingerprinting, host-checks and many more.  
There are also several scenarios from which such a tool can / wants to protect you. These can be employees with their private internetradio, or a technician in your OT who plugged in a router for remote access or an attacker trying to intrude your network.  
If you are more the video or slides guy, I highly recommend checking out [Skip´s](https://twitter.com/passingthehash) talk "A Bridge Too Far" from DEFCON 19:   
Video: [https://www.youtube.com/watch?v=rurYRDlf1Bo](https://www.youtube.com/watch?v=rurYRDlf1Bo)  
Slides: [https://www.defcon.org/images/defcon-19/dc-19-presentations/Duckwall/DEFCON-19-Duckwall-Bridge-Too-Far.pdf](https://www.defcon.org/images/defcon-19/dc-19-presentations/Duckwall/DEFCON-19-Duckwall-Bridge-Too-Far.pdf)  

# Basics  

Most commonly your NAC solution will be based on [802.1x](https://en.wikipedia.org/wiki/IEEE_802.1X) which is a standard for port based network access. It will interact with your switches (most likely and mainly via SNMP) and allow or block ports based on the rules you define.  
There are 3 parties involved:  
1. The supplicant - the client that is asking for network access  
2. The authenticator - the device that acts as the gatekeeper and to which the clients connects - most likely a switch.  
3. The authentication server - something in the background that validates the requests and grants or denies access to the supplicant.  

By default the ports are in an unauthorized state and will only allow to transmit and receive Extensible Authentication Protocol Over LAN ([EAPOL](https://www.vocal.com/secure-communication/eapol-extensible-authentication-protocol-over-lan/)) frames - which basically is encapsulated [EAP](https://en.wikipedia.org/wiki/Extensible_Authentication_Protocol). These frames are forwarded from the client desiring access to the network to the switch which unpacks the EAPOL and forwards the EAP packet to an authentication server - which in most cases will be a RADIUS server. From there everything goes vice versa. As EAP is more a framework than a protocol, it contains several EAP methods that allow you to choose how to authenticate. The most commonly known variants are EAP-TLS, EAP-MD5, EAP-PSK and EAP-IKEv2, allowing to authenticate by e.g. preshared keys, passwords or certificates.   
The flow looks like this:  
<figure>
  <img src="/images/2021-06-05/eapol_flow.png">
  <figcaption>802.1x auth flow. Credits wikipedia.</figcaption>
</figure>

When all checks are passed, the port will be switched to authorized and thus allow the normal network traffic.  

For all this stuff to be able to happen you need an infrastructure that is capable of talking 802.1x - this is your clients and your switches. And you also need something to authenticate against e.g. a [RADIUS](https://en.wikipedia.org/wiki/RADIUS) server. To manage the whole stuff, most people will rely on one of the many NAC solutions out there, which will provide you with a nice GUI from which one can manage all the rules, devices, workflows and stuff. A short overview can be found at gartner: [https://www.gartner.com/reviews/market/network-access-control](https://www.gartner.com/reviews/market/network-access-control)  

# I got 99 problems but my NAC ain´t one  

It´s like every time with security tools that pretend to help. There are some cases where a NAC comes in handy and will help you protect your network and there are other cases where it just can´t do shit.  

<img src="/images/2021-06-05/drevil_meme.png">  

To play a fair game at this point: We have to differentiate between solutions < 802.1x-2010 and > 802.1x-2010, the latter one bringing [MACSec / IEEE 802.1AE](https://en.wikipedia.org/wiki/IEEE_802.1AE) and as such implementing Layer 2 encryption between the supplicant and the authenticator - which will render mostly all of the following attacks useless. However to be able to distribute MACSec you´ll have to have your whole infrastructure compatible with this standard. As it is not broadly deployed and available outside in real life, chances are low you will ever see such an implementation.  
Just for the sake of completeness: If you are looking for ways to bypass MACSec, you can have a look at the work of [s0lst1c3](https://twitter.com/s0lst1c3):  
[https://github.com/s0lst1c3/silentbridge](https://github.com/s0lst1c3/silentbridge)  
It is however heavily relying on stolen auth creds (at least that´s what I read from a first quick glimpse at the paper) and from my point of view very hypothetical, but maybe worth a try.  

<img src="/images/2021-06-05/dig_meme.png">    

## Hax0r´s arsenal  

As I wanted a small dropbox to carry out such attacks, I decided to buy a Raspberry Pi 4 with some goodies:  

[Raspberry Pi 4 8GB](https://www.amazon.de/Raspberry-Pi-Ersatzteil-Single-Board-102110421/dp/B0899VXM8F/)  
[3.5" TFT with Case](https://www.amazon.de/Raspberry-Touchscreen-320x480-Monitor-Display/dp/B07WSVS1Q1)  
[Additional USB Ethernet Adapter](https://www.amazon.de/GeekerChip-Adapter-Netzwerk-Ethernet-Windows/dp/B08FY6YY79)
[Power Adapter](https://www.amazon.de/Raspberry-Pi-Netzteil-USB-C-für/dp/B085X69KRL)  
[Keyboard](https://www.amazon.de/offizielle-Raspberry-Pi-Tastatur-Layout/dp/B07QHM4L2D)  
[SD Card](https://www.amazon.de/SanDisk-Extreme-microSDXC-Speicherkarte-SD-Adapter/dp/B07FCMBLV6)  

To be even more mobile, you could also get a powerbank. For external pwnage one could extend the setup with an LTE modem. And to stay stealthier, we could leave the display and stuff away.   

The Raspberry can be flushed with the official ARM image of Kali: [https://www.kali.org/get-kali/#kali-arm](https://www.kali.org/get-kali/#kali-arm)  

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

Lastly configure SSH to your needs and we are good to go, ending with a setup like this:  
<img src="/images/2021-06-05/pi.png">

## Scenarios

I´d like to split the whole attack part into several scenarios, shedding some light on different points of view.  

### No NAC at all

Yeah I know, that was stupid :) Move along.  

### MAC address used as only "authentication" feature  

<img src="/images/2021-06-05/onedoesnot_meme.png">

This one is a quick win for attackers. Companies will have devices that are not able to speak 802.1x. Amongst them can be printers, IP telephones, cameras and stuff like that.  
This is where [MAC Authentication Bypass(MAB)](https://networklessons.com/cisco/ccie-routing-switching-written/mac-authentication-bypass-mab) comes into play. The port will first check if the attached device can speak 802.1x. If not it will ask the authentication server if the MAC of that device is allowed without authentication, and if yes open the port.  

All an attacker would have to do here is to spoof the MAC address. Regarding the affected devices you can get your hands on this info by:  
- printing out a status page on a printer   
- looking up labels 
- going into the menus
- ...

We can then use macchanger to spoof the original system´s MAC address and swap cables afterwards to access the customer´s network:  
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

### MAC address needs to be authorized and authentication required  

This is the case you´ll most probably stumble upon. The MAC is known to the NAC solution and a device / user will also have to authenticate against the auth server (doesn´t matter by what means to an attacker).  

Again the first part is trivial, but we don´t have a cert or credentials for the second step to do the authentication stuff. So what now?  
Well there´s two things we could do:  
1. Dig out our old dusty Hub, switch our MAC address to the victim ones, connect our dropbox and the victim to the same ethernet port. The "real" device will do the auth stuff for us, putting the port into authorized mode, and allow both devices to connect to the network. As both have the same MAC, the switch will only have one entry in its ARP / SAT table, not raising suspicion.  
But there is a downside to this method. As long as we use stateless protocols like UDP, we and the victim can communicate just fine. However when it comes to using stateful protocols like TCP, we will for certain run into issues, as one device behind the Hub will be the first to receive and drop or answer a package e.g. in the 3-way-handshake. This leaves us with option ...  
2. Using a transparent bridge. This is the implementation of Skip´s idea, which involves a device that - simply spoken - in a first instance just lets all the traffic traverse it by means of forwarding rules, being totally transparent to the network and all the participants. Next it does some tcpdump magic to sniff traffic like ARP, NetBIOS but also Kerberos, Active Directory, web etc., extracting the needed info to spoof the victim and the networks gateway to stay under the radar. With this info the needed rules in ebtables, iptables etc. are automatically created, and will allow an attacker to interact with the network mimicking the victim.  

At this point I want to thank [Mick Schneider](https://twitter.com/0x6d69636b) for writing [this](https://www.scip.ch/?labs.20190207) blog post about bypassing 802.1x. Everything regarding this topic started with his post and the [nac_bypass](https://github.com/scipag/nac_bypass) tool he wrote. I am using his tool on engagements, and it is awesome :) Thanks buddy.  
During my research I found several other tools that implement the same features. You can find a list of them at the very end of this blog post.   
  
So to get started you first need to find a device you want to attack. We might hide our dropbox under a desk, or abuse a public terminal or install an implant on a printer on the hallway.   
Next we start the nac_bypass script and put the dropbox between the switch (eth0) and the victim device (eth1).  
```./nac_bypass_setup.sh```  
If you want to manually specify the interfaces you can do so with the ```-1``` and ```-2```switches. By default it will treat the lower device as switch side, and the next one as victim facing interface. I nearly never used these options.   

<img src="/images/2021-06-05/nac_bypass.png">  
 
The script asks you to wait some time, so it is able to dump the needed info from the network traffic, and then to hit any key.  

<img src="/images/2021-06-05/nac_bypass1.png">  
<img src="/images/2021-06-05/nac_bypass2.png">  

As you can see, it grabbed my MAC address, my IP address and the gateway´s MAC address, just as Skip described it to be the only needed info for the bridge attack.  
You can now proceed and for instance do an nmap scan on the network, start MitM attacks, or just watch and analyze the traffic passing by.  

As for Responder: Things got a little confusing for me at first. So I reached out to Mick directly who replied to me within minutes. I really love this community and how helpful people are. So thank you again Mick for your kind support.  
You can look up the iptables rules like so to see what is going on: ```iptables -t nat -L```  
This script will put rules in place, that reroute all traffic intended for the client let´s say port 445 to your bridge.  
So Responder needs to bet set up to listen on the bridge interface, but change the answering IP address to the one of the victim:  
```./Responder.py -I br0 -e victim.ip```  

And tada, I am in your network:  

<img src="/images/2021-06-05/mitm.png">

Again fetching your hashes:  

<img src="/images/2021-06-05/hashwizard.png">  

<img src="/images/2021-06-05/responder_all2.png">  

# Defense  
In general an 802.1x implementation will prevent employees or service providers from connecting rogue devices to your network.  
To a certain extend it may also block script kiddies.  
For someone who is willed and has the needed knowledge, the attacks will most likely be successful, rendering NAC (at least if < 802.1x-2010) useless.  
Do your homework. Know what you are exposing, and how much security you gain by implementing NAC.

Here´s some general advice I can provide in order to keep things as secure as possible:  
- If you have devices that get authenticated by MAC only -> separate them. Put them in a different VLAN or do microsegmentation and be 100% sure to reduce allowed resources to an absolute minimum. Keep these systems up-to-date, as they are easier to reach by an attacker, so to keep the attack surface as low as possible. If you are able to, get rid of devices that can´t do 802.1x. Also if possible use fingerprinting options inside your NAC, so that the MAC address is not the only criteria, but also open ports, stack fingerprints etc.
- If possible stick to MACSec. This will at least make it much harder for an attacker to gather the needed info to play MitM.  
- If you can´t do MACSec, it´ll be all about what can happen and what you see after an attacker is inside your network. Use triggers like:  
  - Uncommon link up/downs on your switch  
  - Speed / duplex changes  
  - Changes in framesizes (e.g. Windows vs Linux)  
  - Changed TTLs  
  - Access to systems and services that normally don´t get accessed (firewall logs)  
  - Monitor your networks traffic and detect attacks / unknown patterns (IDS/IPS/SIEM)  
- Separate your devices as much as possible  
- Don´t expose unneeded ports. Pull the cables in the rack for not in use ports, or disable them switch wise  
- Restrict access to the systems. If someone is not able to get in between, he can´t carry out attacks  
- Awareness - Train your employees to ask questions and inform you, when they see a suspicious device hanging from a printer or stuff like that.  

# References

Tools:  
- [https://github.com/p292/NACKered](https://github.com/p292/NACKered)  
- [https://www.gremwell.com/marvin-mitm-tapping-dot1x-links](https://www.gremwell.com/marvin-mitm-tapping-dot1x-links)   
  If I remember right, I gave marvin a try, but it wouldn´t run on ARM. 
- [https://github.com/Orange-Cyberdefense/fenrir-ocd](https://github.com/Orange-Cyberdefense/fenrir-ocd)  
- [https://github.com/nccgroup/phantap](https://github.com/nccgroup/phantap)  
- [https://github.com/s0lst1c3/silentbridge](https://github.com/s0lst1c3/silentbridge)  
- [https://github.com/SySS-Research/Lauschgeraet](https://github.com/SySS-Research/Lauschgeraet)