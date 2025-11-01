./stack0

variable at esp+0x5c set to 0

lea eax, \[esp+0x1c] -> note that unlike mov, for lea, the address is **not** dereferenced.
mov DWORD PTR \[esp], eax
call gets@plt

- move the address stored at eax into the esp, because this is 32 bit architecture, arguments are passed into functions through the stack 
	- similar to pushin on top the of the stack
	- in this case, just move it, because was already accounted for by compiler previously
- so the buffer starts from esp+0x1c, and ends right before esp+0x5c.
- need to fill in 4\*16=64 entries in the buffer, then 65th will modify the variable

CORRECT!
