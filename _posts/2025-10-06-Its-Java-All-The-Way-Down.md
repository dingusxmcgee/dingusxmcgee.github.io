---
layout: post
title: "It's Java All The Way Down​"
categories: blog
shortdate: 10-06-25
tag: 25
---


On October 5, 2025, Oracle posted about newly exploited [CVEs](https://www.oracle.com/security-alerts/alert-cve-2025-61882.html).
Let's take a peek in this quick and dirty blog post.






## Exploit
From my observation the attack path looks like the following:
- Inital exploit via python script -> malicious template files dropped

You would see apache logs like this:
```
    GET /OA_HTML/OA.jsp?page=/oracle/apps/xdo/oa/template/webui/TemplateCopyPG

    POST /OA_HTML/OA.jsp?page=/oracle/apps/xdo/oa/template/webui/TemplateCopyPG

    GET /OA_HTML/OA.jsp?page=/oracle/apps/xdo/oa/template/webui/TemplateFileAddPG
```

- Malicious template file activated by 'previewing'

You would see apache logs like this:
```
    POST /OA_HTML/OA.jsp?page=/oracle/apps/xdo/oa/template/webui/TemplatePreviewPG
```

- Two types of templates:
    - Template1: contacts a hardcoded IP Address and executes arbitrary java code.

    - Template2: contains an embedded java class file, which is decoded and executed. Loads a backdoor, that allows an attacker to send a specially crafted POST request to "/support/state/content/destination./navId.1/navvSetId.iHelp/" and execute arbitraty java code.


Let's specifically look at the execution path of the code here, not the exploit.

In both templates, its base64 encoded data that is decoded and then executed.


## Template Type 1
Get's the execution context of the current Oracle Weblogic server. This will allow the javascript to execute inside that current process and not drop additional files to disk.


[![10-06-25_1.png](/assets/images/10-06-25/10-06-25_1.png)](/assets/images/10-06-25/10-06-25_1.png)


Retrieve the http response object and send back arbitrary data.


[![10-06-25_2.png](/assets/images/10-06-25/10-06-25_2.png)](/assets/images/10-06-25/10-06-25_2.png)


Connects to hardcoded C2 ip with a 'TLSv3.1' string, appened with a randomly generated number.


[![10-06-25_3.png](/assets/images/10-06-25/10-06-25_3.png)](/assets/images/10-06-25/10-06-25_3.png)


Reads binary data from the C2 into a buffer, decrypts it with a rolling XOR key and then loads it as a Java Class, then invokes it.


[![10-06-25_4.png](/assets/images/10-06-25/10-06-25_4.png)](/assets/images/10-06-25/10-06-25_4.png)


[![10-06-25_5.png](/assets/images/10-06-25/10-06-25_5.png)](/assets/images/10-06-25/10-06-25_5.png)


Returns an html response to the original http request to indicate status of exploit.


[![10-06-25_6.png](/assets/images/10-06-25/10-06-25_6.png)](/assets/images/10-06-25/10-06-25_6.png)


## Template Type 2

Template Type 2 has the same initial steps as Template Type 1.

Attempts to load an existing class ''com.fasterxml.jackson.pb.FileUtils'
If that fails, it loads it via embedded base64


[![10-06-25_7.png](/assets/images/10-06-25/10-06-25_7.png)](/assets/images/10-06-25/10-06-25_7.png)


## FileUtils.class

This class appears to be designed to mimic a legitimate Java class.

Sets some initial values. The hardcoded base64 is an embedded Log4jConfigQpgsuaFilter.class, which is the actual filter that will be applied to incoming http requests.

[![10-06-25_8.png](/assets/images/10-06-25/10-06-25_8.png)](/assets/images/10-06-25/10-06-25_8.png)


The FileUtils function:
Get's servlet contexts, creates a new Filter instance and then injects it into each context.


[![10-06-25_9.png](/assets/images/10-06-25/10-06-25_9.png)](/assets/images/10-06-25/10-06-25_9.png)


The add filter function specifies 'REQUEST', FORWARD','INCLUDE' and 'ERROR' types for the filter.


[![10-06-25_10.png](/assets/images/10-06-25/10-06-25_10.png)](/assets/images/10-06-25/10-06-25_10.png)


The functions 'getContextsByMbean', 'getContextsByThreads', and 'getContext' are all responsible for getting various context information in the weblogic server process. The goal is to hook into all weblogic application contexts.


[![10-06-25_11.png](/assets/images/10-06-25/10-06-25_11.png)](/assets/images/10-06-25/10-06-25_11.png)

[![10-06-25_12.png](/assets/images/10-06-25/10-06-25_12.png)](/assets/images/10-06-25/10-06-25_12.png)

[![10-06-25_13.png](/assets/images/10-06-25/10-06-25_13.png)](/assets/images/10-06-25/10-06-25_13.png)


The embedded base64 is then decoded and decompressed using Gzip in the 'getFilter' function.


[![10-06-25_14.png](/assets/images/10-06-25/10-06-25_14.png)](/assets/images/10-06-25/10-06-25_14.png)


It attempts to load the existing class, "Log4jConfigQpgsubFilter" and if this fails, it will load it from the embedded base64.

The filter is then registered and invoked, applied to the url "/support/*"


[![10-06-25_15.png](/assets/images/10-06-25/10-06-25_15.png)](/assets/images/10-06-25/10-06-25_15.png)


## Log4jConfigQpgsubFilter.class


This is the filter applied to incoming http requests.

The 'install' function, takes a byte array, decrypts it with a hardcoded AES IV and Key and reads the bytes into memory.



[![10-06-25_16.png](/assets/images/10-06-25/10-06-25_16.png)](/assets/images/10-06-25/10-06-25_16.png)


The 'doFilter' function checks for incoming POST requests and if they match a specific header value:

```
X-ORACLE-DMS-ECID=9391zlei28zserjz20aaun
```
And a specific URI:

```
/support/state/content/destination./navId.1/navvSetId.iHelp
```

It will call the install function to decrypt the content and load it into memory, then execute the body.
It will also return various value in the http response if the 'install' was successful or not.


[![10-06-25_17.png](/assets/images/10-06-25/10-06-25_17.png)](/assets/images/10-06-25/10-06-25_17.png)



Ultimately what this comes down to is arbitrary code execution and a backdoor that allows additional arbitrary code execution.


Thanks for reading :)


# IOCs:

## Network based indicators:
95.217.144.48

185.80.234.254

64.20.35.130

192.241.102.198

185.174.100.242

162.55.17.215

85.17.28.253

104.194.11.200

31.210.170.160