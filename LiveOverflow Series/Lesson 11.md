
qemu-system-x86_64 -boot d -cdrom exploit-exercises-protostar-2.iso -m 1024

- Boots up the iso in a virtual machine
- /bin/bash
- Login:
	- user
	- user
- cd /opt/protostar/bin

file stack0
- setuid ELF 32-bit LSB executable. Dynamically linked, not stripped
- setuid means that Set User Id is set on the file
	- eg. the sudo command
	- sets effective user as root
- dynamically linked -> to view which libraries are being shared,
	- ldd stack0