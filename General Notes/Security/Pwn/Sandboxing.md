- Second line of defense
	- Attack happens -> minimise impact
- Reference monitor: piece of code that checks all references to object
	- Can put in kernel space or program itself
	- If in kernel: need higher privilege
	- Check access rights
- Syscall sandbox: a reference monitor for protecting OS resourse objects from an app

Example
- Just monitor the syscall that program makes
	- eg. Exec() -> which spawns a new program
	- if this seems malicious/suspicious, just block
- Different policies can be made
	- Linux seccomp
		- no syscalls except exit() sigreturn(), read(), write()..
	- Seccomp-bpf
		- configurable policies

Policy design
- Allow listing > Block listing

3 security principles
- Separation of Concerns
	- Separate policy from its enforcement
- Minimise Trusted Code Base (TCB)
	- Reduce what one needs to trust
	- Separate verifier from enforcement
	- eg. compiler many lines of code, but just need to write a checker to gurad memory access and trust the checker
- Least privilege
	- Each function only given necessary privileges
	

Namespace isolation
- Namespaces: feature of linux kernel
	- One set of processes sees only one set of resources, another sees another set

Privilege separation
- eg. SSH server, separate the network and filesystem reading code.