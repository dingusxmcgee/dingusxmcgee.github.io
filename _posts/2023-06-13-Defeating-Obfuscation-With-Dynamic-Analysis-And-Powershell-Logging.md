---
layout: post
title: "Defeating Obfuscation With Dynamic Analysis And Powershell Logging​"
categories: blog
---

It all started with 'Creative Content Production.js'

[![6-13-23_1.png](/assets/images/6-13-23/6-13-23_1.png)](/assets/images/6-13-23/6-13-23_1.png)


This script is the output of another malicious .js script and involved a malicious scheduled task. The scripts origins appear to be from a malicious .docx file, though the exact chain of events is not clear and unfortunately the system is no longer available.

Sounds like a fun script. Let's check it out!

[![6-13-23_2.png](/assets/images/6-13-23/6-13-23_2.png)](/assets/images/6-13-23/6-13-23_2.png)


45.9 MB!

That's a big script!

[![6-13-23_3.png](/assets/images/6-13-23/6-13-23_3.png)](/assets/images/6-13-23/6-13-23_3.png)


Ok, this looks benign. Time to go get an afternoon snack and have a quick nap!

Just kidding. Just looking at this giant blob of text makes me want to scream.

This is a massive, obfuscated .js file, which appears to be using a variety of obfuscation techniques like replacing extraneous characters, concatenating variables, and single line formatting.

It would take way too much brain power for me to try and manually deobfuscate this, and after 30-40 minutes of trying to clean it up, understand the flow and discern the purpose. I gave up.

Time to go get an afternoon snack and have a quick nap!

Ok, ok ok, let's take another look.

Instead of relying on static analysis, lets approach this dynamically and see what we can glean.

We're going to use two classic tools: Process Hacker and Procmon.

Let's start with Process Hacker.

I know from the Crowdstrike Detection that the following commandline was used by the Scheduled Task to run the script:

[![6-13-23_4.png](/assets/images/6-13-23/6-13-23_4.png)](/assets/images/6-13-23/6-13-23_4.png)

And though you can use either wscript or cscript to run .js scripts, lets use cscript.


[![6-13-23_5.png](/assets/images/6-13-23/6-13-23_5.png)](/assets/images/6-13-23/6-13-23_5.png)

And we'll open Process Hacker and wait to see what happens.

[![6-13-23_6.png](/assets/images/6-13-23/6-13-23_6.png)](/assets/images/6-13-23/6-13-23_6.png)

Let's also open ProcMon now, set a filter for cscript.exe and run the capture. Then execute the script.

[![6-13-23_7.png](/assets/images/6-13-23/6-13-23_7.png)](/assets/images/6-13-23/6-13-23_7.png)

Cscript execution under cmd.exe, as expected because we launched the script from the command prompt.

[![6-13-23_8.png](/assets/images/6-13-23/6-13-23_8.png)](/assets/images/6-13-23/6-13-23_8.png)

And after a few seconds, what's this! Powershell?! 

[![6-13-23_9.png](/assets/images/6-13-23/6-13-23_9.png)](/assets/images/6-13-23/6-13-23_9.png)

Let's take a deeper look!

Under the Properties for the powershell process we can immediately see a couple of interesting things:

[![6-13-23_10.png](/assets/images/6-13-23/6-13-23_10.png)](/assets/images/6-13-23/6-13-23_10.png)

The command line is a great example of the result of concatenating different strings together to form the word 'powershell'. This is very common in obfuscated scripts.
Secondly, the parent process is 'non-existent', which means the cscript process that launched it has been terminated.

What I want to draw attention to here is the empty commandline.
Why would a malicious script launch powershell.exe with no arguments?
Well, you can safely assume that's not actually the case. Let's see if Procmon has what we are looking for.

Procmon will have captured quite a bit of activity, and a lot of it will be benign actions from cscript as it spins up and accesses windows resources to run the script. Let's filter by process creation events and look for powershell.

[![6-13-23_11.png](/assets/images/6-13-23/6-13-23_11.png)](/assets/images/6-13-23/6-13-23_11.png)

Unfortunately, same result here. No command line arguments!

This is a devious little trick that attackers can use. You can pass an entire command to powershell and evade simple detection/analysis by piping it to powershell followed by a dash '-' (there are probably other ways to do this as well).

Like this:

[![6-13-23_12.png](/assets/images/6-13-23/6-13-23_12.png)](/assets/images/6-13-23/6-13-23_12.png)

If we monitor for this in procmon, sure enough we get this:

[![6-13-23_13.png](/assets/images/6-13-23/6-13-23_13.png)](/assets/images/6-13-23/6-13-23_13.png)

Not quite the same as the malicious script, but it appears they used a similar method to achieve the same result.

So how do we know what the attacker ran with powershell? The powershell process has been running in the background this whole time! Clearly it's up to something.

This is where Powershell host logging can really come in handy, luckily its quite easy to setup via group policy, either in a domain environment or individually for on the fly analysis like this.

First, kill the powershell process, then open gpedit.msc on the windows analysis machine.

Navigate to Computer Configuration - Administrative Templates - Windows Components - Windows Powershell and enable 'Turn on PowerShell Script Block Logging'

[![6-13-23_14.png](/assets/images/6-13-23/6-13-23_14.png)](/assets/images/6-13-23/6-13-23_14.png)

This will fire Event Viewer events for powershell commands/scripts.

Also enable 'Turn on PowerShell Transcription' and configure an output directory to store the transcripts.

[![6-13-23_15.png](/assets/images/6-13-23/6-13-23_15.png)](/assets/images/6-13-23/6-13-23_15.png)

This will give you a nice text file with what we are looking for.

Now click OK and lets run the script again, wait for powershell to run and check the output.

Success!

[![6-13-23_16.png](/assets/images/6-13-23/6-13-23_16.png)](/assets/images/6-13-23/6-13-23_16.png)

If we open this up and take a look, we can quickly spot the malicious content

[![6-13-23_17.png](/assets/images/6-13-23/6-13-23_17.png)](/assets/images/6-13-23/6-13-23_17.png)

Now we can copy this into an ide and take a closer look.

It's worth noting that you can also check the event viewer for similar information from eventid 4104

[![6-13-23_18.png](/assets/images/6-13-23/6-13-23_18.png)](/assets/images/6-13-23/6-13-23_18.png)

Now right away we can see this is setup as a one line script, meaning it has semi-colons where every newline should.
So first thing I will do is replace every semi-colon with a new line.

This gives us something fairly readable.

[![6-13-23_19.png](/assets/images/6-13-23/6-13-23_19.png)](/assets/images/6-13-23/6-13-23_19.png)

Note that I have removed one of the domain names to due to its explicit nature.

And from here, a simple clean up can be performed to understand what the script is doing.

[![6-13-23_20.jpg](/assets/images/6-13-23/6-13-23_20.jpg)](/assets/images/6-13-23/6-13-23_20.jpg)

The while loop at the bottom runs the ConnectToUrl function and loops through these hardcoded urls and attempts to connect to them. It gathers the following system info: path variables, running processes, open windowtitles, desktop items, free disk space, and concatenates them with a hardcoded key, then plugs them into the http request header along with a useragent.

It then waits to receive a response from the php page it connects to and will run (iex = invoke-expression) the response.

Luckily, Crowdstrike prevented the process from connecting to the malicious domains, and there does not appear to be any further indicators of malicious activity.

The lesson here is that logging diversity(as well as host based logging) can be invaluable! We needed Powershell host logs to quickly and easily find the malicious Powershell code that ultimately ran here.