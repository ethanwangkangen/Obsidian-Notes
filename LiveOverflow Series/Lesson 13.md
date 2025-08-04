Answer for stack0

- in gdb
	- info proc mappings to see where stack is stored 

How the stack looks like after being set up:


For 32 bit architecture,

| Higher address    |                                   |                                                                                |
| ----------------- | --------------------------------- | ------------------------------------------------------------------------------ |
|                   | (optional) arguments passed to fn | *for 64-bit architecture, only after firs 6 args. First 6 passed via registers |
|                   | Return Address                    |                                                                                |
|                   | Old base pointer                  | Base pointer                                                                   |
|                   | ... Local variables               |                                                                                |
|                   |                                   | Stack pointer                                                                  |
| **Lower address** |                                   | <br>                                                                           |
GDB
- define hook-stop
	- info registers
	- x/24wx $esp
	- x/2i $eip
	- n
- this prints registers, stack and next 2 instructions whenever a break point is hit

