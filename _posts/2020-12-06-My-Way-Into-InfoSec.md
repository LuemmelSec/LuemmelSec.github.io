---
layout: post
title: My Way Into InfoSec
---

This is my very first blog post ever, which I am trying to use to get a little into github (pages), and because I was in the mood to write something.  
As I am fairly new into being a fulltime InfoSec guy, I´ll be writing about how I got into it and how I landed my current job as a pentester.  
This will also reflect my point of view regarding **the right mindset** and **certifications** that might get you started.

## A little introduction
 
As the time of writing I am 37 years old. I´ve been working in the IT business since I was 17.
I had a job as admin for about 8 years which partly got me in touch with IT-security (Mcafee ePO, Sophos Firewalls, macmon NAC and stuff), before joining my current company like 2,5 years ago.  
Here I was working as a Senior Security Engineer (meaning that I worked with endpoint security related stuff) for over 2 years, and was recently offered the chance to join our pentest-team in the position as Security Consultant.  
*Why everyone is naming job positions to their likings is a mystery I will never be able to understand.*  

So here I am. A total noob regarding penetration-testing, doing stuff that others get arrested for legally and getting payed for it. Or in other words:  
**I finally have the job I always wanted** (at least I believe so).

## Intro to pentesting

During my last job I often got asked: "Why did AV not catch that?" or "How was it possible that...?", and I had no clue. So I started reading about how malware is trying to circumvent AV measures, and how one was able to set countermeasures. I also digged a little into securing Active Directory and stuff, and set up a little lab to play arround. I used metasploit modules to scan our productive environment for systems exploitable by MS17-010, what got me into Kali and all the involved tools.
When I was at home I started to play around with wifi-pentesting - ofcourse only attacking my own wifi ^^  
Reading about and doing this kind of stuff really made me want to learn more, and it was the first time I knew exactly that if I would be able to do this as a fulltime job, that would be awesome. I would become the bad guy amongst the good guys - badass.  

So I booked me some udemy courses like "Becoming a white hat hacker in 30 days" or "Ethical Hacking for everyone", watched some videos regarding vulnhub machines and followed along with my VirtualBox Kali.

The problem I was facing was, that if I didn´t focus and repeat, I was loosing basics over and over again, what really was a drawback each time. When I had to pause for some weeks, I felt like starting from ground up. During this time we moved to a new house, and for almost two years I didn´t touch a computer in my sparetime.

Two years later it got me again and I wanted to change something regarding my job. On my way to work there was a company which solely focusses on IT-security. So I thought to myself: Give it a try.  
A few month later I had a new job. Only downside: They wanted me to staff up their Endpoint-Team (as I had worked with McAfee for the last 8 years) and had no ressources to teach a noob like me on how to pentest. But to me it was the first step in the right direction. Maybe I could still learn something, get involved in some projects or whatever. At least I was as close to pentesting as I could possibly be for the moment.

## The journey begins

The pentest guys in my company were meeting once a month after work to do some [Hack The Box](https://www.hackthebox.eu/) machines, and just having a good time together.
So I asked if I could join and just watch them do all the magic things. And although I didn´t even understand half the shit these crazy fuckers where doing, it felt great just hanging around with them and at least try to soak up some of their knowledge and be part of the gang.

In parallel two of my colleagues, that where working in my team, prepared for their OSCP exam which the company was paying for, because they also liked to dig a little deeper into this world and the knowledge would also help the companies non-pentesters with their daily doings.

*WTF - this is great. My company is paying an OSCP - the exam that everyone is talking about? Sign me up please, this is my chance. I already told you at the job interview that I would love to push myself into this direction.*

## OSCP

The company agreed and I was allowed to also take the course.  
I didn´t want to start from scratch again when beginning the lab-phase of the OSCP, and so decided to do a 6 month selfpaced learning with the help of HTB, and when I felt confident enough start my lab-time.

I began in June 2019, and read a lot of **OSCP like machines** posts on the internet. I solved about 30 boxes, retired and active ones. The biggest inspiration here were my colleagues and of course the infamous [ippsec](https://twitter.com/ippsec), who shares his knowledge like no one else. I wrote down anything that seemed to be of interest, so that I had everything in one place, organized to my needs.

My 90 days of lab-time started in January 2020.  
I decided to first go through all the materials and solve all tasks as well as documenting the pwnage of 10 unique machines in the lab, to get my 5 bonus points. That alone cost me 15 days of hard work. I didn´t touch any of the lab-machines until then.
During my journey a colleague from the pentester-team was doing the labs as well. The two guys from my team had just passed their exam, and another guy from the pensters had finished his a few month ago. So I used every moment I (and probably they) had to have them explain things to me when I got stuck. I visited the pentest-team during my lunch breaks, and they helped me out, always trying to just push me in the right direction. We had a WhatsApp-Group where I was constantly asking questions, trying to understand even the last detail.  

Together with these awesome people I was able to clear the whole lab about one week before my lab-time ended, and pass my exam on April 18th 2020, clearing 4/5 machines. I was completely lost with one of the medium-difficulty machines, beacuse I found no way in. All others were rooted.

I wrote down every minute I spent learning during this journey, ending up with a total of 460 hours of learning effort, all done in my spare-time, after work and on weekends. 

Afterwards I said to myself: *Dude that was it for this year, and also for the next 200 years. Time to get back to real-life.*

At this point I really want to thank my wife and apologize to my little girl. Those were some fucking hard month with very little time for my family. But they were supporting me the whole time, and without them having my back, I wouldn´t have done it.

## Welcome to the team

For some odd reason during this period our pentest-team decided to ask me how I would feel about joining them.  
*Do what? Give me a pen, where do I need to sign. I told you to put me here. THIS IS SPARTA!!!*

But for real. I think the dedication and absolute will that I showed during this time, made them even think about it. I had no practical background, am not the fastest learner, and as such this would mean a lot of time they would need to invest into me. They also had a team-member leaving the company, so I guess it was: Better him than no one :)  
But honestly I am still very unsure if they believe to have made the right decision, nor if I am the right person for the job.  

I was offered a one month trial works, which took place in August 2020. So both sides would be able to see if this is a feasible solution for all of us. Afterwards we would discuss about how a change of departments could be done and when.  
Long story short: I was assimilated and never saw my old office again.

Since October 2020 I work as a full-time pentester - mission completed. Thx to my company and special thanks to the guys that brought me here: [S3cur3Th1sSh1t](https://twitter.com/ShitSecure) & [0x23353435](https://twitter.com/0x23353435).

## Further steps and current situation

As the team mainly puts me on web-pentests or mobile-pentests, I joined snyff´s PentesterLab´s with a pro subscription to help me understand the basics and also to provide some insight to more complex scenarios. That was the best money I personally ever spent on an online-training. It gave me so much value regarding the price I payed for it.

My new team decided that it was a good idea to do Rasta Mouse´s RTO, just beacause they could.  
*Me: Damn bastards. I´m out guys. You know the OSCP thing - enough for this year, maybe next time.  
Also me: Woah this looks like the most awesome lab ever and Red Teaming is the shit <3. I´m in.*

I booked two month of lab-time. Finished the lab in two weeks. Passed my exam two weeks after. What a great job [Rasta Mouse](https://twitter.com/_rastamouse) did here.

By now the more experienced guys take me with them from time to time, when doing internal pentests, so I get used to this kind of assessments. I have a good balance between internal and external pentests and they leave enough room for me to follow along. Thank you.

But to be honest: These guys are out of this world.  
Watching them doing their job is an honour, but at the same time scares me and makes me feel very small.  
It´s hard for me to keep track. There are so many things going on in such little timeframes - always feels like getting hit by a bus, doing 60 mph, with spikes in the front grille, driven by Mike Tyson.  
Sometimes I feel I know less then I knew before the pentest.
I´m in doubt when I am responsible for a pentest, and when they leave it to me to do all the testing. I think that the customer would get much more value for his money when they would be doing the tests instead of me. They will surely find way more problems, misconfigurations and security-holes than I will ever be able to. Maybe I miss crucial parts.   
But I will not give up. The team is pushing me, and for now this is good enough. As long as I don´t dissapoint them I am fine.  

I hope that when looking back in 1 or 2 years I will laugh about this blog-post, being proud of what I have achieved till then.

## Think outside the box mindset

I read blog-posts, tweets, books and stuff to help me getting started in the InfoSec world.  
One of the basics a lot of people are telling you are a must, is the right mindset.  
**You need to think outside the box, like a hacker.**

I neither think outside the box nor do I think like a hacker.  
When doing web-pentests I focus on enumeration and try to find misconfigurations or outdated versions of software which might be or are exploitable. When I see things I stumbled upon during a HTB session or in one of the PentesterLab courses I try to replicate, but I am in no way creative.  
When developing phishing-campaigns I think about what will most likely convince people to click a link or enter their credentials. I spent time to make things look legit. But does that corelate to thinking outside the box?  
During internal pentests I mostly rely on the results of the vuln-scanner or some work of the awesome InfoSec people just published recently on twitter and stuff. There is a common toolset my colleagues use, and some things you check everytime like sensitive data in shares, passwords in AD description fields, NetBIOS turned on and so on. But again - no hacker or thinking outside of the box here.

Maybe this is something that will come to you later on your journey, or people are born this way. I don´t know. But I personally don´t think that what a lot of people are saying is a must have to get you started in InfoSec.

But what is it then you might ask.  
Well to me it´s dedication and passion. You have to love what you do and you need to have a lot of fun doing it.  
This is what will keep you on track when studying night after night for your next exam.  
This is what will keep you interested and motivated learning new stuff.  
And this is what will make you good at what you do.  
I love reading twitter posts and write-ups of people who are leading in this field. People that develop stuff, people that reverse-engineer things, people who share their knowledge on how they hacked things. I try to absorb as much as possible and keep myself up-to-date, knowing that I never will be able to be like them. But believe me, there need to be way more normal InfoSec people than gurus out there. I know where I belong and that is good as it is.

## Certifications

As I live in Germany, certs are not that much of a thing. From time to time you see job offerings that want you to be a CEH or CISSP, sometimes even OSCP, but most of the time they are just asking for a certain time of experience.
I understand that at least in bigger companies they might be in the need to pre-qualify applications. But someone without an OSCP might still be a better candidate than the one who owns it.

To me the main point with certs is, that you show your dedication to put yourself into a situation where you are out of your comfort zone. That you are willing to put a lot of effort into learning something new.

I took the most out of the lab-times during all my online-courses. It gave me much more then the stressful exams. I never had the feeling of having done something special or having learned something new when finishing the exams, although I am somehow proud of having it done.  
But maybe it´s comparable to an assessment. The fun part, where you will learn the most, is the one where you pull out the guns and shoot everything down. But what counts is your report, as this is the proof of your knowledge and value to the customer.

The [OSCP](https://www.offensive-security.com/pwk-oscp/) was something like a prestige object. Everyone starting in InfoSec want´s to own it - and so did I.  
I personally think it´s just because of OffSec´s good reputation from the past, that it is so hyped. It was the first of it´s kind 24 hour realworld, hands on test. The exam is now proctored, what I personally really liked.  
The course itself, besides methodology and basics, taught me very less I need in my current job. Not saying that I didn´t learn a lot of stuff, but I just don´t need it right now.   
What it showed was my dedication to learn towards my colleagues, and this is what opened the door for me.  
Offsec seems to also see a certain need to adopt to modern times. Right afterwards I finished, they extended the lab and course material to include some Active Directory scenarios and machines. No idea how close it is now to what you will face in reality though.  
If you need a dooropener and have the money, go ahead.

[RTO](https://www.zeropointsecurity.co.uk/red-team-ops) was the best course I ever did. Maybe because Active Directory assessments are the things I truely love.  
Awesome course material, to which you will get lifetime access. It felt like reading a good book. I never had more fun following along.  
You have to own a complete company, beginning with a phishing scenario and ending with the complete takeover of several domains.  
This really is very beginner friendly. Everything is well explained, and if you don´t want to dig deeper into certain topics, you can just stick to the course material and will be totally able to earn your exam.  
Only downside to me was, that in the labs as well as in the exam, certain things where not working as intended. You had to do the same thing like 20 times, and on the 20th attempt it would work. But Daniel just started his courses, and I think he is doing an absolute great job and will improve. Always helpful, up-to-date, and thrilling like hell. If you like Red-Teaming or even want to get into AD Security, you can´t go wrong here. I am currently not doing Red-Team assessments, but this helped me so much in understanding how things work together in the environments I face at our customers every day. Compared to what you get it is relatively cheap. If there were to be an RTO during my OSCP time, I´d go RTO all the way.

I also recommend [PentesterLab](https://pentesterlab.com/) to everyone who is starting in web-pentesting or who wants to understand a certain part of web-pentesting in detail. There is so much content you will get for your money. [snyff](https://twitter.com/snyff) like Rasta is very supportive and always willing to help. I like that there is no pressure on you. Take the time you need, earn your badges and everything is fine. If you want to pause for a month, no problem. If you are not in the need to showoff with certifications, this is the way to go.
There´s tons of topics he covers, and he´s extending all the time. From simple web basics to very complexe scenarios. If you don´t find it here I don´t know where.  He´s one of the guys where I can´t believe that someone is capable to have all that knowledge. Kudos to you Louis.

## FIN

So that´s it for my first blog-post.  
If you made it till here - congratulations.  
I hope you liked it, at least a little bit.  
And maybe it´s of some kind of help for someone who also is just starting his journey into InfoSec.

----
****
