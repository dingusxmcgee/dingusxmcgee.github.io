---
layout: post
title: "Oops, Supply Chain Compromise! - Part 1​"
categories: blog
shortdate: 05-28-2024
tag: 2024
---

The year was 2022. Fresh into February and feeling good about the prospects for the days ahead . I had woken up around 7am, nothing unusual. Checking my email on my phone revealed news that was far more effective than any alarm clock. Much richer and full bodied than any cup  of coffee. It was threat hunting notice. A legitimate executable. A suspicious, but otherwise clean, domain. Something darker was lurking beneath the surface…


**NOTEPAD MAKING NETWORK CONNECTIONS!**


But I am getting ahead of myself. Let's back up.

## Inital Access - Come On In, The (Back)Door Is Open!

You see, in the late winter of 2022 a live chat software company suffered a supply chain attack. Ne'er-do-wells had made their way into the companies development infrastructure and inserted a trojan downloader into their primary desktop application.

This is their story.
Sort of.

The vendor in question, was Comm100, and their windows desktop app, is a simple electron app. Big deal right? Wrong! 
Attackers used this to insert a subtle trojan horse into the packaged version of the desktop client. Once published, any desktop clients that automatically contacted the update servers to receive a new version, would receive and execute the trojanized version.

So how did they do it?

Easy!

They simply included a malicious javascript file, and then specified it to be executed automatically by the desktop client application.

Electron apps contain a 'package.json' file which contains some identifying information about the application as well as the directive of the javascript file to run on startup.

Borrowing from [Electron Docs](https://zeke.github.io/electron.atom.io/docs/tutorial/quick-start/), let's look at how this might affect us:

```
The format of package.json is exactly the same as that of Node’s modules, and the script specified by the main field is the startup script of your app, which will run the main process. An example of your package.json might look like this:
{
  "name"    : "your-app",
  "version" : "0.1.0",
  "main"    : "main.js"
}
Note: If the main field is not present in package.json, Electron will attempt to load an index.js.
```

In this case, the attackers replaced the 'main.js' specification with a malicious 'index.js' file in the  same directory.


[![5-28-24_1.png](/assets/images/5-28-24/5-28-24_1.png)](/assets/images/5-28-24/5-28-24_1.png)


So, let's take a look at index.js and see what all the hubbub is about!


[![5-28-24_2.png](/assets/images/5-28-24/5-28-24_2.png)](/assets/images/5-28-24/5-28-24_2.png)


Oh dear.

Well this is going to be fun isn't it!

Let's scroll a little further to the bottom and look at some interesting clues.


[![5-28-24_3.png](/assets/images/5-28-24/5-28-24_3.png)](/assets/images/5-28-24/5-28-24_3.png)


Looks like a c2 or second stage url!
We can also see some other bits like 'log', 'get', 'end', 'error'.
It looks like this is probably an array that's used throughout the script.

This script is quite a mess, but luckily with a little googling we can find that this was likely obfuscated using obfuscate.io, and we can take it to [deobfuscate.io](https://obf-io.deobfuscate.io) to clean it up.


[![5-28-24_4.png](/assets/images/5-28-24/5-28-24_4.png)](/assets/images/5-28-24/5-28-24_4.png)


This is much better.


[![5-28-24_5.png](/assets/images/5-28-24/5-28-24_5.png)](/assets/images/5-28-24/5-28-24_5.png)


Let's break this down.

First import the http module.

Second, send a GET to the  specified url with a function as a callback. This function will process the GET response.

Next, let's quickly breakdown the callback function.

It will take the response and if its 'data' it will store that data to a variable.
If its 'end', it will run eval(execute) the response data previously stored.
If it's 'error', it will log the error to the console.
So effectively this will loop, adding data to the response variable, until there is no more data to process, and then it will execute that data.

Lastly, the overall script will log errors to the console.

So what kind of data was waiting for us at this url?


[![5-28-24_6.png](/assets/images/5-28-24/5-28-24_6.png)](/assets/images/5-28-24/5-28-24_6.png)


Beautiful!

The first thing we will do is use cyberchefs JavaScript Beauitfy recipe to clean up the output.


[![5-28-24_7.png](/assets/images/5-28-24/5-28-24_7.png)](/assets/images/5-28-24/5-28-24_7.png)
 

[![5-28-24_7-1.png](/assets/images/5-28-24/5-28-24_7-1.png)](/assets/images/5-28-24/5-28-24_7-1.png)

Much better.

This script contains a massive obfuscated blob, and is responsible for decoding and executing the obfuscated blob.

Luckily, we do not have to bother with trying to manually obfuscate this mess, we can make two small edits to dump the secondary script.

Change line 31 from eval to 'console.log'.

Remove line 43(it will error and we don’t even need it here).


[![5-28-24_7-2.png](/assets/images/5-28-24/5-28-24_7-2.png)](/assets/images/5-28-24/5-28-24_7-2.png)


Then we can copy and paste, then execute the script in Firefox (or any other browser) to dump the content.

**ALWAYS EXERCISE EXTREME CAUTION WHEN EXECUTING MALICIOUS CODE.**

I am performing this in a safe, sandboxed environment.


[![5-28-24_8.png](/assets/images/5-28-24/5-28-24_8.png)](/assets/images/5-28-24/5-28-24_8.png)


Success!

Copying this to a new file, we can observe some interesting components in what appears to be a command and control script, and our luck continues, because this is not obfuscated in anyway!


[![5-28-24_9.png](/assets/images/5-28-24/5-28-24_9.png)](/assets/images/5-28-24/5-28-24_9.png)


[![5-28-24_10.png](/assets/images/5-28-24/5-28-24_10.png)](/assets/images/5-28-24/5-28-24_10.png)


[![5-28-24_11.png](/assets/images/5-28-24/5-28-24_11.png)](/assets/images/5-28-24/5-28-24_11.png)


Some interesting bits include command execution, message sending/receiving and decoding, as well as fingerprinting and system identification.

The last bit of the script is what executes the whole thing.

This C2 script appears to be custom to this attacker, as I was unable to find any evidence of it being posted online in any form other than posts detailing this specific intrusion, or others from the same group.

In this investigation, we unfortunately don't have any insight into exactly which functions the script executed, but we do know that the script was used to download and execute the next stage, which we will discuss on the next post!