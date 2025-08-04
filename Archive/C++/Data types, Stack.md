A bit - 0 or 1
A byte - 8 bits. eg. 11101000, which can hold 2^8=256 possible values.

Void means no type.

Data types and sizes
- char - 1 bytes
- int - 4 bytes (system dependent)
- float - 4 bytes
- double - 8 bytes
- pointer - 4 or 8 bytes

Memory address refers to a location in RAM
- Each memory address holds a SINGLE byte

----------------------------
What happens when i do 
- int x = 42;

Compile time:
- Space in memory (on stack) allocated for x
- Determines size of int (4 bytes)
- Assigns initial value to be stored(42)
- Converts the line into machine code instructions to reserve memory for 4 bytes and store binary representation of 42 into that space

Runtime:
- Memory is allocated.
- Say address 0x1000 used. Then 0x1000 to 0x1003 reserved for x
- 42 converted into binary: 0b 00000000 00000000 00000000 00101010
	- In hex, this is 0x 00 00 00 2A
- Order depends on CPU
	- Little endian: 0x2A 0x00 0x00 0x00
	- Big endian: 0x00 0x00 0x00 0x2A
	- This is assuming values are read from lower to larger address. ie
	- For little endian:
	  Address    Value
		0x1000     2A
		0x1001     00
		0x1002     00
		0x1003     00

Stack:
- Stack grows from HIGH address to LOW address.
- On Linux systems:
	- RSP: Stack Pointer, points to TOP of stack
	- RBP: Base Pointer, points to BASE of current stack frame

int foo(){
  int x = 42; // Line A
}

int main(){
  foo();
}

Below is the stack when Line A is being executed.

Higher addresses
┌────────────────────────────────────────────────────┐
│ 0x7FFF…E0E0  (unused above return address)       │
├────────────────────────────────────────────────────┤
│ 0x7FFF…E0D8  [return address into main]              <- **RIP**
├────────────────────────────────────────────────────┤       __
│ 0x7FFF…E0D0  [saved RBP of main]                       ← **RBP** = 0x7FFF…E0D0                  |   
├────────────────────────────────────────────────────┤         |
│ 0x7FFF…E0CC  [int x = 42]                                      ← [RBP−4]                                        |
├────────────────────────────────────────────────────┤         |
│ 0x7FFF…E0C8  (padding for alignment)                                                                         -foo
├────────────────────────────────────────────────────┤         |
│ 0x7FFF…E0C0                                                           ← **RSP** after `sub rsp,16`                |
└────────────────────────────────────────────────────┘     __ |
Lower addresses

Registers:
RBP: ...E0D0
RSP: ...E0C0

foo()'s frame is from E0C0 to E0D0 (RBP to RSP).

When foo returns,
- rbp reset to the "saved RBP of main"
- rsp reset to whatever the rsp of main was

