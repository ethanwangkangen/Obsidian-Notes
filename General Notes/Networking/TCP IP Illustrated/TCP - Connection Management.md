# Establishment and Termination
![[Pasted image 20260618012655.png|493]]
Establish:
- C->S: SYN
- C<-S: SYN ACK
- C->S: ACK
Terminate:
- C->S: FIN 
- S<-C: ACK, then FYN
- C->S: ACK

Half-Close operation
- One direction of the connection can terminate while the other continues until its closed

# Initial Sequence Number (isn)
- 32 bit counter that increments by 1 every 4 µs
	- To arrange for sequence numbers for segments on one connection to not **overlap** with those on a **new identical connection**
		- ie., connection had one segment delayed for long period of time, connection closed.
		- But then connection **opened again**, we don't want the delayed segment to be treated as valid data

# Timeout of connection establishment
- Eg. server host down, ARP entry for nonexistent host, etc.
- How frequently does client TCP send SYN to try and establish connection?
	- Eponential backoff: the time between tries doubles each time
- Linux
	- net.ipv4.tcp_syn_retries variables gives max number of times to resend SYN segment

# Notes on important flags
- MSS
	- Exchanged **only during handshake** in SYN and SYN-ACK
	- Each side advertises the **largest TCP payload** (HEADERS EXCLUDED it can receive
	- Both sides then use the **smaller** of the 2 values
	- Value derived from **local interface MTU** minus headers
		- If neither side sends MSS< TCP defaults to 536 bytes (576 byte minimum IP MTU minus 20 IP minus 20 TCP)
		- However, do note that **path between the 2 endoints** may have smaller MTUs. (that's how fragmentation can still happen)
			- This is where **Path MTU Discovery** happens
- Window scale
	- Also negotiated **only during handshake**
	- Window size field is 16 bits -> the value is multiplied by 2^n, where n is scale factor
		- Both sides propose scale factor in handshake
		- Note: scale factor applies to the **receiver's** advertised window


## Path MTU Discovery
- 