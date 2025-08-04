Setup
- sudo pip install pwn

- from pwn import *

Connecting to server
- target = remote("github.com", 9000)

Connecting to target binary
- target = process("./challenge)

Then,
- print(target.recv())
- print(target.recvuntil(b'end'))

- target.send(b'1234')
- target.sendline(b'hello')

- target.interactive()

Attach gdb for debugging:
- gdb.attach(target)

Packing/Unpacking
- p32(x)/p64(x) -> pack 32/64-bit int into bytes
	- eg.
	- p32(0xdeadbeef) -> b'\xef\xbe\xad\xde'
- u32(b)/u64(b) -> the reverse

Cyclic tools
- cyclic(n)
	- b'aaaabaaacaaadaaaeaaafaaa..'
- cyclic_find(val)
	- find the offset of a value in the pattern
	- cyclic_find(0x6161616c)

ELF and libc analysis
```c
elf = ELF('./challenge')

elf.symbols['main']        # Address of main
elf.plt['puts']            # Address of puts@plt
elf.got['puts']            # Address of puts@got

libc = ELF('/lib/x86_64-linux-gnu/libc.so.6')
libc.symbols['system']     # Address of system()
libc.search(b'/bin/sh')    # Address of "/bin/sh"

```
