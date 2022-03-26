---
layout: single
title: "Combining techniques to defeat Windows defender and default Applocker rules"
excerpt: "Using techniques taught in Sektor7's RED TEAM Operator: Malware Development Essentials"
date: 2022-03-24
header:
  thumb: /assets/images/malware_1/1ic4.png
  teaser: /assets/images/malware_1/1ic4.png
  teaser_home_page: true
classes: wide
categories:
  - Malware
tags:
  - Malware
  - AV evasion
  - C#
---


![learnin_malware](/assets/images/malware_1/1ic4.png)


I recently completed Sektor7's ["RED TEAM Operator: Malware Development Essentials"](https://institute.sektor7.net/red-team-operator-malware-development-essentials), and was amazed and enamoured by the C++ techniques and concepts taught therein.  

Having been previously inspired by both [Bobby Cooke's](https://0xboku.com/) aproach to applying course knowledge outside of a lab, in addition to the tradecraft of an ex-colleage and fantastic red-teamer by the name of [Jayden Caelli](https://au.linkedin.com/in/jayden-caelli-849129171). I decided to take a very basic concept and apply the knowledge I received from Sektor7 to create something that would challenge me.  

For the basic technique I chose a standard [xml shellcode runner](https://www.ired.team/offensive-security/code-execution/using-msbuild-to-execute-shellcode-in-c), which I would have to heavily modify in order to get past Defender. This presented a huge obstacle for me personally, as the course taught C++, and the wrapper inside the xml was C#. As a total C# noob, many hours were spent learning how to apply call obfuscation in C#, and frankly I do not have the time or the willpower to cover this little side quest, rest assured that many amusing little mistake were made I learnt a lot :)  

Let's jump into some obligatory screenshots of our environment shall we?  

First we have our up-to-date windows VM, with both real time protection turned on and the default applocker policy:  

![Defender On](/assets/images/malware_1/defender-on.png)  

![Applocker](/assets/images/malware_1/applocker.png)  

As well as our low-priv account, which will be constrained by Applocker's default policy.  

![User](/assets/images/malware_1/user.png)  

So, first let us generate some contrived shelcode to pop calc.exe  

![Calc.exe](/assets/images/malware_1/calc-shellcode.png)

Once we have done this, let's grab the xml shellcode runner template from [here](https://www.ired.team/offensive-security/code-execution/using-msbuild-to-execute-shellcode-in-c) and insert our shellcode.  

![raw shellcode inside vanilla xml](/assets/images/malware_1/non-obfuscated.png)  

We run our malicious xml with MSBuild, and observe that defender catches it. surprised_pokeman_maymay.gif  

![sad trumpets](/assets/images/malware_1/non-obfuscated-result.png)  

Of note are the three win32apis being imported via [PInvoke](https://docs.microsoft.com/en-us/archive/msdn-magazine/2003/july/net-column-calling-win32-dlls-in-csharp-with-p-invoke).  
I knew I had to find a way to abstract away from this functionality, and my searches led me to [delegation](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/delegates/). This was still challenging for me, and after some tyre spinning I was suggested to take a look at [this](https://stackoverflow.com/questions/48969793/how-to-load-dll-dynamically-and-pass-get-value-to-it) stackoverflow post, which gave me enough breadcrumbs to begin crafting some workable code.  

During my initial testing, I chose [MessageBox](https://docs.microsoft.com/en-us/dotnet/api/system.windows.forms.messagebox?view=windowsdesktop-6.0) to ensure that as a proof of concept I could at least call a trivial win32api.  
After being sure that my code was correct I was plagued by null pointer errors. Through troubleshooting I isolated the issue down to either the dllname or the function name, well, we all know that user32.dll is a dll that... exists, I decided to load it up in a debugger and visually confirm for my own sanity that `MessageBox` is indeed still present in user32.dll:  

![always check the debugger...](/assets/images/malware_1/MessageBox-ftw.png)  

Lo and behold, it does not exist.  
After amending my code, I was able to confirm that my delegate code template is indeed good, and calls as it should.  

![MessageBox success](/assets/images/malware_1/messagebox-called-with-working-code.png)  
 

Having confirmed my template, I could now go on to replace using [DllImport](https://docs.microsoft.com/en-us/dotnet/api/system.runtime.interopservices.dllimportattribute?view=net-6.0) with delegates.  

![delegates](/assets/images/malware_1/Converted-calls.png)

This was still caught by AV as I expected, however I had another trick up my sleeve thanks to the Sektor7 course work. I spend an amount of time creating a python script that would generate a malicious xml file containing shellcode supplied in the form of a .bin file as a script arg.  
This script would also AES encrypt the names of the delegates, the shellcode itself, and multiple other strings and variables.  

![script snippet](/assets/images/malware_1/python-script.png)  

![script snippet2](/assets/images/malware_1/python-script2.png)  

![snippet of output](/assets/images/malware_1/Snippet-of-obfuscated-xml.png)  

We can see that by simply applying a few basic techniques, we end up with a vastly different payload than we originally did, with the original on the right and our generated xml on the left:  

![compare](/assets/images/malware_1/xml-compare.png)  

I grabbed a raw https beacon payload from Cobalt Strike and threw it into my script

![cobalt generate](/assets/images/malware_1/python-script-cobalt.png)  

The generated xml ended up too much of a challenge for Defender, and code execution was achieved, with the end result being a nice and shiny Cobalt Strike Beacon.  

![beacon](/assets/images/malware_1/cobalt-strike-beacon.png)  

Unfortunately however, the interactive session with Cobalt Strike gets caught eventually. Meaning that I will have to build upon the techniques applied thus far, most likely with whatever I learn in the next Sektor7 class/OSEP.  
