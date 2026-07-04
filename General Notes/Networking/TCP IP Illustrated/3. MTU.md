# Maximum Transmission Unit
- **Largest** IP packet (in bytes) that a network link can carry **without fragmentation**
- Ethernet's default MTU is 1500 bytes
- If you send a packet larger than the path MTU,
	- Gets fragmented (IPv4 with DF bit clear)
	- or dropped (ICMP "Packet too biug" error)
- Path MTU Discovery (PMTUD) works by setting the DF bit and using those ICMP erros to learn the smallest MTU along the entire path