---
layout: post
title: "Using Rubeus And Certify To Unpac The Hash​"
categories: blog
shortdate: 03-26-2025
tag: 2025
---

Trying something new, little bit of Rubeus, little bit of Certify, little bit of curiosity...Come check it out with me!


## Introduction

This will be a bit of a different blog post today, but recently I've been exploring the more 'offensive' side of things and working to better understand how attackers access networks and then move laterally and how they identify and accomplish their goals. Recently I was looking at using [Rubeus](https://github.com/GhostPack/Rubeus) (a tool written in C# for interacting with and abusing Kerberos in a Windows environment) for some threat detections, specifically looking at its 'Unpac the hash' capability.

Briefly, unpac the hash involves requesting a certificate from an Active Directory Certificate Services environment, and using it to abuse Kerberos Pre-auth to obtain the users NT and LM hash. This allows you to compromise an existing user session(webshell in user context, malware delivery, social engineering etc) and extract a hash that you can use with additional offensive tooling, like Impacket, without having to know the users password directly. This can be performed because when using certificate based authentication, windows allows the user/machine account to fallback to NTLM authentication by encrypting the NT hash in the TGT response. Its this NT hash that we can decrypt using Rubeus, because we requested the certificate in the first place!

This diagram from [thehackerrecipes.com](https://www.thehacker.recipes/ad/movement/kerberos/unpac-the-hash) explains it well


[![3-26-2025_1.png]/assets/images-3-26-25/3-26-25_1.png)](/assets/images/3-26-25/3-26-25_1.png)


There are a lot of great articles and examples out there on Kerberos auth, NTLM relay and more, so I won't attempt to re-create that, and I will link some at the end of this post. These were immensely helpful in my quest to tackle this in our environment!

## Setup

To get started with this, let's setup a simple lab.
It will consist of:

1x Windows Server 2019 - Domain Controller

1x Windows Server 2019 - Active Directory Certificate Services Server(you can set this up on your DC, I chose to do a separate server)

2x Windows 10 Pro - Standard User Endpoints


You can freely and easily download evaluation copies of Windows for this purpose, alternatively there are tools out there to help you auto provision a lab like this. I took these steps manually, using Vmware Fusion on my Intel based Macbook.

Here's a quick rundown of the steps required for setup:
  1. Install windows
  2. Setup Domain controller as a new domain and forest(nothing special, out of the box config)
  3. Turn on Active directory logging to capture our Kerberos events in Event Viewer
  4. Create as many standard users as you like, only need 1 or 2
  5. Setup the endpoints and join them to the domain
  6. Set exclusions in Windows Defender settings to allow Rubeus and Certify to run freely
  7. Setup the Root CA server, standard installation with Web Enrollment enabled.

You should now have the basic setup needed to test this workflow.

## Certify

On one of the user endpoints, let's start with Certify.

[Certify](https://github.com/GhostPack/Certify) (you can also use the python based Certipy) is a tool to query and abuse certificates in a windows environment. We will use it to identify abusable certificate templates and the Root CA server.

First, let me confirm the user I've targeted is low privilege:


[![3-26-2025_2.png]/assets/images-3-26-25/3-26-25_2.png)](/assets/images/3-26-25/3-26-25_2.png)


And on the user endpoint:

[![3-26-2025_3.png]/assets/images-3-26-25/3-26-25_3.png)](/assets/images/3-26-25/3-26-25_3.png)


[![3-26-2025_4.png]/assets/images-3-26-25/3-26-25_4.png)](/assets/images/3-26-25/3-26-25_4.png)


Great. Now lets run Certify!


[![3-26-2025_5.png]/assets/images-3-26-25/3-26-25_5.png)](/assets/images/3-26-25/3-26-25_5.png)


Starting with a simple **'find /enrollable'**, this will give us info about the CA, and the templates that are deployed as enrollable. You can also run this with /vulnerable to find templates that are vulnerable to specific exploits.

With this info, we will look for a standard template, User.


[![3-26-2025_6.png]/assets/images-3-26-25/3-26-25_6.png)](/assets/images/3-26-25/3-26-25_6.png)


What this tells us is the template is **auto enrollable**, which means we can enroll ourselves, and its available to **Domain Users**, which is good, because that’s the only AD group we are in.

Using this information we can now request a certificate as this user, Alice, using Certify.


[![3-26-2025_7.png]/assets/images-3-26-25/3-26-25_7.png)](/assets/images/3-26-25/3-26-25_7.png)


With the **'request'** command we can interact with the CA to request various certificates from templates. Here we are specifying the CA server and the template we want, **'User'**.
Certify then returns us some nice output, letting us know the cert was requested and granted, and then pastes the  private key and certificate data we need to create the certificate file. 

So we can take that output, beginning at **-----BEGIN RSA PRIVATE KEY** and ending at **-----END CERTIFICATE**, and save it in notepad as a .pem file.

Certify will also helpfully tell us how to convert the certificate to a .pfx file needed for further use.


[![3-26-2025_8.png]/assets/images-3-26-25/3-26-25_8.png)](/assets/images/3-26-25/3-26-25_8.png)


If openssl is on your endpoint, you could run this there, however I will transfer this to my host, run it, then copy the .pfx file back to the windows host.

## Rubeus

With the .pfx file in hand, we can now run Rubeus, using the **asktgt** module with **/getcredentials**, to use the certificate to complete the pre auth process and get the NT hash back.


[![3-26-2025_9.png]/assets/images-3-26-25/3-26-25_9.png)](/assets/images/3-26-25/3-26-25_9.png)


Hmmm, that's unfortunate. Luckily with some googling, this may just be a result of my 'out of the box' setup, or a result of creating a separate Root CA server instead of setting it up on my DC.

This is easily solved by created a new certificate on the DC as outlined at the bottom of this issue on [Github] (https://github.com/GhostPack/Rubeus/issues/86).


[![3-26-2025_10.png]/assets/images-3-26-25/3-26-25_10.png)](/assets/images/3-26-25/3-26-25_10.png)


With the certificate created, we have success!

Included in the Rubeus output is the base64 ticket we can use for pass the ticket commands, and at the bottom is our Nt Hash!


[![3-26-2025_11.png]/assets/images-3-26-25/3-26-25_11.png)](/assets/images/3-26-25/3-26-25_11.png)


This hash and the corresponding base64 blob(after being converted) can be used for additional auth in other offensive tools like Impacket, without needing to know the users actual password.

And of course, I started researching this in order to write a detection for it, so let's see what that would look like in the Active Directory logs.

## Detection

First we expect EventCode 4768, **'A Kerberos authentication ticket (TGT) as requested'**, followed by EventCode 4769, **'A Kerberos service ticket was requested'**. These will of course be typical in kerberos authentication flows, but theres a way to focus in on only the events that we really want.
The subject user will be the same in both, our low privilege user Alice, however the key part is the Ticket Options present in the 4769 event.


[![3-26-2025_12.png]/assets/images-3-26-25/3-26-25_12.png)](/assets/images/3-26-25/3-26-25_12.png)

[![3-26-2025_13.png]/assets/images-3-26-25/3-26-25_13.png)](/assets/images/3-26-25/3-26-25_13.png)


Note a couple of things.

Firstly 4768 contains some very useful certificate information. This of course is the certificate that was requested by us, but it could be compared to either certificate request logs, or the certificate management gui on the Root CA server to identify the template used and more info, so its useful here for enrichment.
Secondly, the Ticket Options in this case is actually pretty specific. There's a stellar article [HERE] (https://medium.com/falconforce/falconfriday-detecting-unpacing-and-shadowed-credentials-0xff1e-2246934247ce) from Henri Hambartsumyan that breaks this down. Definitely check that out, it was immensely helpful for me in building this out!

Basically, looking at the source code for multiple offensive tools, we can narrow in on what ticket options are used when requesting the credentials, and we can use that to isolate the suspicious events!

So this 4769 event confirms what we expect based on the article, and we can use this to create a detection that looks for 4769 events with Ticket Options equal to either **"0x40800018" or "0x40810018"**, the first being the hex value WITHOUT the **'CANNONICALIZE'** bit value set.
See the [MSdocs] (https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-4769) for more information. 

You can then combine this with the 4768 event to pull out the certificate information, to add further enrichment to the detection results.

In Splunk you could do something like this
```
index=windowseventlogs sourcetype=wineventlog (EventCode=4769 Ticket_Options IN  ("0x40800018","0x40810018") )  OR (EventCode=4768 Ticket_Options="0x40800010") 
```

Noting that Ticket_Options=0x40800018 is Forwardable, Renewable, Renewable_ok, Enc_tkt_in_skey and Ticket_Options=0x40810018 is Forwardable, Renewable, Canonicalize, Renewable_ok, Enc_tkt_in_skey.

In our environment, this is a very high fidelity detection so it should be very useful in the future!

Thanks for reading, see you next time :)

## Links

Detecting Unpac The Hash [LINK] (https://medium.com/falconforce/falconfriday-detecting-unpacing-and-shadowed-credentials-0xff1e-2246934247ce)

Event Code 4769 [LINK] (https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-4769)

Rubeus Usage [LINK] (https://www.hackingarticles.in/a-detailed-guide-on-rubeus/)

More Rubeus Usage [LINK] (https://labs.lares.com/fear-kerberos-pt2/#UNPAC)

Rubeus Usage in HackTheBox [LINK] (https://0xdf.gitlab.io/2024/10/26/htb-mist.html)

KDC_ERR_PADATA_TYPE_NOSUPP Github Issue [LINK] (https://github.com/GhostPack/Rubeus/issues/86)

Rubeus Usage - thehacker.recipes [LINK] (https://www.thehacker.recipes/ad/movement/kerberos/unpac-the-hash)

Rubeus Github [LINK] (https://github.com/GhostPack/Rubeus)

Certify Github [LINK] (https://github.com/GhostPack/Certify)