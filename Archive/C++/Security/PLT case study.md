main() calls fgets().
- break main
- disassemble main
	- 0x000055555555605f <+124>:	call   0x5555555550f0 <puts@plt>
	- call is similar to a jump function, except it returns back to the next line after it's done
	- 0x5555555550f0 <puts@plt>
		- This means jump to the instruction(s) at that address, which is the puts@plt stub.
- Now i want to analyse what's actually inside the stub
	- disassemble 0x5555555550f0
	- 0x00005555555550f4 <+4>:	bnd jmp *0x4f25(%rip)        
		- /# 0x55555555a020 <puts@got.plt>
	- The commented line is the calculated address that it jumps to. ie, 0x55555555a020 is the actual puts entry in the GOT that is used by the PLT.

With lazy evaluation
- GOT entries are not filled in before runtime. Only after first use
- This means that the GOT section must remaind writable during program execution!
- So essentially what i need to do is replace 0x55..a020 with the address of my win() function.

Board is global variable
- Stored in data segment
- So is GOT
- Analysis shows that board and GOT are closeby.
- board is char\[8]\[8]
	- in a 2d array, all elements of the first row (board\[0]) stored consecutively, then all alements of second row and so on.
	- each char is 1 byte, so each entry takes up one address
	- to calculate memory address.
		- address or board\[row]\[col] = starting address of board + (row_index * size_of_row) + (col index * size_of_element)
			- = 0x55555555a0a0 + (row * 8) + col

puts@got.plt is 0x80 behind that of &board.

