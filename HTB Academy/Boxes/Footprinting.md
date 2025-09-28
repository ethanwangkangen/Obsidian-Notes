
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
the rest -> microsoft remote management

enumerate NFS
- showmount -e 10.129.198.248
	- /TechSupport
- mount the share
	- mkdir target-NFS
	- sudo mount -t nfs 10.129.198.248:/ ./target-NFS/ -o nolock
		- no permission

enumerate SMB