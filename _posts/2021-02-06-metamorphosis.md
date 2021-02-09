---
layout: single
title: "How a SysAdmin becomes a Pentester"
excerpt: "What I did, and what you can do, to become a pentester 31337"
date: 2021-02-06
header:
  #image: /assets/images/NULL
  thumb: /assets/images/sysadmin2pentester/finn_legit.jpg
  teaser: /assets/images/sysadmin2pentester/finn_legit.jpg
  teaser_home_page: true
classes: wide
categories:
  - Starting in Sec
tags:
  - Pivoting_into_security
  - pentesting
  - web exploitation
---

![Hot minute](/assets/images/sysadmin2pentester/alarm-flame-260nw-117772630.jpg)

# It's been a hot minute, and I am sure no one reads these - but it is important for me to reflect on what has been so I know where to go...

__This post will be broken up into three parts:__  
  * a brief synopsis of how I got here from where I was
  * What a SysAdmin (in my opinion) needs to do in order to be competitive as a pentester.
  * An insight look at an actual pentesting scenario 


### To quickly summarise my exposure to IT before pivoting to Sec...
...I have close to 10 years under my belt - which may SEEM like a lot, however most of the time was spent on Helpdesk, incident management and pressing buttons in a sequence to patch large scale environments. Only 3 of those years were spent accumulating hard technical skills, starting when someone took a chance by giving me an opportunity in I would actually call my first _real_ job in IT as a Network Technician. This job, after 7 years finally ignited my passion for tech, up until then it was "just a job" and I was always looking for something better. (Thank you Michael AKA BIG MICK)

During those three years I quickly progressed to being a System Engineer at another company, and shortly after was contracted out to do SysAdmin work at an Airport, where I learnt A LOT.  
I genuinely believe that the reason I progressed so fast was because I was studying for my [OSCP](https://www.offensive-security.com/pwk-oscp/) alongside this work period, this is a whole separate point of discussion and I already have a writeup [HERE](https://kymb0.github.io/zero-2-OSCP/) which includes a more detailed breakdown of all the stuff I just rambled on about.

## So let's begin

Towards the tail end of my OSCP attempts, I could feel that it would not be long until I passed. That, coupled with dissatisfaction with my current employer led me to start scoping out Junior Security roles.  
Now this, is where it gets interesting.  
There seems to be some perpetual cycle where many places do not want to hire Juniors, as it cost money to dedicate seniors to help them NOT be juniors.
But no one wants to employ juniors... so the market is therefore starved for seniors... So the only option is to hire juniors... But there are not enough seniors to train the juniors...  
And thus I realised that to get into this industry, it was going to take more than simply handing out CV's that said "ME ALMOST HAVE OSCP, ME KNOW HOW 2 HAK PLS HIRE ME"  
And so I got in touch with recruiter who is an absolute legit legend of a bloke (whose name/company I will not put in here unless he permits it).  
Although this guy did not get me my first job in Sec _directly_, I would not have had a chance to work for the company I am at now if it wasn't for this guy.

Initially, as I did not yet have my OSCP yet meant getting interviews was tough for the recruiter, something he was honest about. Nevertheless he did try, but it wasn't until I finally passed that he was able to get an interview at a rather large company. By this time I also had my blog going (mainly for the purpose of submitting [SLAE32](https://www.pentesteracademy.com/course?id=3) assignments) which also made me slightly competitive as a junior candidate. I completed 2 rounds successfully (according to feedback was at offer stage) however due to a merger/covid, recruitment (for Juniors) was frozen. The fact I was able to secure an interview in the first place was confidence-boosting however, and it also gave me an opportunity to be invited to the [sectalks](https://www.sectalks.org/) slack group - something I was not even aware existed.  

During the 8 or so months it took to hear back from the larger company, I continued to work on my blog, upskill via any online resource I could find (will link at end of this post), engage actively with the community via entering study groups and posting my blogs, and participating in CTFs.  
All of these steps were crucial to giving me the exposure, inside knowledge, skills, and connections needed to both secure and pass the interview for my first pentesting job.  
Funnily enough around the same time this opportunity came up, the ORIGINAL company now fully merged requested a final interview, which I felt I did quite well at however they could not go through with the offer due to my lack of experience(?)  

As things are meant to be however, the company I work for now was more than happy to bring me onboard for a rather large and complex long term project after just one interview. Now, I think it is important to consider that it really was not just the hours worth of time that I put in during this one interview, this was simply the final piece of 8 months of hard, focused work. I cannot stress the importance of community engagement enough - even if you lurk in any chats or forums you can find - you WILL learn.  

## cp /home/General_IT_person/skillset/* > /home/pentester/

This part is directly aimed at traditional SysAdmins who want to move into Offensive Security at a __competitive__ level.  
__Step 1:__ Unless you are already tearing through HTB and have years of CTF experience under your belt, you have to start studying for your OSCP right now, yes it is an entry level certificate, but it is entry level to one of the hardest tech jobs out there, and you need the experience of the 24 exam to help train your mind.  

__The rest:__ While we have our own skillset and knowledge of networks, business needs/stakeholder engagement as well as core AD knowledge - we are at a massive disadvantage, and that is: We are not devs, and as such we do not possess the mindset of a dev, which let me tell you a seasoned dev will navigate through problems faster than you can say "let me google it". The are professionals at being in situations where they do not know/understand the deeper machinations of a technology, and bringing themselves up to speed in a short amount of time - it's kinda their job.  
Many times I have worked with colleagues who think that is acceptable to solve an issue by running someone else's command/code or following the steps they found on a forum.  
![WRONG](/assets/images/sysadmin2pentester/no-that-is-wrong.jpg)  
This leads to broken and insecure environments. It may expand your repertoire of "fixes",but not you knowledge.  
When you understand how things work, you can break/subvert/navigate around them.  
This also leads me to my next pint, which will be controversial among some people...  

:clap:you:clap:must:clap:learn:clap:to:clap:code:clap:

You have a LOT of catching up to do (I am reminded of my skillgap every day), and being able to fix broken exploits ALA OSCP is simply not good enough, an example why will be provided in the last section of the post.  
I recognised this early on, and put myself through a beginners Python course first, before starting online coding challenges and writing tools from [Black Hat Python](https://nostarch.com/blackhatpython). In addition to this I took a course on Linux assembly which catapulted my understanding of program logic at a deeper level. From here I have dabbled in various other languages to the point I can pick up almost any language and _slowly_ build something adhoc.  

In addition the above, you are going to have to become comfortable with web to the point you know enough to learn on the job. I have never been to Uni or done any real IT course so for me this took a while, I started by creating my own webserver, which I then turned into a [Vulnhub submission](https://www.vulnhub.com/entry/stripes-1,468/). This taught me how each component in a stack works, it's purpose, how they interact, and how they work.  
Next targeted specific attacks, I would spin up [DVWA](https://github.com/digininja/DVWA) and use online challenges to go hard against XSS, SQLi etc until I not only knew how to trigger them but could read existing code and know WHY it worked and WHERE to inject. This is a necessity IMO.  
The last thing I did ties both this and the coding paragraph together: I wrote a [very, very basic](https://github.com/kymb0/General_code_repo/blob/master/Code_templates/bypass_csrf_into_sqli.py) exploit in Python to launch against DVWA.  
(As a side note I DID also begin the [AWAE](https://www.offensive-security.com/awae-oswe/) course, however after stopping for the third time to do a deep dive into a particular subject I decided that while the short amount I had worked through was invaluable, I would revisit it at a later stage)

#### This might seem like a lot but once you get the ball rolling thing start falling into place faster and faster, just keep going. Do not look at this as one entire process, look at it as completing levels until you reach the boss.

## An example of why all this is important:

So our client goes through a particular 3rd party for one of their applications, and during the PEN-test one of our seniors discovered that an API endpoint was vulnerable to SQLi do to [unparameterised](https://cheatsheetseries.owasp.org/cheatsheets/Query_Parameterization_Cheat_Sheet.html) query sent via JSON.  
He proved the compromise and provided the reproduction steps (again, both for proof and so the devs can test their fix) as well as what, in his opinion should be done to mitigate the SQLi - and this most certainly included the use of parameterised queries.  

Now in this situation a RETEST is required, to test the specific vulnerability found and make sure the vector is patched up, and that whichever fix has been deployed does not open any other attack vectors.  

And so a few weeks later I was told I would be the one to retest. I looked through the original job, made sure I could access the endpoint, reached out to the Project Manager to ensure that on the day of testing everything would be ready for me to go, and I would not have to chase anything up.  
I started the test by immediately following the documented reproduction steps, as is obvious, if these are reproducible to the same affect then obviously nothing has been done.  

The steps did NOT work, in fact, all I observed in burp was that EVERY response was a `500 Internal Server Error`... Hmmm.  
So the server is no longer providing any verbosity in it's response, but it IS telling me that the server came back with an error.  
So I decided to see if I could get something other than a 500/prove code execution. The only real way to do this in this scenario is by trying to make the server timeout by injecting either `SLEEP` or `WAITFOR` or, well, you get the idea.  

Lo and behold, the below caused the server to timeout with a `504 gateway Timeout`: 
```
{
  json_val_1 : "foo"
  json_val_1 : "bar"
  json_val_1 : "foo"
  json_val_1 : "bar' waitfor delay'0:0:10'--"

}
```
![zzzzzzz....](/assets/images/sysadmin2pentester/wake_up_m8.jpg)  

Now, why is receiving 2 different responses a big deal?  
Because it means that to a certain degree I am now in control.  
My previous deep dive into sqli led me to the conclusion that I am now dealing with something called [Blind SQL Injection](https://owasp.org/www-community/attacks/Blind_SQL_Injection).  
It is pertainent to note at this point that all the devs has seemingly done at this stage is _break the reproductions steps so that the security testing is a pass and their product can be released_

### Weaponising Server Responses
Now, the cool part:  
Because whoever was responsible for patching this vulnerability up did not do so efficiently/correctly as per OWASP - I was now able to come up with proof of concept exploit, in which truth in a query is __infered__ via the fact I control whether or not the server responds with a `504`.  
This may seem confusing, but consider the following:
If we send the same payload in JSON as before, but with some added code, we can either prevent or cause the server from timeing out with something called BOOLEAN logic, which very basically means: `TRUE` vs`FALSE`, or, `1` vs `0`.
We can say, "if 1=0", or if FALSE, do nothing.  
And adversely we can also say "if 1=1", or if TRUE, wait for 10 seconds thus causing the server to timeout.

#### FALSE, NO TIMEOUT (because the first condition is not met, the second part of the query is not sent)  
```
{
  json_val_1 : "foo"
  json_val_1 : "bar"
  json_val_1 : "foo"
  json_val_1 : "bar' if 1=0 waitfor delay'0:0:10'--"

}
```
#### TRUE, TIMEOUT (because the first condition is met, the second part of the query is sent)  
```
{
  json_val_1 : "foo"
  json_val_1 : "bar"
  json_val_1 : "foo"
  json_val_1 : "bar' if 1=1 waitfor delay'0:0:10'--"

}
```

Now, this may seem arbitrary, but what if instead of saying `if 1=1`, we say, `what if the first character of ADMINISTRATOR's password begins with A`?  
What we have now is the foundation for beginning a bruteforce attack, we can write a Python script that will asses the response form the server - if it is a `504`? guess what - we have the first character of ADMIN's password, then we send `AA`, then `AB` and so on, until we have a complete string.  
This is a very contrived example however this is the exact logic I used to SHOW impact to the business.  

### Retest no.3

So, after all this, and after pretty much the same vuln being found on the first retest you would think the devs in question would pull their socks up and fix the issue correctly, wouldn't you?

No such luck.  

Long story short I was able to prove that all they had done was mitigate my particular attack through string matching/regex - which is a big no-no.  
Unfortunately I was unable to re-exploit in my allocated time (pentesting is allllll about what you can achieve in the time you have). This however put me in a bit of a predicament, as I was unable to re-exploit the endpoint, but was not confident it had been fully patched.  
This is where experience dealing with Project Managers/stakeholders/the non-technical side of the business can be utilised - I was able to communicate the situation in a manner that was non-abrasive yet concise, I essentially suggested that the business ask to actually see the code and steps taken to prevent the vuln, and if parameterisation was not being used, that it should not be deemed fit for release.


The last part of my blog regarding the SQLI and testing/retesting was a bit long winded I know, but I hope that the scenario in it's entirety shows both how valuable the soft skills gained form working on System's can be, as well as the need to _understand_ what you are doing along with a grasp on programming logic.

If you have made it this far - you have reached the end of this post, thank you for getting the whole way through - and I hope that it has helped illuminate your path to becoming a pentester.

See you guys soon for another post - in the meantime I will continue to level up so I can dorn this fancy +5 mace I found :)))))

![war3z](/assets/images/sysadmin2pentester/finn_legit.jpg)  
