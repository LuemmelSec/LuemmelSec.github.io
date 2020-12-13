---
layout: post
title: AS_REP Roasting vs Kerberoasting
---

Recently my team had a discussion about what the exact difference between **AS_REP Roasting** and **Kerberoasting** is.  
As we were short of time, we did not come to a concrete answer and were also not able to find an article that would explain it in short.  

I am neither a professional with years of experience nor a Kerberos guru. So if you are looking for a complex deep-dive, feel free to move along.  

## Introduction

Kerberos is a protocol used to authenticate parties with the help of tickets over a non-secure network.  
It´s a really interesting topic and so I used our discussion as a starting point to dig a little deeper into both attacks and came up with this very high-level write-up comparing the two scenarios.  
The protocol (at least to me) is very complex, and the write-ups I found were never 100% accurate in terms of terminology and clarification of things. You may read that messages are partly signed with a secret-key or with a users password hash or even just with the password. To make things worse this might happen in one single article. So I am quite sure that there will be some mistakes here in my write-up as well, but the basics should be correct.  

Although Kerberos is not a native Microsoft protocol, one will most likely find it in Windows Active Directory environments. As this blog is aiming at Microsoft AD pentests, I will stick to this scenario here, which also implies that all the extended Microsoft features like pre-authentication are present.  

## TL;DR

**AS_REP Roasting** is taking place during the initial authentication procedure within Kerberos. It´s abusing the fact, that for accounts where the option **Do not require Kerberos preauthentication** is set, there is no need to send the (normally required) encrypted timestamp (with the users password hash) at the very beginning. Thus everyone on the network who knows the name of an affected account may ask the **KDC** to authenticate as that user and in response fetch a **AS_REP** response which partly is encrypted with the AS_REP roastable accounts password hash. Once obtained, an attacker can try to offline recover the cleartext credentials.  

**Kerberoasting** aims at asking to use services on the network where the **SPNs** are tied to user accounts, rather than computer accounts. These accounts by nature will most likely have been created by a human being including a password. If that password is weakly chosen or the attacker has lots of lots of time or computing power, then it is possible to recover the cleartext credentials here.  
During the process of asking to access a service on the network, the **TGS** will send a data package that contains a service ticket which is encrypted with the service-accounts password hash, that again like in the **AS_REP roasting** attack can be cracked offline.  

## Terminology

The following abreviations will be used along this article. If you need more detail I recommend looking at the [ldapwiki](https://ldapwiki.com/wiki).

KDC: Key Distribution Center - The trusted 3rd party / the Domain Controller  
TGS: Ticket Granting Server - A subset function of the KDC which issues Service Tickets  
TGT: Ticket Granting Ticket - A userbased ticket used to authenticate to the KDC  
ST:  Service Ticket - Used to authenticate against services  
SPN: Service Principal Name - The name of a service on the network  

## AS_REP roasting

The Kerberos authentication process looks like the following (high level - there is more going on in in detail and parts are missing):
1. Login to a system with username and password  
2. [AS_REQ](https://ldapwiki.com/wiki/AS_REQ) request containing the username, desired service (in this case krbtgt) and a timestamp encrypted with the users password hash is sent to the **KDC** asking for a **TGT**  
3. **KDC** checks if he can:  
   a) decrypt the timestamp with the hash he has for that user in his database and  
   b) if the timestamp is within a certain period of time compared to the servers current time (5 minutes default)  
4. If valid and within the allowed time period the **KDC** returns a [AS_REP](https://ldapwiki.com/wiki/AS_REP) response, which amongst others contains the **TGT** and an encrypted session-key, which can be decrypted with the users password hash.  
5. Client decrypts the session-key and saves the TGT for later usage  

Now with the option **Do not require Kerberos preauthentication** set, the timestamp stuff is disabled for this account and only the default Kerberos cleartext message is needed to ask for a **TGT**.  
This results in any user who has the correct name of that account to be able to request the **TGT**, which in return will be send as part of the **AS_REP** - again with parts encrypted with the users password hash we were asking for. This info can then be used to try to recover the cleartext password.  

## Kerberoasting

**Kerberoasting** aims at asking for a [ST](https://ldapwiki.com/wiki/Service%20Ticket) for a service that is tied to a user account and will most likely contain a human generated password.
Normally [SPNs](https://ldapwiki.com/wiki/ServicePrincipalName) are bound to computer accounts which have a random 128-character password that automatically gets changed every 30 days.

With a valid **TGT** in hand, an attacker may request a **ST** for every **SPN** on the network.
The flow is as follows (involving the steps from the AS_REP roasting section):  
1. With a valid **TGT** a [TGS_REQ](https://ldapwiki.com/wiki/TGS_REQ) request is send to the [TGS](https://ldapwiki.com/wiki/Ticket%20Granting%20Service)  
2. The **TGS** checks if the **SPN** is valid, opens the **TGT** and does some additional tests to it  
3. If everything is okay it generates a **ST** which it encrypts with the service-accounts password hash and sends it back to the client as part of the [TGS_REP](https://ldapwiki.com/wiki/TGS_REP) response
4. The client receives the response, extracts the **ST** and can forward it to the desired service to access it

The problem here lies in the fact, that the **ST** is encrypted with the password hash of the **SPNs** account.  
The attacker can intercept or extract the ticket from memory, and crack the hash offline.  


## Conclusion

**AS_REP roasting** is taking place at the very beginning of the Kerberos authentication procedure. An attacker would only need physical access to the network, but would also have to know the principal name of the account he want´s to ask a TGT for. The option **Do not require Kerberos preauthentication** for the object needs to be set and as such is less likely to be found during an assessment nowadays.

**Kerberoasting** is abusing a normal function inside an environment that makes use of Kerberos, which is the fact that the service tickets are encrypted with the hash of the **SPNs** account. If this password is weak an attacker will most likely be able to recover it. As a prerequisit one needs to have valid credentials and a useraccount to ask the **KDC** for a **ST**.


If you think that I made mistakes, that things are still unlcear or you are missing things, please feel free to hit me up on [twitter](https://twitter.com/theluemmel).  

Special thanks to my awesome collegues [S3cur3Th1sSh1t](https://twitter.com/ShitSecure) and [0x23353435](https://twitter.com/0x23353435) for their input and support.  

I also want to thank the people who already did all the thinking and who wrote down and shared their knowledge:  
[Aidan Preston](https://twitter.com/m0chan98)  
[Sean Metcalf](https://twitter.com/PyroTek3)  
[Will Schroeder](https://twitter.com/harmj0y)  


If you are interested in further info, here are some links that might help:  
[https://zeroshell.org/kerberos/kerberos-operation/](https://zeroshell.org/kerberos/kerberos-operation/)  
[https://www.qomplx.com/qomplx-knowledge-kerberoasting-attacks-explained/](https://www.qomplx.com/qomplx-knowledge-kerberoasting-attacks-explained/)  
[https://www.harmj0y.net/blog/redteaming/kerberoasting-revisited/](https://www.harmj0y.net/blog/redteaming/kerberoasting-revisited/)  
[https://www.harmj0y.net/blog/activedirectory/roasting-as-reps/](https://www.harmj0y.net/blog/activedirectory/roasting-as-reps/)    
[https://m0chan.github.io/2019/07/31/How-To-Attack-Kerberos-101.html](https://m0chan.github.io/2019/07/31/How-To-Attack-Kerberos-101.html)  
[https://iam.uconn.edu/the-kerberos-protocol-explained/](https://iam.uconn.edu/the-kerberos-protocol-explained/)  
[https://adsecurity.org/](https://adsecurity.org/)  