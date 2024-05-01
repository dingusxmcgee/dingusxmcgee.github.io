---
layout: post
title: "The Case Of The Missing Method​"
categories: blog
date: 2024-02-01
tag: 2024
---

Today is a quick and fun one, we are going to look at an unassuming .vbs file titled **"Scanned-REF23CR1103BILLED.vbs"**. Surely legitimate business, right?

Right….?

Well, right away when we open it and see that it is 1917 lines long, we should know that something is very…..wrong.

Right away, with a discerning eye, we can see some junk comments, and junk code.
How do we know its junk code?


[![2-1-24_1.png](/assets/images/2-1-24/2-1-24_1.png)](/assets/images/2-1-24/2-1-24_1.png)


Well, a simple search of the function name, **x0dtc**, reveals it is used 29 times, none of which are actual executions.

Scrolling a little further we see something interesting that looks worth keeping.


[![2-1-24_2.png](/assets/images/2-1-24/2-1-24_2.png)](/assets/images/2-1-24/2-1-24_2.png)



Now this is interesting.
Looks like some junk text, and the beginning of some string building. I see some powershell in there.
Lastly, the large text variable, **Hwxc0**, is PROBABLY base64 (spoiler alert, its base64), but with some spice mingled in between some of the characters.

Lets keep scrolling.

Just below the interesting code, we see some more junk code.
We will confirm this by doing the same trick with the first function, and find that this one, **0oUrkN**, is used 80 times(it's included in the previous function), but NONE of them are executions.


[![2-1-24_3.png](/assets/images/2-1-24/2-1-24_3.png)](/assets/images/2-1-24/2-1-24_3.png)


If we continue on, we will find a few more interesting bits, and a LOT more junk. 
It will take a few minutes to delete all this garbage, but lets make this easier to read.


[![2-1-24_4.png](/assets/images/2-1-24/2-1-24_4.png)](/assets/images/2-1-24/2-1-24_4.png)


Huzzah!

I will add that, in this case, it's quite simple to get the output of this without decoding/piecing it together yourself.
Simply replace **vmejl.Run** with **Wscript.Echo**, save the script and run it via cscript or wscript and voila, the output!

[![2-1-24_4-1.png](/assets/images/2-1-24/2-1-24_4-1.png)](/assets/images/2-1-24/2-1-24_4-1.png)


Now right away we can see the rest of the string building happening, as well as some powershell execution and, as we guessed it, that big string is base64, but they are replacing a single character with 'A' throughout the whole string, and then converting that from base64.
Finally, that decoded text is passed to a new variable that replaces some text with the filename of the .vbs script and powershell is used to run that command.

Let's see what this decoded text looks like, as that's the 'next stage' here.


[![2-1-24_5.png](/assets/images/2-1-24/2-1-24_5.png)](/assets/images/2-1-24/2-1-24_5.png)


I split the last line up so it wasn't so long, but what we have is pretty simple.
A base64 string is downloaded from the first url, decoded and saved into a byte array. That's interesting.
Then, that byte array is reflectively loaded and a specific method and class is invoked, with a second url(Likely intended to be either a C2 address or another download stage, however it was unavailable at the time of analysis), the .vbs script filename, some underscores and dashes and a couple other parameters. Note that the url here is backwards, the method call is likely responsible for reversing that url.

So what's at this first url?


[![2-1-24_6.png](/assets/images/2-1-24/2-1-24_6.png)](/assets/images/2-1-24/2-1-24_6.png)


Its another paste website!

What's sitting at this one?


[![2-1-24_7.png](/assets/images/2-1-24/2-1-24_7.png)](/assets/images/2-1-24/2-1-24_7.png)

[![2-1-24_8.png](/assets/images/2-1-24/2-1-24_8.png)](/assets/images/2-1-24/2-1-24_8.png)


A 21kb text file! That sure looks like base64 to me, confirmed by the script decoding it upon downloading it. Let's use cyberchef to quickly and easyly decode it.


[![2-1-24_9.png](/assets/images/2-1-24/2-1-24_9.png)](/assets/images/2-1-24/2-1-24_9.png)


Well would you look at that. This is an executable file! Which makes a lot of sense given its being loaded into memory by the script.

Let's look deeper at this file.


[![2-1-24_10.png](/assets/images/2-1-24/2-1-24_10.png)](/assets/images/2-1-24/2-1-24_10.png)


DetectitEasy tags it as a 32bit .net dll.


[![2-1-24_11.png](/assets/images/2-1-24/2-1-24_11.png)](/assets/images/2-1-24/2-1-24_11.png)


VirusTotal doesn’t have much for us, a few detections but not much to go on.

Usually, my next task would be to open the file in dnSpyEx, find the class and method called, and analyze its functionality.

However in this case, we have a bit of a strange ending to the story.

The method does not exist!
We can confirm this a couple of easy ways.

First, open the file in dnSpyEx and navigate to Class1 and observer, no method!


[![2-1-24_12.png](/assets/images/2-1-24/2-1-24_12.png)](/assets/images/2-1-24/2-1-24_12.png)


Second, we can simply execute this in an isolated analysis VM, and observe it's behavior.


[![2-1-24_13.png](/assets/images/2-1-24/2-1-24_13.png)](/assets/images/2-1-24/2-1-24_13.png)


Well, that's not going to work very well for the malware now is it.

Lastly, let's check one more thing in powershell.
We will load the dll into a variable, and see what types we have available.


[![2-1-24_14.png](/assets/images/2-1-24/2-1-24_14.png)](/assets/images/2-1-24/2-1-24_14.png)


So we can see Class1, as expected, because we saw that in dnSpyEx, however if we try to load that Type, we will get a null object, because it does not exist. To show this, we simply ask powershell if the variable is equal to $null, and it returns true.


[![2-1-24_15.png](/assets/images/2-1-24/2-1-24_15.png)](/assets/images/2-1-24/2-1-24_15.png)


Hence, when we attempt to load the non-existent method, we get the error "You cannot call a method on a null-valued expression".


And that's it for today! A quick and dirty blog post to start the new year. Fairly simple but I thought it was worth showcasing the idea that, sometimes, the simplest answer is the correct answer. When analyzing malware and trying to understand the developers intent, it can be easy to say 'what am I missing' or 'there must be another layer of obfuscation', but sometimes, the malware is just….incomplete!

I'll take that as a win.

Happy hunting, and thanks for reading :)



# IOCs:

Name                  | Sha256 Hash           |
--------------------- | --------------------- |
Maracaibo.dll        | 35e97d4acc4e0fc8ec1e0f9541e79fa55f101184a1478496b151897eefe74d2e |
Scanned-REF23CR1103BILLED.vbs	       | 84a716db882d3c85f780322a53e3f9ad92390bbaa2618e419e7cd41adced229f |


## Network based indicators:


### URLs:
hxxps://textbin[.]net/raw/ezjmofz3s6

hxxps://pasteio[.]com/download/xmDw1tNRq7rK

hxxps://paste[.]ee/d/WLTpd/0