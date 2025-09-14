		
| **Nmap Option**      | **Description**                                                        |
| -------------------- | ---------------------------------------------------------------------- |
| `10.10.10.0/24`      | Target network range.                                                  |
| `-sn`                | Disables port scanning.                                                |
| `-Pn`                | Disables ICMP Echo Requests                                            |
| `-n`                 | Disables DNS Resolution.                                               |
| `-PE`                | Performs the ping scan by using ICMP Echo Requests against the target. |
| `--packet-trace`     | Shows all packets sent and received.                                   |
| `--reason`           | Displays the reason for a specific result.                             |
| `--disable-arp-ping` | Disables ARP Ping Requests.                                            |
| `--top-ports=<num>`  | Scans the specified top ports that have been defined as most frequent. |
| `-p-`                | Scan all ports.                                                        |
| `-p22-110`           | Scan all ports between 22 and 110.                                     |
| `-p22,25`            | Scans only the specified ports 22 and 25.                              |
| `-F`                 | Scans top 100 ports.                                                   |
| `-sS`                | Performs an TCP SYN-Scan.                                              |
| `-sA`                | Performs an TCP ACK-Scan.                                              |
| `-sU`                | Performs an UDP Scan.                                                  |
| `-sV`                | Scans the discovered services for their versions.                      |
| `-sC`                | Perform a Script Scan with scripts that are categorized as "default".  |
| `--script <script>`  | Performs a Script Scan by using the specified scripts.                 |
| `-O`                 | Performs an OS Detection Scan to determine the OS of the target.       |
| `-A`                 | Performs OS Detection, Service Detection, and traceroute scans.        |
| `-D RND:5`           | Sets the number of random Decoys that will be used to scan the target. |
| `-e`                 | Specifies the network interface that is used for the scan.             |
| `-S 10.10.10.200`    | Specifies the source IP address for the scan.                          |
| `-g`                 | Specifies the source port for the scan.                                |
| `--dns-server <ns>`  | DNS resolution is performed by using a specified name server.          |

## Output Options

|**Nmap Option**|**Description**|
|---|---|
|`-oA filename`|Stores the results in all available formats starting with the name of "filename".|
|`-oN filename`|Stores the results in normal format with the name "filename".|
|`-oG filename`|Stores the results in "grepable" format with the name of "filename".|
|`-oX filename`|Stores the results in XML format with the name of "filename".|

## Performance Options

|**Nmap Option**|**Description**|
|---|---|
|`--max-retries <num>`|Sets the number of retries for scans of specific ports.|
|`--stats-every=5s`|Displays scan's status every 5 seconds.|
|`-v/-vv`|Displays verbose output during the scan.|
|`--initial-rtt-timeout 50ms`|Sets the specified time value as initial RTT timeout.|
|`--max-rtt-timeout 100ms`|Sets the specified time value as maximum RTT timeout.|
|`--min-rate 300`|Sets the number of packets that will be sent simultaneously.|
|`-T <0-5>`|Specifies the specific timing template.|
## Scripts
Categories
- **safe** – won’t harm the target (e.g. banner grabbing, OS detection).
- **intrusive** – may disrupt or crash services (e.g. brute-force login attempts).
- **vuln** – checks for known vulnerabilities.
- **auth** – attempts authentication against services.
- **brute** – tries username/password brute-forcing.
- **discovery** – gathers more information (users, shares, etc.).
- **malware** – detects signs of malware (backdoors, trojans).
- **dos** – tests for denial-of-service vulnerabilities.
- **external** – interacts with third-party services/APIs (like WHOIS).
- **default** – runs when `-sC` (or `--script=default`) is used.

Info Gathering
- `banner` → quick banner grab across services.
- `http-title`, `http-headers`, `http-methods` → web app recon.
- `http-robots.txt`, `http-sitemap-generator` → hidden directories.
- `http-enum` → fingerprint common web apps (CMS, admin panels).
- `http-server-header` → framework/language detection.
- `http-robots.txt` → finds disallowed paths.
- `smb-os-discovery` → info about SMB shares and host OS.
- `nbstat` → NetBIOS names, workgroup, and MAC.


Scripts with wildcard
- Eg. All ftp scripts,
- `--script ftp-*`

Web
`nmap -p80,443 --script=http-title,http-headers,http-methods,http-enum target`
- http-title → grabs webpage title.
- http-headers → shows server info.
- http-methods → lists allowed HTTP verbs (PUT/DELETE risky).
- http-enum → looks for common paths/apps.

SMB
`nmap -p445 --script=smb-os-discovery,smb-enum-shares,smb-enum-users target`
- smb-os-discovery → OS, workgroup, NetBIOS.
- smb-enum-shares → list shares.
- smb-enum-users → list domain/local users.
- smb-vuln-ms17-010 → EternalBlue (classic exam vuln).
- smb-vuln-ms08-067 → older Windows vuln.

Mail
`nmap -p25 --script=smtp-commands,smtp-enum-users target`
- smtp-commands → lists server-supported extensions.
- smtp-enum-users → tries to enumerate valid users.

SNMP
`nmap -sU -p161 --script=snmp-info target`
- snmp-info → basic system info if community string = public.
- (Follow-up with snmpwalk if needed.)

Databases
- MySQL (3306)
	- `nmap -p3306 --script=mysql-info,mysql-enum target`
- MSSQL (1433)
	- `nmap -p1433 --script=mssql-info target`
- Postgres (5432)
	- `nmap -p5432 --script=pgsql-info target`

Vuln checks
- `ftp-anon` → anonymous FTP logins.
- `smb-vuln-ms17-010` → EternalBlue (classic OSCP vuln).
- `smb-vuln-conficker`, `smb-vuln-ms08-067` → old but possible.
- `http-vuln-*` (there are specific scripts for things like Shellshock, Heartbleed, etc.).
- `ssl-heartbleed` → checks for Heartbleed (CVE-2014-0160).
- `http-dombased-xss`, `http-sql-injection` → very basic, not exhaustive.


# Exam Strategy
`nmap -p- --min-rate 1000 -T4 -v target -oN full_ports.txt
- --min-rate 1000: send at least 1000 packets per second
- -T4: verbose timing
- -oN full_ports.txt : save results

Once get list of open ports, then run service/version scan only on those
- `nmap -sC -sV -p <open_ports> target -oN service_scan.txt`