# ARQ, retransmission
- Resend packet until it is received properly
- Sender sends packet
- Receiver sends an ACK
- Sender waits until receives ACK
	- If does timeout, send packet again
# Windows of packets
![[Pasted image 20260618003633.png|551]]
- Sliding window
	- Left and right window edge denotes pacekts which **can be/have already been sent**
	- The left most one has not been acknowledged yet
	- Beyond right edge, cannot be sent

# Flow control
- Problem: **sender too fast** for receiver
	- Force sender to slow down
- 2 ways
	- Rate-based flow control
		- Sender has certain data rate allocation
		- Data never exceeds that allocation
	- Window-based flow control
		- **Window size note fixed**
		- Receiver signal senders how large window to use 
			- **Window advertisement/update**

# Congestion control
- Sender slows down to **not oerwhelm network** between itself and receiver
- Sender has to **infer** how much traffic the mmediate routers can handle by watching for packet loss and RTT changes

> The actual send rate at any moment is min(frwnd, cwnd), ie. rate between the 2 things above
> For reliable UDP project: throw out **both**. Reason: flow control don't need as SPSC ring buffer between threads is backkpressure mechanism. Congestion control don't want as we want it to be fast
# Retransmission timeout
- Sender should wait for the sum of
	- Time to send packet
	- Time for receiver to process it and send ACK
	- Time for ACK to travel back to sender
	- Time for sender to process ACK
- We have to estimate this

# TCP 
- Connection-oriented
	- Must establish connection first (3 way handshake)
- Reliable
- Byte steam abstraction
	- No record markers or message boundaries automatically inserted

## Reliability
- Convert **stream of bytes** to **set of packets** that IP can carry
	- **Packetisation**
	- TCP chooses what best-size chunks to send, typically fits each segment into a single IP-layer datagram that isn't fragmented
	- Different from UDP -> each write generates a single UDP datagram of that size plus headers
- Chunk passed b TCP to IP is a **segment**

- Checksum
	- No ACK sent if invalid checksum, just discarded

- Retransmission
	- Single retransmission for a group of segments
	- Sets a timer when it sends a window of data, updates timeout as ACKs receive
	- If ACK not recceived in time, segment retransmitted

- ACKS
	- Cumulative (byte numebr N means **all bytes** up to N (but not including N) have been received)

- Full duplex service
	- Data can flow in each direction, independent of other direction

# Headers
![[Pasted image 20260618005357.png|557]]
![[Pasted image 20260618005412.png|555]]

- **Socket**: IP + port
	-  Each TCP connection identified by a **pair** of sockets
- **Sequence number** 
	- 32 bit unsigned number, wrap around to 0
	- Identifies the **position** of the first data byte in this segment within the overall byte stream
		- This number counts bytes, not packets
	- Doesn't start at 0, instead starts at some random ISN (Initial Sequence Number) for security reasons
- **ACK number** 
	- The **next** **Sequence Number (see above)** the sender of this ACK **expects** to receive
- **Flags** (9 bits) — the control bits that drive TCP's state machine:
	- SYN — initiates a connection (used in the three-way handshake)
	- ACK — indicates the acknowledgment number field is **valid** (set on almost **every** segment after the initial SYN)
	- FIN — sender is done sending, initiates graceful shutdown
	- RST — hard reset, something went wrong, kill the connection immediately
	- PSH — push this data to the application immediately rather than buffering
	- URG — the urgent pointer field is valid (rarely used in practice)
	- ECE and CWR — explicit congestion notification, routers can mark packets instead of dropping them, and these flags let the endpoints respond
	- NS — nonce sum, a protection mechanism for ECN, also rarely relevant
- **Window** Size (16 bits) 
	- Receive window (rwnd) for flow control
		- how many bytes you can accept right now. 16 bits caps it at 64KB, which is why the window scale option exists to multiply this value.
- **Checksum** (16 bits)
	- Covers the header, data, and a pseudo-header (source IP, dest IP, protocol, length) from the IP layer
- **Urgent Pointer** (16 bits) — offset pointing to the end of urgent data. Only meaningful when URG is set. In practice almost nobody uses this — it was a design idea that didn't age well.
- **Options** (variable, 0–40 bytes) — the important ones are:
	- **MSS** (Maximum Segment Size) — exchanged during the handshake, tells each side the largest segment it can receive. Includes **only TCP data bytes, NO HEADER bytes**
	- **Window Scale** — multiplier for the window size field, negotiated at handshake, lets you have windows much larger than 64KB
	- **Timestamps** — used for more accurate RTT measurement and protection against wrapped sequence numbers (PAWS)
	- **SACK** (Selective Acknowledgment) — lets the receiver report non-contiguous blocks it has received, so the sender can retransmit only what's actually missing rather than everything after a gap

```cpp
frame [ packet [ segment [ application data ] ] ]
```
- Application hands data to TCP
- TCP creates a **segment** (Transport layer PDU)
- That segment (TCP header plus header) gets passed to network layer, which wraps it in IP header to form a **packet**
	- If segment is **larger than path MTU**, IP can split packet into multiple fragments
	- Receiving IP layer reassembles them before handing the completed segment back to TCP
- Packet then wrapped by data link layer in a **frame**

## No packet size?
- The receiver knkows the payload size from calculation, there is no explicit 'payload length' field
- Payload length = IP total length - IP header length - TCP header length
- From this, we can calculate the payload size, and the ACK number (next byte we expect to receive)
# MSS vs MTU
- MTU: Maximum Transmission Unit
	- Largest frame payload that a linke layer network can carry
	- Ethernet: 1500 bytes
	- Includes IP header, TCP/UDP header and application data
- MSS: **Maximum Segment Size**
	- **Largest TCP payload** that fits in a **single** segment without causing IP fragmentation
	- Relevance
		- If TCP sends a segment that, once wrapped in IP and ethernet headers, exceeds MTU, the IP layer has to fragment it
		- Fragmentation is bad: if any one fragment is lost, entire segment must be retransmitted
		- So TCP negotiates MSS during 3 way handshake
			- Both sides advertise MSS in SYN and SYN-ACK options, both sides respect the **smaller**

> Generally TCP is done without fragmentation. Just have one segment per frame.

> However, if fragmentation does exist, the fragments join up to form a segment (IP layer reassembles into a complete IP packet), then segments join up into a byte stream. TCP doens't even know fragmentation happened.

> Fragmentation is a IP concept, so TCP doesn't need to care about it. Just set MSS smaller than MTU to avoid it.