### 1. URL parsing and cache checks

The browser parses the URL into scheme (`https`), host (`example.com`), port (implied 443), and path (`/`). Before touching the network, it checks caches: is there a cached HTTP response? A cached DNS entry in the browser's own cache? Many interviewers skip this, but mentioning "cache first" signals you think about the fast path.

### 2. DNS resolution

The host needs to become an IP address. The lookup chain: browser cache → OS resolver cache (`systemd-resolved` on Linux) → `/etc/hosts` → the configured recursive resolver (your router or 8.8.8.8), queried over **UDP port 53**.

If the recursive resolver doesn't have it cached, it walks the hierarchy: queries a root server ("who handles `.com`?"), then the `.com` TLD server ("who handles `example.com`?"), then example.com's authoritative nameserver, which returns the A record (IPv4) or AAAA record (IPv6). Each answer carries a TTL controlling how long it may be cached.

Note that DNS itself is a UDP application — queries fit in one datagram, loss is handled by application-level retry with timeout. It's a tiny example of the same pattern your project implements: reliability built above UDP by the application. If the response exceeds ~512 bytes (or 4096 with EDNS0), it falls back to TCP — worth knowing as trivia.

### 3. Socket creation and connection

The browser calls `socket(AF_INET, SOCK_STREAM, IPPROTO_TCP)`, getting a file descriptor. It calls `connect(fd, {dst_ip, 443})`. The kernel assigns an ephemeral source port (~32768–60999 on Linux) — the 4-tuple `(src_ip, src_port, dst_ip, 443)` now uniquely identifies this connection in the kernel's connection table.

`connect` triggers the **three-way handshake**:

1. Client sends SYN with a randomized ISN, plus options: MSS (typically 1460), window scale factor, SACK-permitted, timestamps.
2. Server responds SYN-ACK — its own ISN, ACK = client ISN + 1, its own options.
3. Client sends ACK = server ISN + 1. Connection is ESTABLISHED on both sides.

This costs one full RTT before a single byte of application data moves. The option exchange in steps 1–2 is where MSS and window scaling get agreed — they can only be set here, never renegotiated mid-connection.

For HTTPS there's now a **TLS handshake** on top: ClientHello (supported cipher suites, SNI carrying the hostname), ServerHello + certificate, key exchange, and then symmetric encryption for everything after. TLS 1.3 got this down to one additional RTT (or zero for resumed sessions). For interviews you need the shape of this, not the cryptographic detail.

### 4. Sending the request — segmentation

The browser writes the HTTP request bytes into the socket via `send()`/`write()`. This is a **memory copy into the kernel's socket send buffer** (`SO_SNDBUF`) — the syscall returns when the data is buffered, not when it's delivered. Delivery is now entirely the kernel's TCP state machine's job.

TCP slices the buffered byte stream into **segments** of at most MSS bytes each. How much it's allowed to send right now is `min(cwnd, rwnd)` — congestion window and the receiver's advertised window. A fresh connection is in slow start, so cwnd starts around 10 segments. Nagle's algorithm may hold back a small final segment if unACKed data is outstanding (unless `TCP_NODELAY`).

Each segment gets a TCP header: source/dest ports, sequence number (the byte offset of this segment's first byte), ACK number, flags, current window advertisement, checksum.

### 5. IP encapsulation

Each TCP segment is handed to the IP layer, which prepends the IPv4 header: version, total length, identification, TTL (typically starting at 64 on Linux), protocol = 6 (TCP), header checksum, source and destination IP.

The kernel consults the **routing table** with the destination IP: longest prefix match. For an internet destination, no specific route matches, so the default route (`0.0.0.0/0`) wins, pointing at your gateway (e.g., `192.168.1.1`). Key subtlety: this decides the **next hop**, not the packet's addressing — the destination IP in the header stays `example.com`'s address the whole way.

### 6. ARP and the link layer

To build the Ethernet frame, the kernel needs the gateway's **MAC address**. It checks the ARP cache (`ip neigh`); on a miss, it broadcasts an ARP request — "who has 192.168.1.1?" — to `ff:ff:ff:ff:ff:ff`, and the gateway replies with its MAC. The result is cached with a timeout.

The frame is assembled: destination MAC = gateway's, source MAC = your NIC's, EtherType = 0x0800 (IPv4), payload = the IP packet, plus a CRC trailer. Note the addressing split that trips people up: **the IP addresses are end-to-end; the MAC addresses are per-hop.** Every link along the path rewrites the MACs; the IPs (NAT aside) never change.

The kernel hands the frame to the NIC driver, which places a descriptor in the NIC's TX ring buffer; the NIC DMAs the frame from RAM and serializes it onto the wire. This driver/ring/DMA machinery is exactly what DPDK bypasses the kernel to control directly — the connective tissue to your kernel-bypass talking points.

### 7. Per-hop routing across the internet

At each router: frame's CRC checked, Ethernet header stripped, IP header examined. The router decrements **TTL** (dropping the packet and sending ICMP Time Exceeded if it hits zero), recomputes the IP header checksum (it changed, because TTL changed), does its own longest-prefix-match lookup for the next hop, and re-encapsulates in a fresh L2 frame with new MACs for the next link.

Somewhere near your home router, **NAT** rewrites the source IP:port from your private address to the router's public one, recording the mapping so the response can be translated back.

Queuing happens at every hop — packets wait in router buffers when the outbound link is busy. This queuing delay is the variable component of latency, and buffer overflow here is where congestion loss actually occurs. On your project's LAN, this whole section collapses to "one switch, forwarding by MAC table, microseconds" — which is precisely why your protocol can skip congestion control.

### 8. Arrival and demultiplexing at the destination

The server's NIC receives the frame, validates the CRC, DMAs it into a receive ring buffer, and raises an interrupt (or gets picked up by NAPI polling under load). The driver wraps it in an `sk_buff` and pushes it up the stack:

- **IP layer**: validates the header checksum, confirms the destination address is local, handles reassembly if it was fragmented (for TCP with PMTUD, it won't be).
- **Demultiplexing step 1 — protocol field**: protocol = 6 hands the payload to TCP (17 would go to UDP).
- **TCP layer**: validates the TCP checksum, then **demultiplexing step 2 — the 4-tuple** `(src_ip, src_port, dst_ip, dst_port)` looks up the specific connection's socket. Sequence numbers are checked; in-order data is appended to that socket's **receive buffer**; out-of-order data waits in the reorder queue (head-of-line blocking, live and in person). An ACK is scheduled (possibly delayed).

For UDP, this stage is simpler — demux on destination port alone, datagram appended whole to the socket buffer, done. That simplicity is measurable nanoseconds, part of why your protocol lives on UDP.

### 9. Application read and response

The server process is blocked in `epoll_wait`/`accept`/`read`. The kernel wakes it — a scheduler operation with real latency cost (µs-scale, and the reason HFT busy-polls instead of sleeping). The app calls `read()`, copying data from the kernel receive buffer to userspace, parses the HTTP request, and writes a response — which traverses this entire path in reverse.

The browser receives the response bytes, and rendering begins — HTML parsing, subresource fetches (each potentially repeating this whole story, though usually reusing the connection), layout, paint.