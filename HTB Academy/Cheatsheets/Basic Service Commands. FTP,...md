
# FTP
| Category                         | Command / Example                                      | Description                                           |
| -------------------------------- | ------------------------------------------------------ | ----------------------------------------------------- |
| **Connect / Login**              | `ftp <IP> <port>`                                      | Start FTP client session to target                    |
|                                  | `USER <username>`                                      | Provide username (e.g., anonymous)                    |
|                                  | `PASS <password>`                                      | Provide password (any for anonymous)                  |
|                                  | `QUIT`                                                 | Exit session                                          |
| **Modes**                        | `TYPE I`                                               | Binary mode (default for files)                       |
|                                  | `TYPE A`                                               | ASCII mode (text)                                     |
| **Passive / Active**             | `passive`                                              | Enable passive mode (firewall friendly)               |
|                                  | `quote PASV`                                           | Ask server for passive port                           |
|                                  | `PORT h1,h2,h3,h4,p1,p2`                               | Active mode (server connects back)                    |
| **Directory / Listing**          | `PWD`                                                  | Show current remote directory                         |
|                                  | `CWD /path/`                                           | Change remote directory                               |
|                                  | `LS -al`                                               | List files/directories                                |
|                                  | `LS -R`                                                | Recursive listing (if server allows)                  |
|                                  | `NLST`                                                 | Names-only listing                                    |
|                                  | `MLSD`                                                 | Machine-readable listing                              |
| **File Transfer**                | `GET file.txt`                                         | Download file                                         |
|                                  | `PUT file.txt`                                         | Upload file                                           |
|                                  | `APPE file.txt`                                        | Append to remote file                                 |
|                                  | `REST <offset>`                                        | Resume from byte offset                               |
|                                  | `mirror /remote /local`                                | Mirror directories (e.g., wget -m or lftp)            |
| **File Management**              | `DELE file.txt`                                        | Delete remote file                                    |
|                                  | `RNFR old.txt`                                         | Rename from (step 1)                                  |
|                                  | `RNTO new.txt`                                         | Rename to (step 2)                                    |
|                                  | `MKD newdir`                                           | Make directory                                        |
|                                  | `RMD olddir`                                           | Remove directory                                      |
| **Metadata / Info**              | `SIZE file.txt`                                        | Show file size                                        |
|                                  | `MDTM file.txt`                                        | Show last modified time                               |
|                                  | `STAT`                                                 | Show server/session status (ftp-syst)                 |
|                                  | `HELP`                                                 | Show help/commands                                    |
|                                  | `DEBUG`                                                | Enable debug mode (shows commands sent)               |
|                                  | `TRACE`                                                | Packet tracing for commands                           |
| **Security / Encryption**        | `AUTH TLS`                                             | Start explicit TLS (FTPS)                             |
|                                  | `PBSZ 0`                                               | Protection buffer (after TLS)                         |
|                                  | `PROT P`                                               | Protect data channel (encrypt)                        |
|                                  | `openssl s_client -connect <IP>:21 -starttls ftp`      | Connect to FTPS server, view certificate              |
| **Automation / Non-interactive** | `wget -m --no-passive ftp://anonymous:anonymous@<IP>/` | Mirror/download full FTP directory                    |
|                                  | `curl -O ftp://anonymous:anonymous@<IP>/file.txt`      | Download single file                                  |
|                                  | `curl -T localfile ftp://anonymous:anonymous@<IP>/`    | Upload file                                           |
|                                  | `lftp -u anonymous, -e "mirror / /local; bye" <IP>`    | Mirror FTP directories                                |
| **Nmap / NSE Enumeration**       | `sudo nmap -sV -p21 -sC -A <IP>`                       | Version scan, default script scan, aggressive scan    |
|                                  | `sudo nmap -p21 --script=ftp-anon,ftp-syst <IP>`       | Check for anonymous login, server STAT info           |
|                                  | `sudo nmap -sV -p21 -sC -A <IP> --script-trace`        | Trace NSE scripts, see commands/responses             |
| **Connection Testing**           | `nc -nv <IP> 21`                                       | Simple banner grab via TCP                            |
|                                  | `telnet <IP> 21`                                       | Connect to FTP for banner/interact                    |
| **Troubleshoot / Firewall**      | `passive / quote PASV`                                 | Switch to passive mode if data transfer/listing hangs |
|                                  | `wget --no-passive-ftp ftp://...`                      | Force passive FTP for non-interactive download        |
Anonymous login for FTP
- USER: anonymous
- PASS: anonymous@example.com


# SMB

| **Protocol / Tool**        | **Command / Syntax**                                                                                                        | **Purpose / Notes**                                   |
| -------------------------- | --------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------- |
| **SMBclient**              | `smbclient -L //<IP> -U <user>`                                                                                             | List all shares on target (use `-N` for anonymous)    |
|                            | `smbclient //<IP>/<share> -U <user>`                                                                                        | Connect to a specific share                           |
|                            | `smbclient //<IP>/<share> -N`                                                                                               | Connect anonymously                                   |
|                            | `ls`                                                                                                                        | List files/directories in the share                   |
|                            | `cd <dir>`                                                                                                                  | Change directory inside the share                     |
|                            | `get <file>`                                                                                                                | Download file from share                              |
|                            | `mget <files>`                                                                                                              | Download multiple files                               |
|                            | `put <file>`                                                                                                                | Upload file to share (if writable)                    |
|                            | `mput <files>`                                                                                                              | Upload multiple files                                 |
|                            | `!<cmd>`                                                                                                                    | Run local system commands without leaving SMBclient   |
|                            | `help`                                                                                                                      | List all available SMBclient commands                 |
| **Nmap SMB scripts**       | `nmap -p 139,445 -sV --script=smb* <IP>`                                                                                    | Enumerate shares, users, info with NSE scripts        |
| **RPC / rpcclient**        | `rpcclient -U "" <IP>`                                                                                                      | Connect via RPC (null session/anonymous)              |
|                            | `srvinfo`                                                                                                                   | Get server info (OS, version, server type)            |
|                            | `enumdomains`                                                                                                               | List domains/workgroups on server                     |
|                            | `querydominfo`                                                                                                              | Get domain info (users, groups, roles)                |
|                            | `netshareenumall`                                                                                                           | List all shares via RPC                               |
|                            | `netsharegetinfo <share>`                                                                                                   | Get detailed info about a share                       |
|                            | `enumdomusers`                                                                                                              | Enumerate all domain users                            |
|                            | `queryuser <RID>`                                                                                                           | Get info about a specific user                        |
|                            | `querygroup <RID>`                                                                                                          | Get info about a group                                |
|                            | **Bash RID brute-force**: `for i in $(seq 500 1100); do rpcclient -N -U "" <IP> -c "queryuser 0x$(printf '%x\n' $i)"; done` | Enumerate users if RIDs unknown                       |
| **Impacket / samrdump.py** | `samrdump.py <IP>`                                                                                                          | Enumerate users via RPC remotely                      |
| **SMBMap**                 | `smbmap -H <IP>`                                                                                                            | Enumerate shares and permissions                      |
| **CrackMapExec (CME)**     | `crackmapexec smb <IP> --shares -u '' -p ''`                                                                                | Enumerate shares with anonymous/null login            |
| **Enum4Linux-ng**          | `./enum4linux-ng.py <IP> -A`                                                                                                | Automated SMB/RPC enumeration, users, shares, OS info |
RPC Commands
- Connect anonymously to RPC, then send the commands
- ie.
	- rpcclient -U "" 10.129.14.128
	- srvinfo

# DNS

| Protocol / Tool                  | Command / Syntax                 | Purpose / Notes                                                                                             |
| -------------------------------- | -------------------------------- | ----------------------------------------------------------------------------------------------------------- |
| `dig`                            | `dig example.com`                | Query default DNS resolver for `example.com`.                                                               |
| `dig`                            | `dig example.com @<DNS_IP>`      | Query a **specific DNS server** directly at `<DNS_IP>`. Useful for internal network enumeration.            |
| `dig`                            | `dig NS example.com @<DNS_IP>`   | List authoritative name servers (NS records) for a domain.                                                  |
| `dig`                            | `dig AXFR example.com @<DNS_IP>` | Attempt **zone transfer** from the DNS server to get all records for the domain. Only works if allowed.     |
| `dig`                            | `dig ANY example.com @<DNS_IP>`  | Query all available record types that the server will disclose.                                             |
| `dig`                            | `dig +trace example.com`         | Show full recursive resolution path from root servers down to authoritative server.                         |
| `nslookup`                       | `nslookup example.com`           | Query DNS (default resolver) for a domain.                                                                  |
| `nslookup`                       | `nslookup example.com <DNS_IP>`  | Query a specific DNS server directly.                                                                       |
| `host`                           | `host example.com`               | Simple command to resolve a hostname to an IP using default DNS.                                            |
| `host`                           | `host example.com <DNS_IP>`      | Query a specific DNS server directly.                                                                       |
| `resolvectl` / `systemd-resolve` | `resolvectl query example.com`   | Query DNS using systemd-resolved.                                                                           |
| `resolvectl` / `systemd-resolve` | `resolvectl status`              | Show the DNS servers being used by the system.                                                              |
| System file                      | `/etc/resolv.conf`               | Shows configured DNS resolvers. `nameserver 127.0.0.53` indicates local stub resolver via systemd-resolved. |
| `tcpdump`                        | `sudo tcpdump -i any port 53`    | Capture DNS traffic on port 53 to observe queries and responses.                                            |



# Imap/pop3

| Protocol / Tool                 | Command / Syntax                                                      | Purpose / Notes                                                                            |
| ------------------------------- | --------------------------------------------------------------------- | ------------------------------------------------------------------------------------------ |
| **Nmap**                        | `nmap -p 110,143,993,995 <target>`                                    | Scan for POP3 (110/995 SSL) and IMAP (143/993 SSL) services.                               |
| **Nmap Scripts**                | `nmap --script=imap-capabilities -p 143 <target>`                     | Enumerate IMAP capabilities.                                                               |
|                                 | `nmap --script=pop3-capabilities -p 110 <target>`                     | Enumerate POP3 capabilities.                                                               |
|                                 | `nmap --script=imap-ntlm-info -p 143 <target>`                        | Extract NTLM info if supported.                                                            |
| **OpenSSL**                     | `openssl s_client -connect <target>:993` (can replace 993 with imaps) | Connect to IMAPS (SSL).                                                                    |
|                                 | `openssl s_client -connect <target>:995` (can replace 995 with pop3s) | Connect to POP3S (SSL).                                                                    |
| **Telnet / Netcat (Cleartext)** | `telnet <target> 143`                                                 | Connect to IMAP (cleartext).                                                               |
|                                 | `telnet <target> 110`                                                 | Connect to POP3 (cleartext).                                                               |
| **cURL**                        | `curl -k 'imaps://10.129.14.128' --user user:p4ssw0rd -v`             | Interact with IMAP/POP3                                                                    |
| **Basic IMAP Commands**         | `a login <username> <password>`                                       | Authenticate to IMAP. (`a` is a tag; any string works).                                    |
|                                 | `a list "" "*"`                                                       | List mailboxes/folders.                                                                    |
|                                 | `a create "inbox"`                                                    | Creates a mailbox with specified name                                                      |
|                                 | `a delete "inbox"`                                                    | Deletes mailbox                                                                            |
|                                 | `a rename "ToRead" "Important"``                                      | Rename                                                                                     |
|                                 | `a lsub "" * `                                                        | Return subset of names from set of names that User has declared as being active/subscribed |
|                                 | `a unselect INBOX`                                                    | Unselect inbox                                                                             |
|                                 | `a select INBOX`                                                      | Select the inbox.                                                                          |
|                                 | `a fetch 1 body[]`                                                    | Fetch the first email (full body).                                                         |
|                                 | `a fetch <ID> all`                                                    | Retrieve data associated with message in mailbox                                           |
|                                 | `a logout`                                                            | Exit IMAP session.                                                                         |
| **Basic POP3 Commands**         | `USER <username>`                                                     | Provide username.                                                                          |
|                                 | `PASS <password>`                                                     | Provide password.                                                                          |
|                                 | `STAT`                                                                | Get mailbox status (# messages, size).                                                     |
|                                 | `LIST`                                                                | List messages with sizes.                                                                  |
|                                 | `RETR 1`                                                              | Retrieve message 1.                                                                        |
|                                 | `QUIT`                                                                | Exit POP3 session.                                                                         |
|                                 | `CAPA`                                                                | Display server capabilities                                                                |
|                                 | `DELE 1`                                                              | Delete message 1.                                                                          |
| **Hydra (Brute Force)**         | `hydra -l <user> -P <wordlist> imap://<target>`                       | IMAP brute force.                                                                          |
|                                 | `hydra -l <user> -P <wordlist> pop3://<target>`                       | POP3 brute force.                                                                          |
|                                 |                                                                       |                                                                                            |

# SNMP

Bruteforce to get the community string with onesixtyone, then can enumerate with snmpwalk/braa with the given community string.

| Protocol / Tool                  | Command / Syntax                                                                    | Purpose / Notes                                                           |
| -------------------------------- | ----------------------------------------------------------------------------------- | ------------------------------------------------------------------------- |
| **snmpwalk**                     | `snmpwalk -v2c -c public <IP>`                                                      | Enumerate all OIDs with community string (`public` here).                 |
| **snmpwalk (specific OID)**      | `snmpwalk -v2c -c public <IP> <OID>`                                                | Walk a specific subtree (e.g., `.1.3.6.1.2.1.1` for system info).         |
| **snmpget**                      | `snmpget -v2c -c public <IP> <OID>`                                                 | Query a single OID instead of walking the tree.                           |
| **snmp-check**                   | `snmp-check <IP>`                                                                   | Automated enumeration of common OIDs (system, processes, services, etc.). |
| **onesixtyone**                  | `onesixtyone -c <wordlist> <IP>`                                                    | Brute-force SNMP community strings.                                       |
| **onesixtyone (quick scan)**     | `onesixtyone -c /opt/seclists/Discovery/SNMP/snmp.txt <IP>`                         | Use SecLists wordlist to discover valid SNMP community strings.           |
| **braa**                         | `braa <community>@<IP>:.1.3.6.*`                                                    | Bruteforce enumerate OIDs from SNMP host. Very fast but noisy.            |
| **snmpbulkwalk**                 | `snmpbulkwalk -v2c -c public <IP>`                                                  | More efficient than snmpwalk for retrieving large amounts of data.        |
| **snmpgetnext**                  | `snmpgetnext -v2c -c public <IP> <OID>`                                             | Get the next OID in the MIB tree after the one specified.                 |
| **snmptranslate**                | `snmptranslate -Tp`                                                                 | Show MIB tree to understand available OIDs.                               |
| **snmptranslate (specific OID)** | `snmptranslate -On <OID>`                                                           | Convert between OID names and numbers.                                    |
| **snmpwalk (with auth)**         | `snmpwalk -v3 -u <user> -A <authpass> -a MD5 -X <privpass> -x DES -l authPriv <IP>` | SNMPv3 query with authentication & encryption.                            |
| **snmp-check (service enum)**    | `snmp-check -p 161 <IP>`                                                            | Quick system & service fingerprinting over SNMP.                          |
# MySQL


| Command                                              | Purpose / Notes                                                      |
| ---------------------------------------------------- | -------------------------------------------------------------------- |
| `mysql -u <user> -p<password> -h <IP address>`       | Connect to the MySQL server. No space between `-p` and the password. |
| `show databases;`                                    | List all databases on the server.                                    |
| `use <database>;`                                    | Select a database to work with.                                      |
| `show tables;`                                       | Show all tables in the selected database.                            |
| `show columns from <table>;`                         | Show all columns in the specified table.                             |
| `describe <table>;`                                  | Alternative to `show columns from <table>;`.                         |
| `select * from <table>;`                             | Show all rows and columns in the table.                              |
| `select * from <table> where <column> = "<string>";` | Filter table rows by a specific value in a column.                   |
| `select <column1>, <column2> from <table>;`          | Retrieve specific columns only.                                      |
| `select * from <table> limit <n>;`                   | Display the first `n` rows (useful for large tables).                |
| `select * from order by [asc                         | desc];`                                                              |
| `select count(*) from <table>;`                      | Count number of rows in a table.                                     |
|                                                      |                                                                      |
# MSSQL

| Protocol / Tool                             | Command / Syntax                                                                              | Purpose / Notes                                                                   |
| ------------------------------------------- | --------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------- |
| **MSSQL ping (Nmap)**                       | `nmap -p 1433 --script ms-sql-info <target>`                                                  | Checks MSSQL service, version, and basic info.                                    |
| **Metasploit scan**                         | msql_ping in msfconsole                                                                       |                                                                                   |
| **Check empty password / weak auth (Nmap)** | `nmap -p 1433 --script ms-sql-empty-password --script-args mssql.instance-port=1433 <target>` | Tests if MSSQL accounts have empty passwords.                                     |
| **Finding Mssqlclient**                     | `locate mssqlclient`                                                                          |                                                                                   |
| **Enumerate tables / DB (Impacket)**        | `python3 mssqlclient.py <username>@<target>`                                                  | Connect interactively; run T-SQL commands like `SELECT name FROM sys.databases;`. |
| **SQL authentication with password**        | `python3 mssqlclient.py <username>:<password>@<target>`                                       | Use for SQL login (not Windows Auth).                                             |
| **Windows authentication**                  | `python3 mssqlclient.py <username>@<target> -windows-auth`                                    | Use domain/Windows credentials; password entered interactively.                   |
| **Enumerate databases**                     | `SELECT name FROM sys.databases;`                                                             | Lists all databases on the server.                                                |
| **List non-default databases**              | `SELECT name FROM sys.databases WHERE name NOT IN ('master','model','msdb','tempdb');`        | Useful for pentesting; ignores system DBs.                                        |
| **Switch database**                         | `USE <database_name>;`                                                                        | Switch to a target database.                                                      |
| **List tables**                             | `SELECT TABLE_NAME FROM INFORMATION_SCHEMA.TABLES WHERE TABLE_TYPE='BASE TABLE';`             | Lists all tables in current database.                                             |
| **Query table contents**                    | `SELECT * FROM <table_name>;`                                                                 | Retrieves all rows from a table.                                                  |
| **Top N rows**                              | `SELECT TOP 10 * FROM <table_name>;`                                                          | MSSQL equivalent of MySQL’s `LIMIT 10`.                                           |
| **Check users**                             | `SELECT name FROM sys.sql_logins;`                                                            | Enumerates SQL login accounts.                                                    |
| **Dump hashes (Impacket)**                  | `python3 mssqlclient.py -hashes <LMHASH:NTHASH> <username>@<target>`                          | Authenticate using NTLM hashes instead of password.                               |
| **Execute command**                         | `EXEC xp_cmdshell '<command>';`                                                               | Runs OS commands on the MSSQL server (requires privileges).                       |
| **Backup / restore**                        | `BACKUP DATABASE <db> TO DISK='C:\backup.bak';`                                               | Can be used for DB exfiltration if permissions allow.                             |
| **Enumerate roles/permissions**             | `SELECT * FROM sys.database_principals;`                                                      | Check privileges of users in a DB.                                                |

# Oracle TNS

| Protocol / Tool                                     | Command / Syntax                                                                                                                                                                                                                                                                | Purpose / Notes                                                                                                                                                                                                       |
| --------------------------------------------------- | :------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Port scan + basic TNS info (Nmap)**               | `nmap -sV -p 1521 --script=oracle-tns-version <target>`                                                                                                                                                                                                                         | Probe listener and attempt to fingerprint the TNS listener/version. Use `-sV` to get service version.                                                                                                                 |
| **Brute-force SIDs (Nmap NSE)**                     | `nmap -p 1521 --script oracle-sid-brute --script-args oraclesids=/path/to/oracle-sids <target>`                                                                                                                                                                                 | Guess Oracle instance/SID names against the TNS listener using a SID wordlist. (NSE script: `oracle-sid-brute`). ([Nmap][1])                                                                                          |
| **Brute-force Oracle credentials (Nmap NSE)**       | `nmap -p 1521 --script oracle-brute --script-args userdb=/path/users.txt,passdb=/path/passes.txt <target>`                                                                                                                                                                      | Remote username/password auditing against Oracle (uses supplied lists or defaults). Use `--script-args` to point to custom lists. ([Nmap][2])                                                                         |
| **Detect xp\_cmdshell / config via NSE**            | `nmap -p 1521 --script oracle-sid-brute,oracle-info <target>`                                                                                                                                                                                                                   | Combine scripts for discovery; replace `oracle-info` with other NSE scripts if present. (Use carefully to avoid DoS.)                                                                                                 |
| **TNS name reachability**                           | `tnsping <TNS_ALIAS>` or `tnsping <host>:1521`                                                                                                                                                                                                                                  | Test that a TNS alias/listener is reachable and resolves; quick way to check listener response. (Oracle client utility). ([Oracle Docs][3])                                                                           |
| **Listener control (server-side)**                  | `lsnrctl status` / `lsnrctl services` / \`lsnrctl start                                                                                                                                                                                                                         | stop                                                                                                                                                                                                                  |
| **ODAT — install / help**                           | `sudo apt install odat` then `odat -h` or `./odat.py -h`                                                                                                                                                                                                                        | ODAT (Oracle Database Attacking Tool). Shows subcommands (sidguesser, tnscmd, tnspoison, passwordguesser, privesc, etc.). Use ODAT for SID/service discovery, brute force and exploitation modules. ([Kali Linux][5]) |
| **ODAT all**                                        | `./odat.py all -s 10.129.204.235`                                                                                                                                                                                                                                               |                                                                                                                                                                                                                       |
| **ODAT — guess SIDs / Service Names**               | `./odat.py sidguesser -t <target> -p 1521` or `./odat.py snguesser -t <target> -p 1521`                                                                                                                                                                                         | Run ODAT’s SID/Service-Name guessing modules to find valid SIDs/services before attempting auth. (Has options for wordlists, sleep/delay to avoid ORA-12519). ([GitHub][6])                                           |
| **ODAT — brute credentials**                        | `./odat.py passwordguesser -t <target> -p 1521 -s <SID> -U users.txt -P passes.txt`                                                                                                                                                                                             | Try combos against a discovered SID/service. ODAT supports many modules for post-auth attacks (privesc, file read, scheduler, java, etc.). ([GitHub][6])                                                              |
| **ODAT — TNS commands (tnscmd)**                    | `./odat.py tnscmd -t <target> -p 1521 --tns-string "(DESCRIPTION=...)"`                                                                                                                                                                                                         | Send low-level TNS commands (get listener info, VSNNUM fingerprinting). Useful for recon. ([GitHub][6])                                                                                                               |
| **EZConnect (sqlplus) — quick connect**             | `sqlplus username/password@host:1521/SERVICE_NAME` or `sqlplus username/password@//host:1521/service_name`                                                                                                                                                                      | EZConnect string — connect without `tnsnames.ora`. Works for SERVICE\_NAME; SID can also be used in descriptors. ([Stack Overflow][7])                                                                                |
| **Full connect descriptor (sqlplus)**               | `sqlplus user/password@"(DESCRIPTION=(ADDRESS=(PROTOCOL=TCP)(HOST=host)(PORT=1521))(CONNECT_DATA=(SID=ORCL)))"`                                                                                                                                                                 | When EZConnect fails, use full TNS descriptor — necessary for complex connections. ([Database Administrators Stack Exchange][8])                                                                                      |
| **sqlplus (no login) / then connect**               | `sqlplus /nolog` → `SQL> CONNECT username/password@host:1521/SERVICE`                                                                                                                                                                                                           | Start client without connecting (useful when you need to change settings before auth). ([Oracle Docs][9])                                                                                                             |
| **Check current user**                              | `SELECT USER FROM DUAL;`                                                                                                                                                                                                                                                        | SQL query to see the currently connected DB user. Use inside `sqlplus`.                                                                                                                                               |
| **Show DB version / banner**                        | `SELECT * FROM v$version;`                                                                                                                                                                                                                                                      | Quickly fingerprint Oracle version and installed components.                                                                                                                                                          |
| **List pluggable/container / databases**            | `SELECT name,open_mode FROM v$pdbs;` (multitenant) or `SELECT * FROM global_name;`                                                                                                                                                                                              | Useful on 12c+ multi-tenant DBs to discover PDB names and states.                                                                                                                                                     |
| **List users & roles**                              | `SELECT username,account_status FROM dba_users;` (requires privilege) or `SELECT * FROM all_users;`                                                                                                                                                                             | Enumerate DB users (privileged view may require DBA).                                                                                                                                                                 |
| **Describe a table / object**                       | `DESC schema.table;` or `DESCRIBE table;`                                                                                                                                                                                                                                       | Show table columns and types (SQL\*Plus built-in `DESC`).                                                                                                                                                             |
| **Select top rows**                                 | `SELECT * FROM schema.table WHERE ROWNUM <= 10;`                                                                                                                                                                                                                                | MSSQL equivalent uses `TOP` — Oracle uses `ROWNUM` for quick sampling.                                                                                                                                                |
| **Spooling output to a file**                       | `SQL> SPOOL /tmp/output.txt` `SQL> ...queries...` `SQL> SPOOL OFF`                                                                                                                                                                                                              | Capture session output for later analysis. Very handy in labs.                                                                                                                                                        |
| **Formatting in SQL\*Plus**                         | `SET LINESIZE 200` `SET PAGESIZE 100` `COLUMN col_name FORMAT A50`                                                                                                                                                                                                              | Improve readability of wide results inside `sqlplus`.                                                                                                                                                                 |
| **Run a local SQL script**                          | `SQL> @/path/to/script.sql`                                                                                                                                                                                                                                                     | Execute a script file from SQL\*Plus.                                                                                                                                                                                 |
| **Check for OS command exec vectors**               | `SELECT * FROM v$option WHERE parameter='Java';` then test Java/DBMS\_SCHEDULER/UTL\_FILE modules cautiously                                                                                                                                                                    | Look for features that enable code execution (DBMS\_SCHEDULER, Java VM, external tables). Use only in authorised tests.                                                                                               |
| **Attempt SYSDBA connection (when you have creds)** | `sqlplus sys/password@host:1521/SERVICE AS SYSDBA`                                                                                                                                                                                                                              | Connect as SYSDBA (very high privilege). Use only with authorised credentials.                                                                                                                                        |
| **TNS poisoning (concept / ODAT tnspoison)**        | `./odat.py tnspoison -t <target> -s <SID> ...`                                                                                                                                                                                                                                  | ODAT includes tnspoison module to simulate client-side alias manipulation / poisoning—use only in lab environments. ([GitHub][6])                                                                                     |
| **Safe discovery workflow (suggested)**             | 1) `nmap -p1521 --script=oracle-tns-version <target>` → 2) `nmap --script oracle-sid-brute` → 3) `tnsping` / `lsnrctl services` (if server access) → 4) use ODAT `sidguesser`/`passwordguesser` → 5) `sqlplus` connect → enumerate (`v$version`, `all_users`, `dba_role_privs`) | Follow this ordered approach to minimise noisy attempts and ORA-12519 errors; add `--sleep`/rate control in ODAT to avoid overloading listener. ([GitHub][6])                                                         |
|                                                     |                                                                                                                                                                                                                                                                                 |                                                                                                                                                                                                                       |
|                                                     |                                                                                                                                                                                                                                                                                 |                                                                                                                                                                                                                       |



**Linux Remote management**

|Protocol / Tool|Command / Syntax|Purpose / Notes|
|---|---|---|
|**SSH**|`ssh user@host`|Basic SSH login with default key/password.|
||`ssh -p 2222 user@host`|Connect to non-standard port (2222).|
||`ssh -i key.pem user@host`|Use specific private key for authentication.|
||`ssh -v user@host`|Verbose output for debugging (use `-vv` / `-vvv` for more detail).|
||`ssh -o PreferredAuthentications=password user@host`|Force password-based authentication.|
||`ssh -L 8080:localhost:80 user@host`|Local port forward → access remote service on localhost.|
||`ssh -R 9000:localhost:22 user@host`|Remote port forward → expose local service to remote.|
||`ssh -D 1080 user@host`|Dynamic port forward (SOCKS proxy).|
||`scp file.txt user@host:/tmp/`|Copy file to remote host via SSH.|
||`scp user@host:/etc/passwd ./`|Copy file from remote host to local.|
||`sftp user@host`|Interactive file transfer over SSH.|
|**Rsync (local/remote sync)**|`rsync -avz file.txt /backup/`|Sync file locally (`-a` archive, `-v` verbose, `-z` compress).|
||`rsync -avz dir1/ dir2/`|Sync directories locally.|
|**Rsync over SSH**|`rsync -avz file.txt user@host:/tmp/`|Sync file to remote host over SSH (default).|
||`rsync -avz -e "ssh -p 2222" dir/ user@host:/data/`|Use SSH on custom port.|
||`rsync --progress file.iso user@host:/mnt/`|Show transfer progress.|
|**R-services (legacy, insecure)**|`rlogin host -l user`|Remote login (like telnet, unencrypted).|
||`rsh host command`|Execute a single command on remote host.|
||`rcp file.txt host:/tmp/`|Copy file to remote host.|
||`rexec -l user -p pass host command`|Execute command remotely with username/password.|
|**Other relevant**|`ssh-copy-id user@host`|Copy local public key to remote `authorized_keys`.|
||`ssh-keygen -t rsa -b 4096`|Generate SSH keypair.|
||`ssh-agent bash && ssh-add key.pem`|Load private key into agent.|
||`netstat -tulnp|grep ssh`|
||`systemctl status sshd`|Check SSH service status (Linux systemd).|