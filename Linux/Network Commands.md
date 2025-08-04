**netcat**
- [[Socket programming]]
- `netcat port ip`

**wget**
`wget https://....`
- Download files via HTTP

**scp - secure copy**
From local to remote:
`scp important.txt ubuntu@192.168.1.30:/home/ubuntu/transferred.txt`
- important.txt is local file
- ubuntu is user name on remote system
- 192... is remote IP

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

**downloading files from the server**
- as seen on top, can just use wget to download the file
- wget http://MAHCINE_IP:8000/myfile