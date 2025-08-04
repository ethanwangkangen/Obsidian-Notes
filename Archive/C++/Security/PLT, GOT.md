# Procedure Linkage Table
- .plt section in executable's **text** segment
- for every dynamically linked function (eg. printf, fgets) there's a corresponding entry in PLT(eg. printf@plt)


- When I call puts()
	- it is compiled as puts@plt
	- puts@plt does:
		- if there is GOT entry for puts, jumps to address stored there
		- if no GOT entry, it resolves it and jumps there

- Key points
	- Calling the PLT address of a function is equivalent to calling the function itself
		- So if we have a PLT entry for a desirable libc function, can just redirect execution to its PLT entry.
	- GOT address contains addresses of functions in libc, and GOT is within the binary
		- Since it is part of binary, will always be a constant offset away from base.
		- So if PIE disabled/can leak binary base, you know the address that onctains a libc function's address.

https://ir0nstone.gitbook.io/notes/binexp/stack/aslr/plt_and_got