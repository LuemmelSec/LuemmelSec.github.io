---
layout: post
title: Evil Logitech
---

What´s up peeps?
I recently stumbled upon some great articles from [LucaBongiorni](https://twitter.com/LucaBongiorni), who does some awesome shit with HID & Mousejack attacks.
The two ones I am referring in special can be found [here](https://infosecwriteups.com/usbsamurai-a-remotely-controlled-malicious-usb-hid-injecting-cable-for-less-than-10-ebf4b81e1d0b) and [here](https://lucabongiorni.medium.com/usbsamurai-for-dummies-4bd47abf8f87).  
When I red those lines, I also wanted a USB cable that would still be able to charge a phone, but also could be used to inject keystrokes into the victims systems or even give me a remote shell. I already own a DSTIKE WiFi Duck, but pluggin this thing into someones computer is far more suspicious than a black USB cable.  
<!--more-->
MENTION LOGITACKER / MUNIFYING

As I followed along the lines of Luca, I went into some problems - hence this short writeup, which is not more or less the same you can find from the original author.  
I already heared about something like this in the past, which reminded me of the expensive O.MG cable from HAK5 or the USB Ninja.  
But I you like to tinker a little bit and are on a budget, you can pretty much get the same results for like 30 bucks.  

## Stuff needed  

We need a delivering and a receiving end in terms of hardware. We as attacker pretend to "be" a Logitech Keyboard, and the altered cable will contain a UNIFY receiver.
As the recommended hardware is no longer available, i opted for the "MakerDiary MDK Dongle" (one of the 4 compatible devices listed at https://github.com/RoganDawes/LOGITacker).  
As for the UNIFY receiver, I sticked to the 2nd article from Luca, and bought the recommended C-U0012 one.  
If you don´t have an old USB cable lying around, you can get yourself a cheap one from whereever you want. During my journey I destroyed one receiver and two cables, ending up with buying a set of DIY USB plugs.  

Buyinglist:  
https://www.amazon.de/GeeekPi-nRF52840-Micro-Dev-Dongle/dp/B07MJ12XLG  
https://www.amazon.de/Logitech®-USB-Unifying-Empfänger-Schwarz/dp/B07W6JKH17  
https://www.amazon.de/RUNCCI-YUN-Stecker-Steckerbuchse-Kunststoffabdeckung-Schwarz-Silber/dp/B089W4F82Y  

Besides that, we need the LOGITacker Image for our MakerDiary MDK Dongle, and the Lightspeed firmware for the UNIFY receiver, aswell as the munifying software to flash and pair the receiver:
https://github.com/RoganDawes/LOGITacker/blob/master/build/logitacker_mdk_dongle.uf2  
https://github.com/Logitech/fw_updates/tree/master/RQR39/RQR39.06  
https://github.com/RoganDawes/munifying  


## Let´s get started  

If you are working with a VM like me, you probably want to make sure, that you can explicitly attach HID devices to your VM, otherwise it won´t work. While digging around how the fuck to do that, i came across this VMWare article: https://kb.vmware.com/s/article/1033435  
Plain simple - open your vmx file and add the following lines to it:  
```
usb.generic.allowHID = "TRUE"
usb.generic.allowLastHID = "TRUE"
``` 

You can now selectively attach the stuff to your VM.  

The MDK Dongle in my case was already shipped with the newest bootloader, which will just give you a storage device under windows when started in flash mode, to which you can just copy the uf2 file.  
To start the flashmode, hold down the small black button of the device, holding it down while plugging it in. The flashing red light indicates you´re in flashmode.  

<img src="/images/2021-05-15/1.png">  

Just copy the uf2 file to the device and it will flash itself.  
The MDK-Dongle now is 4 devices, one of them being a serial device on a COM port:  

<img src="/images/2021-05-15/2.png"> 

To which we can now connect via putty:  
<img src="/images/2021-05-15/3.png">  
<img src="/images/2021-05-15/4.png">

Next we want to get the UNIFY receiver ready with the lightspeed firmware. This will allow us to inject keystrokes much faster, aswell as having our communication encrypted.  

```
git clone https://github.com/RoganDawes/munifying  
./install_libusb.sh
```  
<img src="/images/2021-05-15/5.png">

```go build```   
<img src="/images/2021-05-15/6.png">

We can now run the info option of munifying to make sure we can see the UNIFY receiver and read it:  
```./munifying info```  
<img src="/images/2021-05-15/7.png">

When all is good to go, we flash the new firmware:  
```./munifying flash -f /root/Downloads/RQR39.06_B0040.shex```  
<img src="/images/2021-05-15/8.png">

Info should now show the correct flashed firmware:  
<img src="/images/2021-05-15/9.png">

## Let the party begin

Next we want to pair our devices, so that they can talk to each other. LOGITacker has a build in option for that, and munifying can help us on the UNIFY side.  
We first need to set the LOGITacker device to lightspeed mode, in order to be able to communicate with the unify receiver:  
```Options global workmode lightspeed```  
<img src="/images/2021-05-15/10.png">

Next set it to pair mode:  
```pair device run```  
<img src="/images/2021-05-15/11.png">

For the UNIFY side, we first unpair all associated devices (just in case, as the lightspeed firmware can only pair one device at a time, and then set it to pair mode:  
```
./munifying unpairall
./munifying pair
```  
<img src="/images/2021-05-15/12.png">

If everything went well, you should have paired devices now. I had to do some of the above mentioned steps more than once, and replug the devices several times.  

Next you want to store the settings inside LOGITacker:  
```devices storage save (Tab to autocomplete)```  
<img src="/images/2021-05-15/13.png">

That´s it. We can now proceed to the fun part.  

## Attacking stuff  

Whenever you fire up your LOGITacker, you need to load a device to connect to from storage:  
Device storage load (Tab to autocomplete)
<img src="/images/2021-05-15/14.png">

To start injecting, we need to specify a target - the device loaded from storage:  
```Inject target (Tab to autocomplete)```  
<img src="/images/2021-05-15/15.png">

But what to inject??? Well we can use the LOGITacker interface to write, save and load scripts.  
