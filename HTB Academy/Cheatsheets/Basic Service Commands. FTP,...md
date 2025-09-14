
# FTP
| Category           | Command / Example                                      | Description                                                                 |
|--------------------|--------------------------------------------------------|-----------------------------------------------------------------------------|
| **Connect / Login** | ftp <IP>                                              | Start FTP client session to target                                          |
|                    | USER <username>                                        | Provide username (e.g., anonymous)                                         |
|                    | PASS <password>                                        | Provide password (any for anonymous)                                        |
|                    | QUIT                                                   | Exit session                                                                |
| **Modes**           | TYPE I                                                 | Binary mode (default for files)                                             |
|                    | TYPE A                                                 | ASCII mode (text)                                                           |
| **Passive / Active** | passive                                               | Enable passive mode (firewall friendly)                                     |
|                    | quote PASV                                             | Ask server for passive port                                                 |
|                    | PORT h1,h2,h3,h4,p1,p2                                 | Active mode (server connects back)                                          |
| **Directory / Listing** | PWD                                               | Show current remote directory                                               |
|                    | CWD /path/                                             | Change remote directory                                                     |
|                    | LS                                                    | List files/directories                                                      |
|                    | LS -R                                                 | Recursive listing (if server allows)                                        |
|                    | NLST                                                   | Names-only listing                                                          |
|                    | MLSD                                                   | Machine-readable listing                                                   |
| **File Transfer**   | GET file.txt                                          | Download file                                                               |
|                    | PUT file.txt                                          | Upload file                                                                 |
|                    | APPE file.txt                                         | Append to remote file                                                       |
|                    | REST <offset>                                         | Resume from byte offset                                                     |
|                    | mirror /remote /local                                  | Mirror directories (e.g., wget -m or lftp)                                  |
| **File Management** | DELE file.txt                                         | Delete remote file                                                          |
|                    | RNFR old.txt                                          | Rename from (step 1)                                                        |
|                    | RNTO new.txt                                          | Rename to (step 2)                                                          |
|                    | MKD newdir                                            | Make directory                                                              |
|                    | RMD olddir                                            | Remove directory                                                            |
| **Metadata / Info** | SIZE file.txt                                         | Show file size                                                              |
|                    | MDTM file.txt                                         | Show last modified time                                                     |
|                    | STAT                                                  | Show server/session status (ftp-syst)                                       |
|                    | HELP                                                  | Show help/commands                                                         |
|                    | DEBUG                                                 | Enable debug mode (shows commands sent)                                     |
|                    | TRACE                                                 | Packet tracing for commands                                                 |
| **Security / Encryption** | AUTH TLS                                         | Start explicit TLS (FTPS)                                                   |
|                    | PBSZ 0                                                | Protection buffer (after TLS)                                               |
|                    | PROT P                                                | Protect data channel (encrypt)                                              |
|                    | openssl s_client -connect <IP>:21 -starttls ftp      | Connect to FTPS server, view certificate                                     |
| **Automation / Non-interactive** | wget -m --no-passive ftp://anonymous:anonymous@<IP>/ | Mirror/download full FTP directory                                            |
|                    | curl -O ftp://anonymous:anonymous@<IP>/file.txt       | Download single file                                                        |
|                    | curl -T localfile ftp://anonymous:anonymous@<IP>/     | Upload file                                                                 |
|                    | lftp -u anonymous, -e "mirror / /local; bye" <IP>    | Mirror FTP directories                                                      |
| **Nmap / NSE Enumeration** | sudo nmap -sV -p21 -sC -A <IP>                 | Version scan, default script scan, aggressive scan                          |
|                    | sudo nmap -p21 --script=ftp-anon,ftp-syst <IP>       | Check for anonymous login, server STAT info                                  |
|                    | sudo nmap -sV -p21 -sC -A <IP> --script-trace        | Trace NSE scripts, see commands/responses                                    |
| **Connection Testing** | nc -nv <IP> 21                                     | Simple banner grab via TCP                                                  |
|                    | telnet <IP> 21                                        | Connect to FTP for banner/interact                                          |
| **Troubleshoot / Firewall** | passive / quote PASV                              | Switch to passive mode if data transfer/listing hangs                        |
|                    | wget --no-passive-ftp ftp://...                        | Force passive FTP for non-interactive download                               |
Anonymous login for FTP
- USER: anonymous
- PASS: anonymous@example.com


# SMB

|**Protocol / Tool**|**Command / Syntax**|**Purpose / Notes**|
|---|---|---|
|**SMBclient**|`smbclient -L //<IP> -U <user>`|List all shares on target (use `-N` for anonymous)|
||`smbclient //<IP>/<share> -U <user>`|Connect to a specific share|
||`smbclient //<IP>/<share> -N`|Connect anonymously|
||`ls`|List files/directories in the share|
||`cd <dir>`|Change directory inside the share|
||`get <file>`|Download file from share|
||`mget <files>`|Download multiple files|
||`put <file>`|Upload file to share (if writable)|
||`mput <files>`|Upload multiple files|
||`!<cmd>`|Run local system commands without leaving SMBclient|
||`help`|List all available SMBclient commands|
|**Nmap SMB scripts**|`nmap -p 139,445 -sV --script=smb* <IP>`|Enumerate shares, users, info with NSE scripts|
|**RPC / rpcclient**|`rpcclient -U "" <IP>`|Connect via RPC (null session/anonymous)|
||`srvinfo`|Get server info (OS, version, server type)|
||`enumdomains`|List domains/workgroups on server|
||`querydominfo`|Get domain info (users, groups, roles)|
||`netshareenumall`|List all shares via RPC|
||`netsharegetinfo <share>`|Get detailed info about a share|
||`enumdomusers`|Enumerate all domain users|
||`queryuser <RID>`|Get info about a specific user|
||`querygroup <RID>`|Get info about a group|
||**Bash RID brute-force**: `for i in $(seq 500 1100); do rpcclient -N -U "" <IP> -c "queryuser 0x$(printf '%x\n' $i)"; done`|Enumerate users if RIDs unknown|
|**Impacket / samrdump.py**|`samrdump.py <IP>`|Enumerate users via RPC remotely|
|**SMBMap**|`smbmap -H <IP>`|Enumerate shares and permissions|
|**CrackMapExec (CME)**|`crackmapexec smb <IP> --shares -u '' -p ''`|Enumerate shares with anonymous/null login|
|**Enum4Linux-ng**|`./enum4linux-ng.py <IP> -A`|Automated SMB/RPC enumeration, users, shares, OS info|
RPC Commands
- Connect anonymously to RPC, then send the commands
- ie.
	- rpcclient -U "" 10.129.14.128
	- srvinfo