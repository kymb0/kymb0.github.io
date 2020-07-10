---
layout: single
title: "SEH primer for CTP"
excerpt: "A primer for n00bs like myself in preparation for the CTP course offered by Offensive Security"
date: 2020-07-xx
header:
  #image: /assets/images/SHELLCODING32.png
  thumb: /assets/images/SHELLCODING32.png
  teaser: /assets/images/SHELLCODING32.png
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
## A primer on Structured Exception Handling (win32 exploitation)

*This article is written to help those who want a conceptual grasp on SEH exploits, I wrote this to help myself obtain a clear understanding before my CTP course starts.*

##### There are already many fantastic articles on this subject out there (some of which are linked throughout this article), which I encourage you to consume in order to get an even better understanding. 

#### What is an exception Handler?

In simple terms, an Exception Handler is exactly as the name implies, it *handles exceptions*, that is to say, when a program reaches something outside it's known functionality, an exception is raised (eg if a program wants an input between 2 and 5, and you put in 6). The program then needs to know how to *handle* this, either with an error message, exit code, or whatever else.

A Structured Exception Handler will follow a structure in  regards to handling exceptions

For every exception handler, there is an Exception Registration Record structure.
These structures are chained together to create a "linked list" (a linked list contains a sequence of data records)

When an exception is triggered the OS scans down this list to evaluate the other exception functions until it finds a suitable exception handler for the current exception. (If no error is found, windows places a generic handler at the end to ensure it wil be handled in some manner)

In order to do this the Exception Handler must contain both a pointer to the current “Exception Registration Record” (SEH) and a pointer to the “Next Exception Registration Record” (nSEH). 

As the windows stack grows downwards, the order is reversed.


As an exception occurs in a program function the exception handler pushes the elements of it's structure to the stack as this is part of the function prologue to execute the exception, during which time the SEH will be located at esp +8.

When an exception occurs and the Exception Handler (SEH) is called, it’s value is put in EIP. Since we have control over SEH, we now have control over EIP and the execution flow of the application. 


#### SO WHY IS THIS RELEVANT TO US??!

Well, when we send an attack string which happens to overwrite the SEH, windows will zero out the CPU registers, thus breaking our jump to shellcode ;(

#### What we can do:

Overwrite SEH with a pointer to POP POP RETN. 

#### Why would we want to do that?

Well, first remember that after passing the first exception, EIP is overwritten with the value of SEH.
Second, the registers to which the bytes are popped into **are not important**, what is important is that when a POP occurs, ESP is shifted +4. When a RET is called, the ESP address are moved into EIP and executed.

Remember that the SEH is located at esp+8 so if we increment the stack pointer with 8-bytes and return to the new pointer we will then be executing nSEH. 

After which we will have at least 4-bytes room at nSEH to write some opcode that will jump to an area of memory that we control where we can place our shellcode 

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

![SEH_control](/assets/images/seh/SEH_control.jpg)

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

Now as you can see below, EIP is now at the address of POP+POP+RET, step through until RET is called, and the next screenshot will show we have reached nSEH.

![SEH_break_PPR](/assets/images/seh/SEH_break_PPR.jpg)

![SEH_after_PPR](/assets/images/seh/SEH_after_PPR.jpg)

Now you may have guessed something based of the below sequence, that's right! if we continue we end up in an infinite loop - this is because as we hit BBBB, ANOTHER exception is raised, thus calling POP+POP+RET again...

So now we need to replace nSEH with opcode that will JMP over the next few bytes of space. A safe amount would be 10, and pad the beginning of shellcode with NOPS.

# THIS IS A DRAFT
