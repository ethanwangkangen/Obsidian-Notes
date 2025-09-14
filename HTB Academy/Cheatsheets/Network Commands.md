

**netcat**
- [[Socket programming]]
- `nc -p <source port> <destination ip> <destination port>`
- `nc -lvnp 4444 > saved_here.txt` 
	- run this on local
	- start a listener on port 4444
	- the output is redirected from stdout into the saved_here.txt file
- `nc local_ip 4444 < file.txt`
	- run this on remote
	- sends the file through the connection

From remote to local:
`scp ubuntu@192.168.1.30:/home/ubuntu/documents.txt notes.txt`

**serving files from host** (from a directory)
`python3 -m http.server`
- starts a HTTP web server in the current directory, using python in built module http.server
- starts a server @localhost:8000
	- can put into browser to see

- by default, only listens on localhost
	- to allow external acces,
	- `python3 -m http.server 8000 --bind 0.0.0.0`
	- binds the server to all network interfaces so devices on local network/internet (if port forwarded) can acccess it

- then on the other machine, can wget

**wget or curl**
`wget https://ip:port/file.txt`
- Download files via HTTP

**connecting to VPN**
- `sudo openvpn user.ovpn`
	- openvpn is the client
	- user.ovpn is the VPN key
- If we then run ifconfig
	- should see a tun adapter if we successfully connected to the vpn

**scp - secure copy (if logged in via ssh)** 
From local to remote:
`scp important.txt ubuntu@192.168.1.30:/home/ubuntu/transferred.txt`
- important.txt is local file
- ubuntu is user name on remote system
- 192... is remote IP
