```python

# encode() converts string/char to byte representation
# decode() converts byte representation to string representation

# String to bytes
s = "hello"
b = s.encode("utf-8")   # b = b'hello'
b = bytes(s, 'utf-8')
b = bytes([104, 101, 108, 108, 111]) # b'hello
b = b'hello'

# note that each char is a byte, so hello is 5 bytes.
# hello -> 0x68 0x65 0x6C 0x6C 0x6F

# Bytes to string
s2 = b.decode("utf-8")  # s2 = 'hello'

# Hex representation:
payload = b"AAAAAA"
# this is the same as
payload = b'\x41\x41\x41\x41\x41\x41'
# \x41 is hex 41 = dec 65 = A

payload += b"\x08\x04\x92\x96" [::-1] #reverse if little endian

```

**Bits and Bytes**
A bit = 1 or 0.
A byte = a **group** of 8 bits.

**Representations**
Binary, hexadecimal, decimal are just different ways of representing a number.
eg.
Binary: 0b00001000000001011001001010010110
Hexa: 0x08059296
Deci: 134582934

Each hex digit represents 4 bits.
2 hex digits represent 8 bits, or a whole byte.

So, say in 32-bit architecture,
The 32-bit address is shown as 0x08059296.

To convert this into bytes, i need to split up into groups of 2 hex digits.
so \x08 \x04 \x92 \x96

**Python representations**


```python
# integers

int i = 100
print(f"0x{i:x}) # 0x64
int j = 0x64 # The same as above
print(100 == 0x64) # True


# strings
hex_str = hex(i) # '0x64'

# bytes
bytes.fromhex(...) #hex string -> raw bytes. eg. 68656c6c6f -> b'hello'
b.hex()         #raw bytes -> hex string (opposite of above)


str.encode(...) # text -> bytes, eg. 'hello' -> b'hello'
bytes.decode(...) # bytes -> text

```


