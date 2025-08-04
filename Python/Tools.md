
import struct
- eip = struct.pack("I", 0x80483f4) // little endian, for 32 bit
	- b'\xf4\x83\x04\x08'
- rip = struct.pack("<Q", 0x0000000000401234)
	- < means little endian
	- Q stands for unsigned long long (8 bytes) for 64 bit