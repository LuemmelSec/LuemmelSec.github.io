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

It´s like everytime with security tools that pretend to help you. There are some cases where a NAC comes in handy and will help you protect your network and there are other cases where it can´t do shit.