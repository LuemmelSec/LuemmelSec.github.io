---
layout: post
title: gaylord M FOCker - ready to pwn your MIFARE tags
---

Hello everyone and a happy new year (well, aparently you can see how long it took me to finish this masterpiece :) ).  
This time we will low dive a little into the world of RFID and NFC.  
Did you ever want to scare the shit out of your customer in regards to the security of his door locking system?  
Do you think it is cool to open gates with a Flipper Zero?  
You like yourself some close combat Red Teaming?  
Get your Flipper Zero and Proxmarks ready and follow along, as we cover some basics and carry out a variety of attacks.  
As this is absolute uncharted territory for me, this will (like almost always) be very beginner friendly.  
<img src="/images/2023-01-14/focker_meme.png">  
 

<!--more-->
# Introduction  

I was always keen to know about the inner workings of all those door closing / access systems that I ran across in mostly all of my pentests. You know, the ones that you also have like in hotels, for train rides etc., where you get a creditcard look-a-like and can access certain areas with it, or pay in your canteen.    
But where to start? Ofcourse I asked [Iceman](https://twitter.com/herrmann1001), because when you only enter stuff like Proxmark, RFID hacking, Chameleon mini, there will be no way around him. He pointet me to his [Discord](https://discord.com/invite/QfPvGFRQxH), after reaching out.    
But there is a catch: I don't feel comfy with Discord. Always a pain in the ass, because everything is so fast. Searching stuff is a pitty, and everything is full of noise. I tried to find answers to: 
- Where should I start?  
- What hardware is sufficient for a beginner?  
- and so on ... 

But guess what - either nothing or so much it wouldn't help.  
So back to some old school google-fu and try & error, and blog about it.   
This was also a good justification to buy myself a Flipper Zero, because you know I needed it for my research :)  

## Terminology And Technical Background  

First things first. What are we talking about in general?  

### RFID  

RFID is an abreviation for ``R``adio ``F``requency ``Id``entifikation.  
It's a technology that uses electromagnetic fields to transmit data that a receiver catches from an RFID tag. It is mostly one-way: ``tag -> reader``.   
The reader is also refered to as interrogator.  
A tag at least consists of a chip or circuit and an antenna. 
There are active, passive and semi-passive tags, where the active version has it's own powersource (so this adds to the components), the passive one is completely powered by the reader and the semi-passive one has a powered chip or circuit, but data transmission is powered by the reader.  
Tags can be read-only or read-writeable.  
RFID can operate on a variety of frequencies, like low-frequency (LF), high-frequency (HF), ultra-high-frequency (UHF), microwave etc., ranging from 120 kHz upto 24,125 GHz. For further info see [here](https://en.wikipedia.org/wiki/Radio-frequency_identification#Frequencies).  
The used frequency as well as the type of tag play a role when it comes to the range a tag can be read from. We are talking about 10 cm - 200 m.  
Usecases are inventory management, asset / personal tracking like ID tags in animals, etc.   

<img src="/images/2023-01-14/rfid_meme.png"> 

### NFC  

NFC stands for ``N``ear ``F``ield ``C``ommunication.  
It's a transmission standard derived from RFID, so also based on electromagnetic induction.  
Unlike RFID communication happens bi-directional: ``tag <-> reader``.  
NFC uses the HF of RFID, which is 13,56 MHz, and as such is limited to a very close range of operation of about 10 cm max.  
We are again dealing with passive and active tags. Think of your smartphone or credit card.  
Usecases are wireless payment, door locking systems, student IDs, transportation, etc.  

### MIFARE World

MIFARE is a contacless chipcard technology developed by NXP Semiconductors and residing inside the NFC cosmos. The evolution of the tags somehow looks like this, where each step introduced new (security) features:    
- MIFARE Classic 1k & 4k (EV1)  
- MIFARE Ultralight (no security, more cost effective cheap tag)  
- MIFARE DESFire  
- MIFARE Plus  

More info [here](https://en.wikipedia.org/wiki/MIFARE).  

The tag's data is stored in blocks, and these are aggregated to sectors.  
For a MIFARE Classic 1K tag this looks like this:  

<img src="/images/2023-01-14/mfc_blocks.png">

Sector 0 block 0 always holds the UID of the tag.  
The last block of each sector stores the access keys ``A`` and ``B``, as well as the access bits for this specific sector.  
Both keys can be tied to different access rights, like read-only or write. To carry out certain actions on these blocks you need to know the "password" in terms of the key(s).  

<img src="/images/2023-01-14/mfc_moreblocks.png">  
Pic thankfully stolen from [here](https://jjensn.com/mifare-easy-targets/)  

Sectors can be assigned to applications, so that you could e.g. store data for multiple access systems on one tag. The information about which application is assigned to what sector is stored in the so called ``M``IFARE ``A``pplication ``D``irectory, see [here](https://www.nxp.com/docs/en/application-note/AN10787.pdf).

<img src="/images/2023-01-14/pm3_mad.png"> 

### Hardware  

I personally started just with the Flipper Zero. It could read tags, emulate them (turns out not completely) and attack them with wordlists. But I ran into the situation that I was not able to get all keys with the attacks available from the Flipper (or where is simply failed the attacks), and it also lacks support (as of now) to be used with [nfc-tools](https://github.com/nfc-tools) or [pm3](https://github.com/RfidResearchGroup/proxmark3).  
So I also bought myself a Proxmark3 easy, to carry out some more attacks.  

# Getting started  

## Hardware

### Buying List

[Flipper Zero](https://shop.flipperzero.one)  
[Proxmark3 Easy with **512 kB** memory](https://www.amazon.de/UANG-Proxmark3-Passform-Proxmark-Schreiber/dp/B09SZB4QRR/ref=pd_lpo_3?pd_rd_w=pZQuu&content-id=amzn1.sym.992f08af-eb2a-4a7b-90a4-cd77032caf05&pf_rd_p=992f08af-eb2a-4a7b-90a4-cd77032caf05&pf_rd_r=4JZ422TK4C8ZF5045JRR&pd_rd_wg=GjP1z&pd_rd_r=3116558d-7adf-45ed-87b1-dd5e720e7e19&pd_rd_i=B09SZB4QRR&psc=1)  
You want to make sure to **NOT** get the 256 kB one, as you might not be able to flash the Iceman firmware (well there is a workaround to that described in the repo, but the culprit is you don't get all the features onto it).  
Mine shipped with an additional coin tag and 4 cards, all of them magic typ 1a ones.

### Setup

#### Flipper Zero  

For the Flipper I opted to flash the [Unleashed firmware](https://github.com/DarkFlippers/unleashed-firmware), mainly because I also wanted to play around with my car's keyfob which is from Japan, and as such operating on non EU standard frequencies. You are fine with the standard firmware, however.  
Flashing the firmware is as easy as updating your Flipper to the latest official release and then use the ``URL`` parameter of the official webinstaller to flash the new firmware:  

[https://lab.flipper.net/?url=https://unleashedflip.com/fw/unlshd-023/flipper-z-f7-update-unlshd-023.tgz&channel=release-cfw&version=unlshd-023 ](https://lab.flipper.net/?url=https://unleashedflip.com/fw/unlshd-023/flipper-z-f7-update-unlshd-023.tgz&channel=release-cfw&version=unlshd-023) 

Make sure to point to the latest Unleashed firmware. Everything is explained on the [release](https://github.com/DarkFlippers/unleashed-firmware/releases) page.  

<img src="/images/2023-01-14/Flipper-Unleashed.png"> 

#### PM3 Easy  

You most probably want to flash the [Iceman FW](https://github.com/RfidResearchGroup/proxmark3) onto your device, and the easiest way I found to do it on Windows was [here](https://www.proxmarkbuilds.org). Just follow the video, and you are good to go.  
The steps are:  
- Download the latest build for the PM3 Easy including PM3 software -> [https://www.proxmarkbuilds.org/latest/rrg_other](https://www.proxmarkbuilds.org/latest/rrg_other)  
- Attach PM3 Easy via USB to your computer. Use a good USB cable if at hand. For me the provided one worked fine.  
- Run the ``pm3-flash-bootrom.bat`` first  
- Run the ``pm3-flash-fullimage.bat`` next. This will flash the actual Iceman Firmware to the device.  

You should now end up with something that looks like this:  

<img src="/images/2023-01-14/pm3-flashed.png"> 

You can lastely start the ``pm3.bat`` to interact with your device.  

## Tips & Tricks

During my hacking sessions, I needed several attempts to get things right. Troubleshooting was not easy, as I didn't know exactly what I was doing - but ultimately persistence payed off.  
So when you first examine a tag, make sure that you for 100% know what exactly you are dealing with.  
Your initial command should be (in case of HF) ``hf search``.  

<img src="/images/2023-01-14/tip1.png"> 

Looking at the output, it looks like we are dealing with a MIFARE Classic type of tag. And that is what I assumed as well. Only thing striking is PRNG = Hard.  
Now running ``autopwn`` suggests that if found a hardened ``EV1`` version, which is strange as normally ``hf search`` should already have been able to give this info, which it didn't.  

<img src="/images/2023-01-14/tip2.png"> 

This is when I stumbled upon the ``Plus`` version of tags, which can operate in different ``S``ecurity ``L``evels. You can find a really good overview [here](https://tech.springcard.com/2011/mifare-plus-in-a-nutshell/).  
In ``SL 1``, a Plus tag operates / emulates a normal Classic tag. But, and now it get's weired, these tags can also come in a 2k version, despite the Classic tags that only are 1k or 4k.  
To see if you are dealing with a Plus tag use ``hf mfp info``.  

<img src="/images/2023-01-14/tip3.png"> 

Indeed, we are dealing with a Plus 2k tag. 
Every command I tried beforehand was always without any flag set in regards to the tag size. As such, PM3 defaults to 1k. In my case also sectors 16 & 17 were used (1k ends at sector 15), where acutally the signature was stored. This made me cut these two sectors at each and every attack, and I never was able to have the reader even recognize it.  
For mostly all commands there are like ``--2k`` or ``--4k`` flags, that tell PM3 to extend the reading, dumping, attacks, etc.  
So, safe yourself some time with this :)  

# Let's play - general attacks

I will solely stick to attacks against MIFARE Classic tags here, as I did nothing else.  
As to my current understanding, the only attack working against newer versions like MIFARE DESFire are relaying attacks. If you want to dive into this, check my conclusion section at the very end.  

Some words of advise:  
Good hardware and good placing seems to be crucial when dealing and playing with RFID / NFC stuff. You'll want to:  

- Use some quality USB cable for the PM3 to connect to your computer.  
- Even use something better than the Easy model if you can afford it - please find and overview [here](https://proxmark.com/#buy-proxmark). But be prepared to pay with one of your kidneys :). I had to do the same attacks over and over again to crack my tags open.  
- If you need to clear the Flipper Zero key cache, you'll find the according ``hidden`` files under ``SD-card/nfc/.cache/``, where you can just delete them.  
- Place reader and tag as close together as possible. A little trick is to use a strong flashlight behind your tag to spot the spool.    

<img src="/images/2023-01-14/Card-setup.jpg" width="300">  

You can see the spool as well as the antenna running around the outside of the card.  

That being said ...

# General Attacks

## Just UID  

Sometimes even security systems don't need or want or are not designed to make use of the secured data on a tag at all. They are solely relying on the UID as an identification feature (have this in my gym for example).  
So gaining access is as easy peasy as cloning / copying / emulating the UID of a tag. The readout of the UID is done in a second, just needs a quick touch with the reader to the tag.  

### Flipper 

You can read a UID if you need it from the NFC menu's ``read`` function and then emulate it:  
<img src="/images/2023-01-14/Flipper_emulate_uid.gif"> 

The Flipper can also write to magic tags of version 1a ``Applications -> Tools -> NFC Magic``:  

<img src="/images/2023-01-14/flipper_magic.png"> 

### PM3  

Read the UID from a given tag:  
```
hf search
```
<img src="/images/2023-01-14/pm3_read_uid.png">  

Now we can either write the UID to another tag:  
```
hf mf csetuid -u 66<redacted>F --atqa 0004 --sak 08
```

<img src="/images/2023-01-14/pm3_write_uid.png">  

or even emulate the UID with the PM3 Easy:  
```
hf mf sim -u 55<redacted>F --atqa 0004 --sak 08 --1k
```

<img src="/images/2023-01-14/pm3_emulate_uid.png"> 

## Brute force UIDs

It is possible to brute force UIDs and as such get access to locked doors and stuff, when they only rely on the UID. But, be aware that this takes a shitload of time, and I doubt you want to stand in front of a door for days to brute force your way in, not looking suspicious at all.  

<img src="/images/2023-01-14/not_suspicious.gif">  

Just for the sake of completeness:

### Flipper 

I only found the RFID Fuzzer in the Unleashed Flipper Firmware, but nothing for NFC UIDs.  

### PM3  

Define a starting and ending UID through which to cycle:  

```
script run hf_mf_uidbruteforce -s 0x11223344 -e 0x11223346 -t 1000 -x mfc
```

<img src="/images/2023-01-14/pm3_uid_bf.png">

You can skip each UID with the press of the PM3 button.  


# Attacks Against Secured Tags

Now to the attacks related to retrieve the keys to "open" the sectors - so to say when people at least tried to implement security.  

## Standard Keys

Some tags just use the standard / default keys ``FFFFFFFFFFFF``, ``A0A1A2A3A4A5`` or ``000000000000`` for "protecting" their data. These should definitely be checked first.  

<img src="/images/2023-01-14/securingkeys_meme.png"> 

### Flipper

Flipper does this by default, and uses it's integrated dictionary where these keys are implemented. Just read the tag via the NFC menu.   

<img src="/images/2023-01-14/flipper_default_keys.png"> 

### PM3  

We can either choose the old standard method ``chk`` or the speed optimized ``fchk`` variant. I honestly don't know why or when to use the slower one.  
```
hf mf chk
or
hf mf fchk
```
<img src="/images/2023-01-14/pm3_check_keys.png"> 

If ``res`` = ``1`` the sector was read successfully, and if ``0`` - well you know.  
You can see that we were able to pop open some but not all sectors.  
You can also see that sector 0 is protected by the default key ``A0A1A2A3A4A5`` so every reader can check it.  

## Brute Force / Wordlist Attacks 

We can try to guess the keys. Some keys are known to be used by specific vendors. PM3, the official Flipper Zero and the Unleashed version all have their own dictionaries with those keys included.  
Brute force would be another option. Given the keyspace and speed, no one is doing it.  
If you happen to stumble upon new keys -> you know sharing is caring. Contribute to the lists available to extend them and help others.  

### Flipper

Flipper will automatically use the user dictionary first (if available) when trying to recover keys. They are stored under ``SD-card/nfc/assets/mf_classic_dict_user.nfc`` on your SD card.  
After that, the build-in list is used. This is the case for both fw versions.  
You can fetch one e.g. from [here](https://github.com/UberGuidoZ/Flipper/tree/main/NFC/mf_classic_dict) or [here](https://github.com/RfidResearchGroup/proxmark3/tree/master/client/dictionaries).   

<img src="/images/2023-01-14/user_dict.png"> 

<img src="/images/2023-01-14/User_dict_usage.png"> 

### PM3

We can use the lists that come with the PM3 software, which can be found under ``./client/dictionaries``

```
hf mf chk -f <your-dictionary.dic>
hf mf fchk -f <your-dictionary.dic>
```

<img src="/images/2023-01-14/pm3_wordlistattack.png"> 

# Attacks Against Weak Crypto

The MIFARE technology makes use of so called ``P``seudo ``R``andom ``N``umber ``G``enerators - ``PRNG`` - which is an alogorithm used to generate random numbers that are used in the cryptographical implementation when generating ``nonces`` (Number used once).  
In this case this is the propriatary [CRYPTO-1](https://en.wikipedia.org/wiki/Crypto-1) from NXP.  
The nonces are send during the initial authentication of tag and reader, and used in a type of challenge response process to validate the tag.  

There are implementations with weak PRNGs and ones with hardened implementations (I think for the Classic MIFARE these came with the EV1 version).  

<img src="/images/2023-01-14/mfc_weak_hard.png">  

When CRYPTO-1 was completely reverse engineered in 2008 by some dudes from the Radboud University in the Netherlands ([Dismantling MIFARE Classic](https://www.sos.cs.ru.nl/applications/rfid/2008-esorics.pdf)), the so called ``CRAPTO-1`` library was released as open-source counterpart to the original, which opened the door for several tools and attacks against MIFARE tags implementing the broken crypto.  

## Nested Attack

When the PRNG is detected to be ``low`` the keys can be retrieved by the so called ``nested`` attack. It's an offline attack, meaning we only need the tag and no reader.  
The only other prerequisit is that we need to know at least one valid key to no matter what sector.  
It is a timing attack against the so called ``L``inear ``F``eedback ``S``hift ``R``egister - ``LFSR``, and allows an attacker to calculate valid keys based on one known key.  
A good explaination can be found in this [article](https://securityguill.com/nfc_mifare.html).  

What we basically do is with e.g. a known key A of block 0, we can recover key A of block 4.  

This attack is sometimes refered to as the ``MFOC`` attack, but the ``M``I``F``ARE Classic ``O``ffline ``C``racker is just the name of a [tool](https://github.com/nfc-tools/mfoc), that implented this (and later also the hardnested) attack.  

### Flipper

Well, the good ol' dolphin is not capable of doing things like this.  

### PM3  

Let's take our example of the weak tag, where we ended up with this, after trying the default keys:  

<img src="/images/2023-01-14/pm3_nested1.png">  

Sector 1, which consists of blocks 4-7, can be accessed with the ``A`` and ``B`` key of ``FFFFFFFFFFFF``.  
According to the intro, we are now able to use key ``A`` of block 4 to derive the ``A`` key of block 8 which is the first one in sector 2:  

```
hf mf nested --blk 4 -a -k FFFFFFFFFFFF --tblk 8 --ta
```

Et voila, we got the next ``A`` key:  

<img src="/images/2023-01-14/pm3_nested2.png">  

This ping-pong can be continued, until we have recovered all keys.  

For you lazy ass script-kiddies the mf3 software contains an ``autopwn`` feature, which will automagically determine what tag it is dealing with, and carry out all attack steps needed to pop it open:  

```
hf mf autopwn
```

<img src="/images/2023-01-14/pm3_autopwn.png">  

<img src="/images/2023-01-14/hackerman.png"> 

<img src="/images/2023-01-14/pm3_cracked.png">  

The data can now be dumped, manipulated, emulated, copied - you name it.  

## Darkside Attack or Key Stream Recovery Attack

<img src="/images/2023-01-14/vader_meme.png">  

This attack is where you most probably want to start of, if you have no key at all. The attack will allow you to retrieve a valid key A or B.  
If we don't have at least one key but are dealing with an exploitable tag, we can switch to the so called [darkside](https://eprint.iacr.org/2009/137.pdf) attack. This again abuses flaws in the PRNG implementaion of CRYPTO-1 chained together with according error responses that eventually lead to leaking bits of the keystream which allows to recover actual sector keys. As this attack can take a hell lot of time, you most likely want to only recover one key, and than continue with the nested attack.  
As far as I understood - and I hate math and am too dumb for it - something like this happens during the cipher initialization phase:  

<img src="/images/2023-01-14/Authentication+&+initialization.jpg">  

- Tag and reader each chose and exchange random nonces ``Nt`` and ``Nr`` respectively  
- Some mathematical operation based on these values is taking place  ¯\\_(ツ)_/¯  
- Some (8 in total) parity bits are checked by the tag before the actual results of the mathematical operations are validated  ¯\\_(ツ)_/¯  
- If all 8 parity bits are incorrect the tag won't answer the reader  
- If all 8 parity bits are correct, but the rest is incorrect, the tag will respond with a 4-bit error-code of ``0x5`` also known as ``NACK`` indicating a problem during transmission  
- As this error code is of a fixed value an attacker can recover some parts of the keystream when this error occurs  

Please read slides no. 14 and 22 of the [Blackhat talk from 2014](https://www.blackhat.com/docs/sp-14/materials/arsenal/sp-14-Almeida-Hacking-MIFARE-Classic-Cards-Slides.pdf) and the before mentioned whitepaper to (maybe) fully understand it.  

Some people also refer to the attack as ``MFCUK attack`` . However, ``MFCUK`` stands for ``M``I``F``ARE ``C``lassic ``U``niversal tool``K``it, and is the name of a tool build around this attack. It is part of the ``nfc-tools`` github repo and can be found [here](https://github.com/nfc-tools/mfcuk).  

### Floppy

<img src="/images/2023-01-14/flippercant_meme.png"> 

### PM3

Unfortunately I don't have a tag that is prone to this type of attack. But the syntax again is simple.  

``
hf mf darkside
``

<img src="/images/2023-01-14/pm3_darkside.png">  

Ofcourse you can use ``autopwn`` here as well.  

##  Hardnested Attack

If you happen to stumble upon a MIFARE Classic tag with a good PRNG, you can still attack it offline with the hardnested attack. The [technical details](http://www.cs.ru.nl/~rverdult/Ciphertext-only_Cryptanalysis_on_Hardened_Mifare_Classic_Cards-CCS_2015.pdf) are again proudly brought to you buy the dutch guys.  
Unfortunately a prerequisit here is, like with the normal nested attack, that you need to have at least one known key to carry out the hardnested attack. Besides that, the attack flow is the same.   
I wasn't able to understand anything from the paper, nor was I able to find some highlevel stuff explaining it for dummies. All I know is that we are again talking about flaws in CRYPTO-1. Feel free to enlighten me if you can explain it like I am 5.    

### Flopper  

Nope

### PM3

Same syntax as with the nested attack, however ...

... drum roll ...

... this time we write ``hardnested`` instead of just ``nested``.  


```
hf mf hardnested --blk 4 -a -k FFFFFFFFFFFF --tblk 8 --ta
```

Awesome!!! 

<img src="/images/2023-01-14/pm3_hardnested_start.png">  
<img src="/images/2023-01-14/pm3_hardnested_result.png">  

From here redo for the next sector.  

And yes you can use ``autopwn``.  

##  Reader Attack  

Your last resort if all the above mentioned attacks aren't working.  
In this scenario we piggyback or emulate a valid tag to collect nonces directly from communication with a valid reader.  
The collected nonces allow an attacker to calculate keys out of the bit stream, and if we recover one key -> you by now know what to do.  

### Flipper

Hey he is back. Well, at least theoretically. I didn't get the attack working with the setup I had.  
Nonces can be collected by either emulating a random tag or, and this is recommended, by emulating a cloned tag. This also seems to be the problem, as the Flipper currently is not capable of fully emulating every MIFARE Classic tag available out there it seems.  
See [here](https://twitter.com/theluemmel/status/1614928615040483334) for some references.  
If you can emulate a tag successfully, it should go like this:  

<img src="/images/2023-01-14/Detectreader.gif"> 

When approaching the reader, your screen will change to something like this:  

<img src="/images/2023-01-14/flipper_collect.png">  

It will most likely ask you to move the Flipper away and then re-approach the reader several times, until enough nonces were collected.  
This will result in an ``.mfkey32.log`` hidden file in your nfc folder:  

<img src="/images/2023-01-14/flipper_mfkeylog.png"> 

This file can now be parsed with the help of the awesome [mfkey32](https://github.com/equipter/mfkey32v2) tool, which offers a variety of ways to do it, which will ultimately give you some keys to start with.  

### PM3  

The Proxmark3 will automagically try to extract keys from sniffing communication between a real tag and reader, and as far as I know this is also implemented with the mfkey32 stuff.    
You want to sandwich your tag, PM3 and reader to get the best results, because we are actually sniffing the real communication (it should also be possible with the emulation mode I guess):  

<img src="/images/2023-01-14/pm3_sandwich.png"> 
<img src="/images/2023-01-14/pm3_sandwich2.png"> 

As you can see, I unscrewed the LF antenna as well as the two "front" screws, so I can place the tag closer to the HF antenna and approach the reader as close as possible.

```
hf 14a sniff -rc
```
Press PM3 button to stop the sniffing

To show the trace and apply the mifare decoding to it we run
```
trace list -1 -t mf
```
<img src="/images/2023-01-14/pm3_tracesniff1.png"> 

If you are lucky, the trace will give you keys:  

<img src="/images/2023-01-14/pm3_tracesniff2.png"> 

If you happen to be on some kind of engagement, you can save the dump for later usage to disk, because if you turn off the PM3, everything in memory will be lost.  

```
trace save -f dump
trace load -f dump.trace
```

## Tamper Data

But what if you were only able to steal the tag of some low privileged user you might ask.  
Well in certain circumstances you might be of luck, and the encrypted data will reveal you what to do, like being able to tamper some ID that is stored in sector 5 of whatever.  
Once you have access to all the data in the tags sectors, you can dump the content, tamper it, and then for instance emulate the tampered tag or write it to a magic tag.  

### Flapflop  

Well, as the little dolphin wasn't able to emulate MIFARE tags in my case, I didn't follow this trail in depth.  
From the device's menu you can't tamper the data at all.  
But, if Flipper was able to recover all keys, you can access the data in the according ``.nfc`` files on the SD card, tamper the values, write it back to Flipper, and emulate the tag.  

### PM3

If we successfully dumped a tag's content, we can tamper the data in the emulator's memory directly:  

```
hf mf esetblk --blk 1 -d 31323331323300000000000000000000
```

The data you provide is ``HEX``, so in this case ``31323331323300000000000000000000`` in text is ``123123``.  

Once changed, we can either emulate the "new" tag or write it to another tag, in this case a magic tag:  

```
hf mf sim --1k
or 
hf mf esave -f filename
hf mf cload -f filename.bin
```

# Conclusion / Defenses

This blog post was just an introduction to a small subset of stuff related to the whole RFID / NFC world. There is much more to test, experiment with and pwn. A cheap PM3 Easy or a Flipper can be good friends in the beginning. 
I had a lot of fun testing things out, and wan't to especially thank my current company and my boss, that allowed me to also "play around" with these tools and technology during my work-time.    

- If you want to be on the safe(r) side, use more advanced technology like MIFARE DESFire, that is not prone to the attacks described here.  
Attacks against these hardened tags may be relay attacks. Please find these resources to further get along:  
https://github.com/nfcgate/nfcgate  
https://www.usenix.org/conference/woot20/presentation/klee  
https://www.cs.tufts.edu/comp/116/archive/fall2017/mluo.pdf   
- Use setups that detect magic tags, so if an attacker was able to dump your card, but only tries to enter your building with a clone, you can detect him at the gate. Be aware, that there are ways around this with the availability of magic gen 2 tags.  
- Use setups with tamper protection. There are attacks that use implants inside the legit readers to sniff and dump data. You want an alarm popping up when this happens.  
- Train people to not expose their tags when not needed. Wear your badges hidden when not on the campus, and be aware that these things are / can be like keys to the kingdom's heart.  
- Don't rely on things like vendor statements and marketing bullshit! Get your stuff tested and challenged - regularely.  

# Ressources  

[https://www.abr.com/what-is-rfid-how-does-rfid-work/](https://www.abr.com/what-is-rfid-how-does-rfid-work/)  
[https://www.abr.com/passive-rfid-tags-vs-active-rfid-tags/](https://www.abr.com/passive-rfid-tags-vs-active-rfid-tags/)  
[https://www.techtarget.com/iotagenda/definition/RFID-radio-frequency-identification](https://www.techtarget.com/iotagenda/definition/RFID-radio-frequency-identification)  
[https://wlius.com/blog/rfid-vs-nfc-whats-the-difference/](https://wlius.com/blog/rfid-vs-nfc-whats-the-difference/)  
[https://jjensn.com/mifare-easy-targets/](https://jjensn.com/mifare-easy-targets/)  
[https://securityguill.com/nfc_mifare.html](https://securityguill.com/nfc_mifare.html)  
[https://smartlockpicking.com/slides/Confidence_A_2018_Practical_Guide_To_Hacking_RFID_NFC.pdf](https://smartlockpicking.com/slides/Confidence_A_2018_Practical_Guide_To_Hacking_RFID_NFC.pdf)  
[https://eprint.iacr.org/2009/137.pdf](https://eprint.iacr.org/2009/137.pdf)  
[https://www.blackhillsinfosec.com/rfid-proximity-cloning-attacks/](https://www.blackhillsinfosec.com/rfid-proximity-cloning-attacks/)  
[https://www.blackhat.com/docs/sp-14/materials/arsenal/sp-14-Almeida-Hacking-MIFARE-Classic-Cards-Slides.pdf](https://www.blackhat.com/docs/sp-14/materials/arsenal/sp-14-Almeida-Hacking-MIFARE-Classic-Cards-Slides.pdf)  

# Acknowledgement

Big shoutout to all you awesome people sharing knowledge and tools:  
[iceman](https://twitter.com/herrmann1001) for his awesome work in so many aspects of this field  
[jjensn](https://twitter.com/paprikasunday) for his cool blog  
[Guillaume Leplat](https://twitter.com/SecurityGuill) for his cool blog   
[Sławomir Jasek](https://twitter.com/slawekja) for one of the coolest slide decks ever  
[Ray Felch](https://twitter.com/llcoder) for the cool post at BHIS  
[Márcio Almeida](https://twitter.com/marcioalm) for his cool BlackHat talk  
And all the ones I forgot to mention of where I couldn't find enough info.  

Stay safe and happy pwning.  
LuemmelSec
