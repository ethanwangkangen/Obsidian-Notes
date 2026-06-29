- Does **not** provide 
	- Error correction
	- Sequencing
	- Duplicate elimination
	- Flow control
	- Congestion control
- Does provide
	- Error **detection**
	- End-to-end checksum 

- No connection setup (no handshake latency), no ordering guarantees, no retransmission, no control flow, no congestion control
- **Preserves** message **boundaries** (unlike TCP's byte stream)
	- ie. if we call **sendto() 3 times**, with "AAA", "BB", "CCCC", the receiver gets **exactly 3 recvfrom() calls** returning those things separately
	- Each send is its own discrete datagram, they never get merged or chopped up
	- VS TCP bytestream -> those 3 sends might arrive as one recv() or more than 3.
		- Hence why TCP applications need their own framing (length prefixes, delimiters etc.) to figure out where one message ends and the next begins

- Each UDP output operation requested by application produces exactly **one** UDP datagram, causes **one** IP datagram to be sent

# UDP header
- UDP header only 8 bytes
![[Pasted image 20260615184122.png]]
![[Pasted image 20260615184206.png]]

# Fragmentation - not handled
- UDP does not handle fragmentation, it is compleely unaware of it
- If a UDP datagram (IP header + UDP header + payload) exceeds path MTU, fragmentation happens at the **IP layer**
	- IP layer splits packet into fragments, each carrying **same IP identification field**, **fragment offset**, and a **More Fragments (MF)** flag set on all fragments except the last one
		- Only the first fragment contains the UDP header
	- The receiving IP layer reassembles them before delivering the complete datagram up to UDP
- Problem 
	- If **any** fragment is lost, the **entire datagram** is lost (UDP has no retransmission)
	- Hence why in practice, want to keep UDP datagram under path MTU 
		- typically ≤1472 bytes of payload on Ethernet (1500 MTU − 20 byte IP header − 8 byte UDP header)
# Integrity - UDP checksum
- Verifies integrity of the entire UDP datagram - header and payload
	- Catches bit flips, corruption, misdelivery
- Algorithm
	- One's complement sum
	- Take all the data being checksummed, treat it as a sequence of 16-bit words, add them all together using ones' complement arithmetic (where carry bits wrap around and get added back into the sum)
	- Then take the ones' complement (bitwise NOT) of the result
- Checksum computed over a 12-byte **pseudo-header** derived (solely) from fields in the IPv6 header
	- The pseudo-header is virtual, used only for purposes of checksum computation
	- Is never actually transmitted
	- For Ipv4, it consists of 
		- **Source IP address** (32 bits)
		- **Destination IP address** (32 bits)
		- **Zero** byte (8 bits of padding)
		- **Protocol number** (8 bits — 17 for UDP)
		- **UDP length** (16 bits)
	- Sender constructs this pseudo header, concatenates UDP header (with checksum field set to 0) and payload, compute ones' complement sum over whole thing, writes result into checksum field
	- Receiver constructs the same pseudo header from the IP layer it received, runs the same calculation, verifies

![[Pasted image 20260615184858.png]]

- IPv4 - UDP checksum is **optional**
	- If sender doesn't compute, field is set to 0x0000
- IPv6 - Checksum is **mandatory**