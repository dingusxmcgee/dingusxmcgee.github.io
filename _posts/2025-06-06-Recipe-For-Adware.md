---
layout: post
title: "Recipe For Adware​"
categories: blog
shortdate: 06-06-25
tag: 25
---


On June 2 2025, @xorist posted a screenshot of some javascript code from a 'recipe app' in the InvokeRE community discord. What followed was a rabbit hole of confusion, mysterious functionality, dashed dreams, more confusion, and ultimately culminated in Yahoo Search.

Let's cook up some Adware :)



## I'm Hungry
As with most things in life, it all boils down to food. Food is required to live, it gives the body the nutrients it craves(add some Brawndo to supersize your results). Keeping food interesting is a hard job, and that's where recipes come in handy. Other people have done all the hard work of finding the right combination of ingredients and techniques, all you have to do it replicate it! So you open your email and what do you find, but an article from the New York Times about a delicious 'Vegan Chocolate Cake' recipe that's sure to knock your socks off!


[![6-06-25_1.png](/assets/images/6-06-25/6-06-25_1.png)](/assets/images/6-06-25/6-06-25_1.png)


Inside the article is a link to the recipe, so of course you happily click through. Upon opening the link, you are greeted with a very convincing pop up ad(in recipe terms, that's like a spice that you sprinkle on top of your internet experience, it keeps things interesting!).


[![6-06-25_2.png](/assets/images/6-06-25/6-06-25_2.png)](/assets/images/6-06-25/6-06-25_2.png)


'Ah ha!' you say, 'there it is! The chocolate cake recipe I have been seeking! All I have to do is click this 'Open' button!' and so you gleefully click again, surely one step closer to chocolate covered bliss.

But what you're met with is not a chocolate cake recipe at all….but…another recipe website?


[![6-06-25_3.png](/assets/images/6-06-25/6-06-25_3.png)](/assets/images/6-06-25/6-06-25_3.png)


"Ok then" you say to your self, "At least it looks well designed." And, assured by the website banners promise to 'Find Your Perfect Recipe' you soldier on. Noticing a shiny green 'Download App' button in the top right, you are delighted that an entire catalog of recipes can now be available to you, right on your desktop! So of course you click.


[![6-06-25_4.png](/assets/images/6-06-25/6-06-25_4.png)](/assets/images/6-06-25/6-06-25_4.png)


Windows only? Stinks to be a linux user right now! So you quickly smash that download button and open your newly download 'RecipeLister.exe'


[![6-06-25_5.png](/assets/images/6-06-25/6-06-25_5.png)](/assets/images/6-06-25/6-06-25_5.png)


What you're met with is a tad underwhelming. 'This just looks like the website…' you sigh to yourself, but anyway, you fire up the search feature and search for a new bread recipe.


[![6-06-25_6.png](/assets/images/6-06-25/6-06-25_6.png)](/assets/images/6-06-25/6-06-25_6.png)


[![6-06-25_7.png](/assets/images/6-06-25/6-06-25_7.png)](/assets/images/6-06-25/6-06-25_7.png)


"That should be delicious later!" you shout with joy, and you forget about the recipe app and move on with your day.

Unfortunately for you, while you were searching for bread recipes, this recipe app was doing some nefarious things in the background….

Let's see what's under the hood.

## Malicious Ingredients
[![6-06-25_8.png](/assets/images/6-06-25/6-06-25_8.png)](/assets/images/6-06-25/6-06-25_8.png)


RecipeLister.exe is just an NSIS installer, which means it will execute an 'installer script', drop some files, and likely execute them.

RecipeLister.exe will create a new directory: %userprofile%\Appdata\Local\Temp\2w1rXpxZnwDUwuTeNvdD6FUkeI0 (it may also be created as %userprofile%\Appdata\Local\Temp\1\2w1rXpxZnwDUwuTeNvdD6FUkeI0). And in this directory it will dump its contents.


[![6-06-25_9.png](/assets/images/6-06-25/6-06-25_9.png)](/assets/images/6-06-25/6-06-25_9.png)


Among the contents is an Electron App, "Recipe Finder - Recipe Lister.exe". Aptly named, it will execute its 'main.js' script, along with others, to show you the recipelister website as if it were in your browser.

If we navigate to the /resources directory and unpack the 'app.asar', we can find the main.js script in question.

Inside main.js is where things get interesting.


[![6-06-25_10.png](/assets/images/6-06-25/6-06-25_10.png)](/assets/images/6-06-25/6-06-25_10.png)


'MinimalDecodeStago.js' huh. That sounds interesting. Crypto and Zlib libraries. Ok. Invisible one and zero and a key? Looks like a decryption key to me!


[![6-06-25_11.png](/assets/images/6-06-25/6-06-25_11.png)](/assets/images/6-06-25/6-06-25_11.png)

[![6-06-25_12.png](/assets/images/6-06-25/6-06-25_12.png)](/assets/images/6-06-25/6-06-25_12.png)

[![6-06-25_13.png](/assets/images/6-06-25/6-06-25_13.png)](/assets/images/6-06-25/6-06-25_13.png)


What we have next are some very obvious decode/decryption functions.
From the code it appears the json response from the server will potentially contain encoded characters. Note that the constants correspond to the ones declared at the start of the script, and given they are, as aptly named, invisible characters, they would not show up in the electron app interface! Sneaky!


Heres what an example chunk of the json response looks like:


[![6-06-25_14.png](/assets/images/6-06-25/6-06-25_14.png)](/assets/images/6-06-25/6-06-25_14.png)


Unfortunately, in our testing, we were unable to get the application to return a 'valid' response that it could execute. So the exact payload in scope is still unknown, however, I did manage to find some endpoints in my envrionment that successfully executed SOMETHING beyond the electron app.
Let's finish with what that looks like.

2 minutes after running the electron app, we see the following commandlines executed in succession:

```
C:\WINDOWS\system32\cmd.exe /d /s /c "tasklist /FI "IMAGENAME eq chrome.exe" /NH"
 
C:\WINDOWS\system32\cmd.exe /d /s /c "taskkill /F /IM chrome.exe"
 
C:\WINDOWS\system32\cmd.exe /d /s /c ""C:\Program Files\Google\Chrome\Application\chrome.exe" --restore-last-session --hide-crash-restore-bubble --noerrdialogs --disable-session-crashed-bubble "hxxps://manahegazeda.org/search?ccd=QUMyZGVzdgVdVHd6B1dXdHAHXVBzegQYVHB2AFRWdz8DU1F2dAdSUHZzTiEkCCJ7NQojAFopLCYwSFIQMxBLDjQMFXEhAwUBWlciFiRTIyAABmslNgACdwMsKBlUIDoDNHc=&q=starttt""
```

Where the domain specified by Chrome may also be "sappointedmanah[.]org".

Now this is interesting! I see no other activity from the recipe app after this, and I don't see chrome doing anything interesting either, except doing DNS lookups for some odd domains.
My initial hunch was an installed malicious chrome extension but after finding nothing to that effect, @.koozy suggested checking the Chrome preferences file, which revealed quite a bit.


[![6-06-25_15.png](/assets/images/6-06-25/6-06-25_15.png)](/assets/images/6-06-25/6-06-25_15.png)

[![6-06-25_16.png](/assets/images/6-06-25/6-06-25_16.png)](/assets/images/6-06-25/6-06-25_16.png)

[![6-06-25_17.png](/assets/images/6-06-25/6-06-25_17.png)](/assets/images/6-06-25/6-06-25_17.png)


It appears the recipe app has updated some important browser preferences including the 'new tab url' the 'default search provider' and cookie settings for these suspicious domains, along with a few other settings.


I extracted the entire UserData folder from the host and a quick look in the rest of the Chrome UserData folder didn’t turn up anything else obvious, so I think what we are left with here is….adware? :D

For fun, I loaded up the UserData folder in a VM and started up chrome. The result was a bit enjoyable if I'm being honest.

I can't figure out how to embed a video here, but click this for a little Adware Demo :D
[![6-06-25_18.mp4](/assets/images/6-06-25/6-06-25_18.mp4)](/assets/images/6-06-25/6-06-25_18.mp4)


You can clearly see the fake search box, followed by 2 redirects that are not very speedy and ultimately we land on….Yahoo search???? It's a strange choice for sure.

It seems that all this effort has gone into deploying adware. Can't say I'd be on board, but there must be money to be made, otherwise they wouldn't be doing it.

Additionally, it's worth noting that the payload could be capable of doing more, or the server could be capable of serving up entirely different payloads, it's hard to say.

I think for now, its best to stick with grandmas old cookbooks. There's no adware in those recipes.


Small update at the end here, while writing this blog post, the recipe website appears to have been flagged as phishing and some more folks have written a couple articles about it. I guess the jig is up


[![6-06-25_19.png](/assets/images/6-06-25/6-06-25_19.png)](/assets/images/6-06-25/6-06-25_19.png)


Thanks for reading :)

For some additional notes and context, check out these posts!

[Squibblydoo X post](https://x.com/SquiblydooBlog/status/1930956119469429149)

[Blumira Article](https://www.blumira.com/blog/suspicious-code-spike-fraudulent-recipe-application)

[Security Magics Article](https://security5magics.blogspot.com/2025/06/suspicious-recipe-app.html)



## Special Thanks

Special thanks to xorist, cyb3rjerry .koozy(these three who by far did the bulk of the analysis work and testing of the payload!) Josh, altrok and Rajnikanth in the Invoke RE discord for all their help and ideas while we analyzed this campaign. And extra thanks to Rajnikanth for the blog title idea :)




## Update 6-8-25
There has been some interesting discussion on this campaign since posting this blog, and I thought it worth a brief update.

Firstly, there has been discussion around labeling this "Adware" and that it is incorrect. As I think about it, I agree. This is malware that, at the time of posting, was deliviring adware. Due to the way the attack is structured, the payload could be anything, in fact the Adware payload could be a decoy, or a less 'extreme' payload to evade complete detection/analysis. It's a bit hard to say at this point.
The fact that this mechanism could be used to deliver a WIDE variety of payloads makes it more akin to a 'backdoor' as mentioned by @struppigel, and I happen to agree.

Secondly, there has been some additional [chatter on X](https://x.com/InvokeReversing/status/1931053939996426673).

@squibblydoo notes that a code signing cert would cost approximately $3,000 USD and so to deliver a simple adware payload at the end of this infection chain seems a bit out of place. They also mentioned that this cert has been revoked now, and you can see the same result when you look at the file on Virus Total.

It was also mentioned that there was a similar IOC, lookupkitchen[.]com.

When I looked into this I found some interesting details.

If you check that domain on VirusTotal, you can see some related files.


[![6-06-25_20.png](/assets/images/6-06-25/6-06-25_20.png)](/assets/images/6-06-25/6-06-25_20.png)


There's actually 2 separate hashes here(listed in IOCs below).


[![6-06-25_21.png](/assets/images/6-06-25/6-06-25_21.png)](/assets/images/6-06-25/6-06-25_21.png)


Looking at 0a6be2102904d3e597aa914234cb82af13f7a4eb3545ead6da9b4afd696e0a25 we can see it has no detections, and when we actually download this and check, it is another NSIS installer, it also drops an electron app into appdata\local\temp, however the main.js in this electron app is completely devoid of malicious content.

In fact, the server url is of some interest here.


[![6-06-25_22.png](/assets/images/6-06-25/6-06-25_22.png)](/assets/images/6-06-25/6-06-25_22.png)


And one more thing, if we look at the signature status, its not signed!


[![6-06-25_23.png](/assets/images/6-06-25/6-06-25_23.png)](/assets/images/6-06-25/6-06-25_23.png)


This, combined with the localhost url and lack of malicious content makes me think this was the very first 'test' of the method. Everything else about the application is the same.
Let's note that this was first seen on VT on 4/2/25.



Looking at 4331c79e34c13857f419448cfdad67c1216f90d27629514ca6ad3281592a4dbf, it actually appears to be identical to RecipeLister.exe.
NSIS instsller, electronapp, and identical main.js file, including identical decryption logic and decryption key. The file is also signed with the same certificate as RecipeLister.exe


[![6-06-25_24.png](/assets/images/6-06-25/6-06-25_24.png)](/assets/images/6-06-25/6-06-25_24.png)


What's odd is that the certificate does not show as revoked here, where it does for RecipeLister.exe. I do not know why.
The first seen date on this hash for VT is 4/19/25.

And if we compare to the first seen date for RecipeLister.exe, its 5/6/25.

What this says to me is that Lookupkitchen was the initial 'testbed' for this infection chain.

We get a little bit of insight into the process, testing the bare electron app, then testing the added malicious content, and then finally, the real campaign. Of course its possible this is also a precursor to additional activity. Perhaps the adware payload is just an additional test and there is more to come via a different domain/hash/etc.

I will be keeping an eye out thats for sure!

Thanks for reading :)

# IOCs:

Name                  | Sha256 Hash           |
--------------------- | --------------------- |
RecipeLister.exe        | 1619bcad3785be31ac2fdee0ab91392d08d9392032246e42673c3cb8964d4cb7 |
Recipe Finder - Recipe Lister.exe	       | 9c58aaca8dde7198240f7684b545575e4833d725d67f37e674e333eeb3ec642c |
LookupKitchen.exe (benign)        | 0a6be2102904d3e597aa914234cb82af13f7a4eb3545ead6da9b4afd696e0a25 |
LookupKitchen.exe (malicious)	       | 4331c79e34c13857f419448cfdad67c1216f90d27629514ca6ad3281592a4dbf |
Recipe Finder - Lookup Kitchen.exe (benign)        | e407f70f62bc89bd8e33d5f99917bedee8e536dca9a7b3790da4d63f4094cc32 |
Recipe Finder - Lookup Kitchen.exe (malicious)	       | b07cbb84d08e579320f9006acac55275618fad5334069dd9251162cc64f013cf |


## Network based indicators:


### Domains:
manahegazeda[.]org

goog.manahegazeda[.]org

home.manahegazeda[.]org

sappointedmanah[.]org

sappoisearchedmanah[.]org

ww1.sappoisearchedmanah[.]org

home.sappointedmanah[.]org

recipelister[.]com

lookupkitchen[.]com