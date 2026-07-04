# Networking & Systems Knowledge Checklist — Annotated

Reference for the reliable UDP project and HFT interviews. Each item has answer notes. Treat unchecked boxes as gaps to close.

---

## 1. IP Layer Fundamentals (TCP/IP Illustrated ch 1, 2, 5)

- [ ] **The layering model — L2/L3/L4 and why layering exists**
    
    - L2 (link, Ethernet): frames, MAC addresses, delivery within one network segment. L3 (network, IP): packets, IP addresses, routing across networks. L4 (transport, TCP/UDP): segments/datagrams, ports, process-to-process delivery.
    - Each layer encapsulates the one above: your payload → UDP header prepended → IP header prepended → Ethernet header/trailer.
    - Why: separation of concerns. IP doesn't care what it carries; Ethernet doesn't care where the packet is ultimately going. Lets each layer evolve independently.
- [ ] **IPv4 header fields**
    
    - Version (4), IHL (header length in 32-bit words, min 5 = 20 bytes), total length (16 bits → 65,535 max), identification + flags + fragment offset (fragmentation), TTL, protocol (6=TCP, 17=UDP), header checksum (header only, not payload), source IP, destination IP.
    - Header checksum is recomputed at every hop (TTL changes). It only protects the header — payload integrity is the transport layer's job.
- [ ] **TTL**
    
    - 8-bit counter decremented by 1 at each router. At 0, the packet is dropped and an ICMP Time Exceeded is sent back to the source.
    - Exists to kill packets stuck in routing loops.
    - Traceroute exploits it: send packets with TTL=1, 2, 3... and collect the ICMP Time Exceeded responses to map each hop.
- [ ] **IPv4 addressing: CIDR, subnet masks**
    
    - CIDR: `192.168.1.0/24` means the first 24 bits are the network prefix. Replaced classful addressing (A/B/C).
    - Subnet mask (`255.255.255.0` = /24) splits an address into network and host parts. Routers use `dest_ip AND mask` to decide if a destination is local or needs a gateway.
    - **Masks live in routing tables and interface configs, NOT in the IP header.** A packet only carries the destination address; each router applies its own table. (You already internalized this — it's a common misconception interviewers probe.)
- [ ] **Special addresses**
    
    - `127.0.0.1` loopback; `0.0.0.0` INADDR_ANY (bind to all interfaces); `255.255.255.255` limited broadcast.
    - Private ranges: `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16` — not routable on the public internet, need NAT.
    - Multicast: `224.0.0.0/4` (class D); `239.0.0.0/8` is administratively scoped ("private multicast") — use this for your project.
- [ ] **Loopback behavior**
    
    - Packets to 127.0.0.1 are short-circuited in the kernel — no NIC, no wire, no driver.
    - Benchmark implication: loopback numbers exclude NIC latency, interrupt handling, and real network jitter. Still valid for comparing protocol overhead (your protocol vs TCP under identical conditions), but say "over loopback" when quoting numbers.
- [ ] **MTU and the 1472 limit**
    
    - Ethernet MTU = 1500 bytes of L3 payload. Minus 20 (IP header) minus 8 (UDP header) = **1472 bytes** of UDP payload before fragmentation.
    - With IP options or tunneling (VPNs), the effective MTU is smaller.
- [ ] **Fragmentation**
    
    - If a datagram exceeds MTU, IP splits it into fragments; only the first carries the UDP header. Reassembly happens only at the destination.
    - Losing ONE fragment loses the whole datagram — the receiver can't reassemble, holds partial fragments until a timer expires, then discards. Loss probability multiplies.
    - Design consequence: keep every packet ≤ 1472 bytes so one datagram = one Ethernet frame.
- [ ] **Path MTU Discovery (PMTUD)**
    
    - Sender sets the DF (Don't Fragment) bit. If a router's next link has a smaller MTU, it drops the packet and returns ICMP "Fragmentation Needed" with the smaller MTU. Sender reduces segment size.
    - TCP does this automatically. UDP applications must handle it themselves — one more reason to just stay under 1472 on a LAN.
- [ ] **Routing decisions**
    
    - Router compares destination IP against its routing table using **longest prefix match** — the most specific matching route wins (a /28 route beats a /24 beats the default /0).
    - No match → default gateway (`0.0.0.0/0`). Hosts do the same: destination on my subnet → ARP directly; otherwise → send to gateway's MAC.

**Interview Qs:**

- _"What happens when you send a packet / type a URL?"_ — DNS resolution → socket creation → (TCP: handshake) → data segmented → IP encapsulation → ARP for next-hop MAC → frame on wire → per-hop routing (longest prefix match, TTL decrement) → destination reassembles → demultiplex by protocol + port → socket buffer → application read.
- _"Why is fragmentation bad for UDP?"_ — amplifies loss (lose any fragment, lose all), reassembly cost, some networks drop fragments outright.

---

## 2. UDP (ch 10)

- [ ] **The complete UDP header**
    
    - 8 bytes: source port (2), destination port (2), length (2, header+payload), checksum (2). That is the entire protocol.
    - Source port is optional for one-way traffic (can be 0) but you need it for NACKs to come back.
- [ ] **UDP checksum + pseudo-header**
    
    - Checksum covers UDP header + payload + a **pseudo-header** borrowed from IP: source IP, dest IP, protocol number, UDP length.
    - Why the pseudo-header: detects _misdelivered_ packets (right checksum, wrong host) — the checksum breaks if the IP addresses were corrupted in transit.
    - Optional in IPv4 (all-zeros = not computed), mandatory in IPv6. In practice always on.
- [ ] **What UDP does NOT provide**
    
    - No ordering, no reliability, no duplicate suppression, no flow control, no congestion control, no connection state.
    - Your project's one-line pitch: "I added back exactly the subset of these that market data needs — sequencing and retransmission — without the ones it doesn't."
- [ ] **Connectionless semantics**
    
    - No handshake; each `sendto` is independent; the kernel keeps no per-flow state.
    - Consequence: first packet can carry data (zero connection-setup latency), but also nothing tells you the peer exists or is listening (a datagram to a dead port may trigger ICMP Port Unreachable, which UDP senders usually never see).
- [ ] **Message boundaries**
    
    - UDP preserves them: one `sendto` = one `recvfrom`, always. TCP is a byte stream: two `send`s can arrive as one `recv`, or one `send` split across several.
    - Why it matters for market data: each update is a self-contained message; with TCP you'd need your own framing layer (length prefixes) on top.
	    - Eg. HTTP, FIX -> all carries framing machinery that UDP gives for free
- [ ] **Why/where packets drop**
    
    - **Receiver socket buffer overflow** — the #1 cause in practice and in your project. If the app reads slower than packets arrive, the kernel buffer (`SO_RCVBUF`) fills and the kernel _silently_ discards. Check `/proc/net/snmp` (UDP InErrors / RcvbufErrors) to prove it.
	    - Every socket has a **kernel-side-receive buffer**, queue of datagrams that have arrived and been processed by the stack, but not yet read by application
		    - Application drains this with `recvfrom` calls
		    - Size is `SO_RCVBUFF`
		- If sender sends faster than receiver drains, results in this overflow
    - Router/switch queue overflow (congestion, tail drop), checksum failure (corruption), TTL expiry (routing loops — not on a LAN).

- [ ] **Max datagram size**
    
    - Theoretical: 65,535 minus headers (~65,507 payload). Practical: 1472 to avoid fragmentation on Ethernet.
- [ ] **When UDP is the right choice**
    
    - Latency-critical traffic, loss-tolerant or custom-reliability applications, one-to-many (multicast), controlled networks where you don't need congestion control. Market data hits all four.

**Interview Qs:**

- _"TCP vs UDP?"_ — reliability/ordering/flow control/connection vs none of those; byte stream vs message boundaries; head-of-line blocking vs independent datagrams; handshake RTT vs zero-setup; congestion control vs constant-rate; per-connection state vs stateless.
- _"Why UDP for market data?"_ — multicast (one send, N receivers — impossible with TCP), no head-of-line blocking, no handshake, no congestion-control throttling, custom recovery tuned for the environment.
- _"How does UDP lose packets on a reliable LAN?"_ — receiver buffer overflow. The network delivered it; the host dropped it. This answer signals real systems experience.

---

## 3. TCP Core Mechanics (ch 12, 13)

- [ ] **TCP's guarantees and the byte-stream model**
    
    - Reliable (retransmission), ordered (sequence numbers), flow-controlled (window), congestion-controlled, connection-oriented, byte-stream.
    - Byte-stream: sequence numbers count _bytes_, not messages. No message boundaries — the application must frame its own messages.
- [ ] **TCP header fields**
    
    - Src/dst port, sequence number (32-bit, byte offset), ACK number (next byte expected), data offset, flags (SYN, ACK, FIN, RST, PSH, URG), window size (16-bit), checksum, urgent pointer, options (MSS, window scale, SACK-permitted, timestamps).
- [ ] **Three-way handshake**
    
    - Client → SYN (seq = client ISN). Server → SYN-ACK (seq = server ISN, ack = client ISN + 1). Client → ACK (ack = server ISN + 1).
    - Costs one full RTT before data flows. Your protocol: no handshake, first datagram is data.
    - Why three and not two: both sides must confirm the other can _receive_ (each side's sequence number must be acknowledged). Two-way would leave the server unsure its ISN arrived, and old duplicate SYNs could create phantom connections.
- [ ] **Why ISNs are randomized**
    
    - Security: predictable ISNs allow off-path attackers to inject/spoof segments into a connection.
    - Correctness: reduces the chance a delayed segment from an old incarnation of the same 4-tuple is accepted by a new connection.
- [ ] **MSS**
    
    - Maximum Segment Size, announced in SYN options. Typically 1460 on Ethernet (1500 − 20 IP − 20 TCP).
    - Not negotiated down to a common value — each side announces what it can receive; sender respects the peer's MSS and PMTUD.
    - No equivalent in your design — you just cap payloads at 1472 yourself.
- [ ] **Connection teardown**
    
    - Four segments: FIN → ACK → FIN → ACK. Each direction closes independently (half-close: I'm done sending, you may continue).
    - Either side can initiate. The side that sends the first FIN and receives the last data ends up in TIME_WAIT.
- [ ] **TIME_WAIT**
    
    - The active closer waits **2 × MSL** (Maximum Segment Lifetime; MSL commonly 30–60s, so 1–2 min) before fully closing.
    - Two reasons: (1) if its final ACK was lost, the peer retransmits FIN and someone must be there to re-ACK; (2) lets old duplicate segments from this connection die before the same 4-tuple can be reused.
    - Practical consequence: restarting a server fails with "address already in use" until TIME_WAIT expires — hence `SO_REUSEADDR`. You will hit this in development constantly.
- [ ] **RST segments**
    
    - Sent when: a segment arrives for a nonexistent connection (e.g., SYN to a closed port), or an application aborts (SO_LINGER 0).
    - Difference from FIN: FIN is orderly ("no more data from me, deliver what's buffered"); RST is abortive ("connection is dead, discard buffers immediately"). RST is not ACKed.
- [ ] **Half-open connections and SYN floods**
    
    - Half-open: one side thinks the connection is up, the other has no state (crash, reboot). Detected when data triggers an RST.
    - SYN flood: attacker sends SYNs with spoofed sources, never completes handshakes, exhausting the server's SYN backlog.
    - SYN cookies: server encodes connection state in its ISN instead of allocating memory, reconstructing state from the returning ACK — stateless handshake under attack.
- [ ] **TCP state machine (trace a normal lifecycle)**
    
    - Server: CLOSED → LISTEN → SYN_RCVD → ESTABLISHED. Client: CLOSED → SYN_SENT → ESTABLISHED.
    - Active close: ESTABLISHED → FIN_WAIT_1 (sent FIN) → FIN_WAIT_2 (FIN ACKed) → TIME_WAIT (received peer's FIN, sent ACK) → CLOSED after 2MSL.
    - Passive close: ESTABLISHED → CLOSE_WAIT (received FIN, app hasn't closed yet) → LAST_ACK (app closed, sent FIN) → CLOSED.
    - Many CLOSE_WAIT sockets in production = the application is failing to close() — a classic diagnostic question.
- [ ] **Simultaneous open/close**
    
    - Both sides SYN at once (SYN_SENT → SYN_RCVD → ESTABLISHED): legal, one connection results. Both FIN at once: both pass through CLOSING → TIME_WAIT. Rare; pure trivia, know they exist.

**Interview Qs:**

- _"Why three-way, not two?"_ — see above: both ISNs must be acknowledged; protection against old duplicates.
- _"What if the final handshake ACK is lost?"_ — server stays SYN_RCVD and retransmits SYN-ACK; if the client sends data, the data's ACK number completes the handshake implicitly.
- _"What is TIME_WAIT for?"_ — lost-final-ACK recovery + letting old segments die. Follow-up trap: TIME_WAIT is on the side that closes _first_ (usually the client — which is why servers should let clients close, or why busy clients exhaust ephemeral ports).

---

## 4. TCP Reliability & Retransmission (ch 14) — your project's main contrast

- [ ] **Cumulative ACKs**
    
    - ACK number = "next byte I expect" = everything before it received. Receiver can't say "I have 1,2,4,5 but not 3" (without SACK) — it just keeps ACKing 3.
    - Sender infers loss indirectly: duplicate ACKs or timer expiry. Slower than being told directly.
    - Sender holds unACKed data in **sender buffer** for possible retransmission
	    - In contrast, sender buffer in UDP is just staging area that drains to the NIC
- [ ] **RTO computation**
    
    - RTO derived from smoothed RTT: `SRTT = (1−α)·SRTT + α·RTT_sample`, plus a variance term `RTTVAR`; `RTO = SRTT + 4·RTTVAR` (Jacobson/Karels). Adapts to network conditions.
    - Exponential backoff: each retransmission of the same segment doubles the RTO.
    - Karn's algorithm: don't take RTT samples from retransmitted segments (you can't tell which transmission the ACK is for).
    - Your contrast: on a LAN, RTT is stable and tiny — an adaptive RTO is complexity you don't need. Fixed, configurable NACK timeout (~100µs) is simpler and faster.
- [ ] **Fast retransmit**
    
    - 3 duplicate ACKs → retransmit the missing segment without waiting for RTO. Still requires 3 more segments to arrive _after_ the loss before recovery starts.
    - Your NACK fires on the _first_ out-of-order packet — one packet of delay, not three.
- [ ] **SACK**
    
    - TCP option; receiver reports up to ~3-4 non-contiguous received blocks ("I have 1000–2000 and 3000–4000"). Sender retransmits only the actual holes.
    - Conceptually the closest TCP feature to your design — the difference is SACK rides on ACKs (positive reporting with gap info), while NACK is pure negative reporting with zero traffic when nothing is lost.
- [ ] **Head-of-line blocking**
    
    - Segment N lost → N+1, N+2... sit in the receiver's kernel buffer, invisible to the application, until N is retransmitted and arrives. One lost packet stalls the entire stream for ≥ 1 RTT (often RTO).
    - For market data: every buffered update is going stale while you wait. **This is the single most important concept in your interview story.**
    - Your design choice: optional immediate delivery with a gap flag — the application decides whether ordering or latency wins.
- [ ] **NACK vs ACK — be able to defend your choice cold**
    
    - NACK advantages: zero reverse traffic in the no-loss case (the common case on a LAN); receiver-driven and immediate (first gap detection → NACK now); scales to multicast (N receivers can't all ACK every packet — ACK implosion — but NACKs are rare).
    - NACK weaknesses (know these — they're the follow-up question): (1) receiver can't NACK a packet it doesn't know exists — trailing-packet loss is invisible → solved with heartbeats carrying the latest sequence number; (2) sender never learns what was received → retransmit buffer can't be safely trimmed based on receiver state, must be sized by worst case; (3) a lost NACK needs a re-NACK timer.

**Interview Qs:**

- _"How does TCP detect loss?"_ — two mechanisms: 3 dupACKs (fast) and RTO expiry (slow fallback).
- _"Why NACK for your project?"_ — the three advantages above, then preempt the weakness: "the trailing-packet problem is real, I handle it with heartbeats."

---

## 5. TCP Flow Control, Nagle, Congestion (ch 15, 16)

- [ ] **Sliding window / receive window**
    
    - Every TCP segment advertises the receiver's remaining buffer space. Sender may have at most `min(cwnd, rwnd)` unacknowledged bytes in flight.
    - Note: this 'buffer' is the **receiver socket buffer**
	    - Remember in UDP, if receiver socket buffer overflows, packets are just **silently dropped**
	    - TCP contrast-> buffer full -> **cannot lose data** because buffer's free space **is** the advertised receive window
	    - Sender is contractually barred from sending beyond it
	    - So overflow is prevented, but a full buffer results in **stalling** instead of **packet loss**
    - Zero window: receiver full → sender stops → sender sends periodic window probes to detect reopening (otherwise deadlock if the window-update segment is lost).
- [ ] **Window Scale option**
    
    - 16-bit window field caps at 64KB — too small for high bandwidth×delay paths. Window scale (negotiated in SYN) left-shifts the field by up to 14 bits (→ 1GB windows).
    - No equivalent in your design: no windows at all.
- [ ] **Silly Window Syndrome**
    
    - Receiver advertises tiny windows (app reads byte-by-byte) → sender sends tiny segments → 40 bytes of headers per few bytes of data.
    - Avoided by: receiver delays window updates until a full MSS of space exists; sender (Nagle) avoids sending tiny segments.
- [ ] **Nagle's algorithm**
    
    - Rule: if there's unACKed data outstanding, buffer small writes until an ACK returns (or a full MSS accumulates).
    - The classic deadlock with delayed ACKs: Nagle waits for an ACK; delayed ACK waits ~40–200ms before sending it → small request/response traffic gets stuck at ~40ms+ latencies.
    - `TCP_NODELAY` disables Nagle. Every trading system sets it. Your protocol never buffers — every message goes out immediately.
- [ ] **Delayed ACKs**
    
    - Receiver waits up to ~40ms (Linux) / 200ms (spec) hoping to piggyback the ACK on outgoing data or coalesce ACKs. `TCP_QUICKACK` disables per-socket (temporarily) on Linux.
- [ ] **Congestion control basics**
    
    - cwnd (congestion window) = sender's self-imposed limit, separate from rwnd. Effective window = min of both.
    - Slow start: cwnd starts small (~10 MSS today), doubles per RTT (exponential) until loss or ssthresh.
    - Congestion avoidance: past ssthresh, +1 MSS per RTT (linear, AIMD).
    - Loss signals: RTO → back to slow start (severe); 3 dupACKs → fast recovery: halve cwnd, retransmit, continue (mild).
    - Interpretation: TCP treats loss as "the network is congested, everyone slow down."
- [ ] **Why no congestion control in your protocol**
    
    - Its assumptions don't hold: you're on a dedicated LAN you control, not sharing the internet with strangers. Loss ≠ congestion (it's buffer sizing or corruption).
    - Its costs are unacceptable: slow start throttles a new/idle flow exactly when a market-data burst arrives; cwnd halving on stray loss adds latency variance to the tail.
    - Honest caveat if pushed: "on a shared or WAN network this protocol would be antisocial and would need rate limiting — the LAN scope is a deliberate design boundary."

**Interview Qs:**

- _"Flow control vs congestion control?"_ — flow control protects the _receiver_ (rwnd, advertised); congestion control protects the _network_ (cwnd, inferred). Sender obeys the min of both.
- _"Nagle + delayed ACK interaction?"_ — the deadlock above; the fix is `TCP_NODELAY`. Standard HFT question.

---

## 6. Multicast (ch 2 + Beej's/man pages)

- [ ] **Address range**
    
    - Class D: 224.0.0.0–239.255.255.255. 224.0.0.x is link-local reserved (routing protocols). 239.0.0.0/8 is administratively scoped — private multicast, use it for your project.
    - A multicast address identifies a _group_, not a host. Senders don't join; they just send to it.
- [ ] **IGMP**
    
    - Host↔router protocol: "I want traffic for group G." Kernel sends IGMP Membership Report when you call `IP_ADD_MEMBERSHIP`; router tracks which segments have members and prunes forwarding elsewhere. (PIM handles router-to-router multicast routing; know the name only.)
    - Switches use IGMP snooping to forward group traffic only to ports with subscribers.
- [ ] **How exchange feeds work**
    
    - Exchange sends each update once to a multicast group; the network fabric replicates; every subscriber NIC receives simultaneously — inherent fairness (no subscriber is "first" by connection order).
    - **A/B feeds**: exchanges broadcast two identical streams on different groups/paths. Subscribers listen to both and take whichever copy of each sequence number arrives first — halves effective loss and latency spikes. Excellent interview tidbit; mention your architecture could arbitrate A/B with its sequence numbers.
    - Gap recovery in real feeds: separate TCP "retransmission/replay" servers, or snapshot+incremental channels. Your NACK design is the LAN-scale version of this idea.
- [ ] **Socket mechanics**
    
    - Receiver: bind to the group's port (with `SO_REUSEADDR` so multiple local receivers can share it), then `setsockopt(IPPROTO_IP, IP_ADD_MEMBERSHIP, ip_mreq{group, interface})`.
    - Sender: plain UDP socket, `sendto` the group address. `IP_MULTICAST_TTL` (1 = link-local), `IP_MULTICAST_LOOP` (=1 to receive your own sends — needed for single-machine testing).
    - `SO_REUSEPORT` vs `SO_REUSEADDR`: REUSEADDR allows shared multicast binds & TIME_WAIT rebinding; REUSEPORT additionally load-balances unicast across sockets — for multicast receivers, REUSEADDR is the one you need.
- [ ] **Unicast vs broadcast vs multicast**
    
    - Unicast: one-to-one; N subscribers = N sends = sender bandwidth scales with N.
    - Broadcast: one-to-everyone-on-segment; wasteful, doesn't cross routers.
    - Multicast: one-to-subscribers; sender sends once regardless of N; the network does replication.
- [ ] **Why reliable multicast is hard**
    
    - ACK implosion: N receivers ACKing every packet floods the sender — doesn't scale.
    - Sender can't track per-receiver state at scale; receivers have different loss patterns.
    - Standard answer: NACK-based recovery (rare, loss-triggered) + heartbeats — exactly your design; PGM (RFC 3208) and Aeron are the reference points to name-drop.

**Interview Qs:**

- _"Why multicast instead of N TCP connections?"_ — bandwidth (send once), fairness (simultaneous delivery), no per-connection state/handshakes, no per-subscriber HOL blocking.
- _"How do you make multicast reliable?"_ — your entire project is the answer: sequence → gap detect → NACK → retransmit, heartbeats for trailing loss.

---

## 7. Sockets API & I/O Multiplexing

- [ ] **Core calls**
    
    - `socket(AF_INET, SOCK_DGRAM, IPPROTO_UDP)` → fd. `bind` attaches a local addr/port — required for receivers; senders skip it (kernel assigns an ephemeral port on first send, typically 32768–60999).
    - `sendto`/`recvfrom`: one datagram per call, address passed/returned per call. `recvfrom` with a too-small buffer silently _truncates_ the datagram (MSG_TRUNC flag tells you) — size buffers to max datagram.
- [ ] **sockaddr_in and byte order**
    
    - `sin_family = AF_INET`; `sin_port = htons(port)`; `sin_addr.s_addr = inet_pton(...)` or `INADDR_ANY`.
    - Network byte order = big-endian. `htons`/`htonl` on the way in, `ntohs`/`ntohl` on the way out — any multi-byte integer that crosses the wire or enters a sockaddr.
    - Forgetting `htons` on the port is the classic first bug: you bind port 8080 (0x1F90) but actually get 36895 (0x901F).
- [ ] **Key socket options**
    
    - `SO_REUSEADDR`: rebind through TIME_WAIT; shared multicast binds.
    - `SO_RCVBUF`/`SO_SNDBUF`: kernel buffer sizes. Linux _doubles_ the value you set (bookkeeping overhead) and caps at `net.core.rmem_max` — raise the sysctl for big buffers. Undersized RCVBUF = silent drops under burst.
    - `SO_TIMESTAMP`: kernel stamps arrival time, delivered as ancillary data via `recvmsg` — removes scheduling jitter from latency measurements.
    - `TCP_NODELAY`: disables Nagle (TCP benchmark side).
- [ ] **Blocking vs non-blocking**
    
    - Default: `recvfrom` blocks until data. `fcntl(fd, F_SETFL, O_NONBLOCK)` → returns −1/`EAGAIN` immediately when nothing's there.
    - HFT hot paths busy-poll non-blocking sockets (burn a core, save the wakeup latency); everyone else uses a multiplexer.
- [ ] **select vs poll vs epoll**
    
    - `select`: fd_set bitmap, 1024-fd hard limit, set destroyed each call (rebuild every time), kernel scans all fds — O(n).
    - `poll`: array of pollfd, no fd limit, still O(n) kernel scan per call.
    - `epoll`: interest list registered once in the kernel (`epoll_ctl`); kernel maintains a ready list as events occur; `epoll_wait` returns only ready fds — O(ready). Wins with many mostly-idle fds.
    - Honest nuance: with ONE socket (your project) the difference is negligible — you use epoll for the timeout-as-timer pattern and because it's the production idiom.
- [ ] **epoll API + timeout-as-timer**
    
    - `epoll_create1(0)` → epfd. `epoll_ctl(epfd, EPOLL_CTL_ADD, sockfd, &ev)` with `ev.events = EPOLLIN`. `epoll_wait(epfd, events, max, timeout_ms)` → n ready (0 = timeout).
    - Timer pattern: timeout = time until next NACK-retry deadline; `epoll_wait` returning 0 = the timer fired, re-send NACKs. (Finer than ms → `epoll_pwait2` takes a timespec, or busy-poll.)
- [ ] **Level-triggered vs edge-triggered**
    
    - LT (default): fd reported ready as long as data remains — you may read partially and return.
    - ET (`EPOLLET`): reported only on _new_ arrivals — you must drain the socket to EAGAIN in a loop or risk stalling forever on buffered data. Pairs with non-blocking mode by necessity.
    - ET saves redundant wakeups in high-fanout servers; LT is simpler and correct-by-default. Use LT; know both.
- [ ] **sendmsg/recvmsg, iovec, cmsg**
    
    - `iovec{base, len}` array in `msghdr` = scatter/gather: send header + payload from two buffers in one syscall, no coalescing copy.
    - `msg_control` carries ancillary data (`cmsghdr` chain): walk with `CMSG_FIRSTHDR`/`CMSG_NXTHDR`, match `cmsg_level == SOL_SOCKET && cmsg_type == SO_TIMESTAMP`, extract `timeval` from `CMSG_DATA`.
- [ ] **NIC → recvfrom path (conceptual)**
    
    - Packet hits NIC → DMA into a ring buffer → interrupt (or NAPI polling under load) → driver wraps it in an `sk_buff` → IP layer (checksum, routing decision: local) → UDP layer (checksum, demux by dst port) → appended to the socket's receive buffer → blocked reader woken → `recvfrom` copies kernel→user.
    - Latency taxes on this path: interrupt handling, scheduler wakeup, kernel/user copy, syscall overhead. This list is your segue to kernel bypass.
- [ ] **Kernel bypass (discussion-level, not implementation)**
    
    - DPDK: userspace poll-mode drivers; NIC ring buffers mapped into the process; no interrupts, no syscalls, no copies; burn cores busy-polling.
    - Solarflare Onload: intercepts the sockets API, implements TCP/UDP in userspace over the NIC — kernel bypass _without_ changing application code.
    - RDMA: NIC writes directly into remote application memory; CPU on the remote side not involved.
    - The "how would you make it faster" ladder: busy-poll non-blocking sockets → `recvmmsg` batching → SO_TIMESTAMPING hardware stamps → Onload → DPDK → FPGA. Know the ladder; say you scoped the project to the sockets API deliberately.

**Interview Qs:**

- _"select vs poll vs epoll?"_ — the O(n) vs O(ready) + registration model answer above. Bonus: mention `kqueue` (BSD) / IOCP (Windows) exist.
- _"What happens in the kernel when a packet arrives?"_ — the NIC→socket path above, then pivot to where the latency goes.

---

## 8. Atomics, Memory Ordering & Lock-Free (C++ Concurrency in Action ch 5, 7)

- [ ] **Data race definition**
    
    - Two+ threads access the same memory location, at least one is a write, no happens-before ordering between them → undefined behavior. Not "might get a stale value" — UB, full stop.
- [ ] **std::atomic vs volatile** _(guaranteed question)_
    
    - `atomic`: indivisible operations + memory-ordering control + participates in the C++ memory model (establishes happens-before).
    - `volatile`: only tells the _compiler_ not to cache/elide/reorder the access relative to other volatiles. No atomicity (torn reads/writes possible), no CPU-level ordering, no inter-thread synchronization. It's for memory-mapped hardware registers, not threads.
- [ ] **Memory orderings**
    
    - `relaxed`: atomicity only, no ordering. Use: statistics counters where you only need the final total.
    - `acquire` (loads): no subsequent reads/writes in this thread may be reordered _before_ the load. "Subscribe."
    - `release` (stores): no preceding reads/writes may be reordered _after_ the store. "Publish."
    - `acq_rel`: both, for read-modify-write ops.
    - `seq_cst` (default): acquire+release + a single global total order over all seq_cst ops that every thread agrees on.
    - The pairing rule: a release store _synchronizes-with_ an acquire load that reads the stored value → everything before the release happens-before everything after the acquire.
- [ ] **x86 vs ARM memory models**
    
    - x86 (TSO — total store order): strong; only store→load reordering happens in hardware. Plain `mov` already gives acquire/release semantics — those orderings are _free_ on x86. `seq_cst` stores need `mfence` (or `xchg`) — that's the cost you avoid.
    - ARM: weak — loads and stores reorder freely; acquire/release need real barrier instructions (`ldar`/`stlr`).
    - The trap: code with wrong orderings often _works on x86_ because the hardware is forgiving, then breaks on ARM — and the compiler can still reorder on x86 too. Correctness reasoning must use the C++ model, not the hardware.
- [ ] **Compiler vs CPU reordering**
    
    - Both exist; both are constrained by atomics/fences. `relaxed` still stops the compiler from tearing or eliding the access, but permits reordering around it.
- [ ] **compare_exchange weak vs strong**
    
    - CAS: `if (*ptr == expected) { *ptr = desired; return true; } else { expected = *ptr; return false; }` atomically.
    - `weak` may fail _spuriously_ (returns false even when values matched — LL/SC architectures). Use in loops (you retry anyway, and it's cheaper). `strong` for one-shot attempts.
- [ ] **ABA problem**
    
    - Thread 1 reads A, gets preempted; thread 2 changes A→B→A (e.g., pops a node, pushes a recycled node at the same address); thread 1's CAS sees "still A" and succeeds on stale assumptions → corruption.
    - Mitigations: tagged pointers (pointer + generation counter CAS'd together), hazard pointers, epoch reclamation.
    - Your SPSC queue is immune: indices only move forward monotonically and each is written by exactly one thread — no CAS at all, so no ABA. Know _why_ you're immune; it's the natural follow-up.
- [ ] **Lock-free vs wait-free vs obstruction-free**
    
    - Obstruction-free: a thread running alone finishes in bounded steps. Lock-free: _some_ thread always makes progress (individual threads may starve). Wait-free: _every_ thread finishes in bounded steps.
    - Your SPSC queue is wait-free: push and pop are a fixed number of instructions, no loops, no retries.
- [ ] **False sharing**
    
    - Cache coherence works in 64-byte lines (MESI: a core writing a line invalidates every other core's copy). Two variables on one line, written by different threads → the line ping-pongs between cores even though the threads never touch each other's data.
    - Your queue: head (consumer-written) and tail (producer-written) MUST be on separate lines → `alignas(64)` (or `std::hardware_destructive_interference_size`).
    - Detection: `perf c2c` on Linux, or the symptom — adding padding massively speeds things up.
- [ ] **Your SPSC queue, cold** _(you must be able to whiteboard this)_
    
    - State: `buffer[capacity]` (capacity = power of 2), `alignas(64) atomic<size_t> head` (consumer's), `alignas(64) atomic<size_t> tail` (producer's).
    - Invariants: empty ⇔ `head == tail`; full ⇔ `((tail+1) & (capacity−1)) == head`; one slot always wasted to distinguish the two.
    - Push (producer): `load head (acquire)` → full? return false → write `buffer[tail]` → `store tail+1 (release)`.
    - Pop (consumer): `load tail (acquire)` → empty? return false → read `buffer[head]` → `store head+1 (release)`.
    - Justify every ordering: producer's _release on tail_ publishes the data write; consumer's _acquire on tail_ makes it visible — this pair is the correctness core. Consumer's release on head publishes "slot free"; producer's acquire on head sees it before overwriting. Own loads of your own index can be relaxed (single writer = you).
    - Power-of-2 → `& (capacity−1)` instead of `%` (division is ~20–40 cycles; AND is 1).
- [ ] **Cached-index optimization** _(depth points)_
    
    - Producer keeps a plain (non-atomic) copy of the last head it saw; only reloads the real atomic head when the cached value says "full." Same mirror-image trick for the consumer with tail. Cuts cross-core cache traffic dramatically in the common non-full/non-empty case. This is what Folly/rigtorp-style queues do.

**Interview Qs:**

- _"Implement/walk through an SPSC queue"_ — the whiteboard spec above, orderings justified line by line.
- _"What's false sharing, how do you find and fix it?"_ — MESI line ping-pong; `perf c2c`; `alignas(64)`.
- _"Why is seq_cst slower than acquire/release?"_ — the mfence on x86 stores + the global total order constraint on the optimizer.

---

## 9. Serialization & Wire Formats

- [ ] **Struct padding and alignment**
    
    - Members are aligned to their natural alignment; compiler inserts padding. `{uint8_t, uint32_t, uint16_t}` → 1 + 3(pad) + 4 + 2 + 2(tail pad) = 12 bytes, not 7.
    - Rule of thumb: order members largest-to-smallest to minimize padding.
    - Always `static_assert(sizeof(Header) == N)` on wire structs — catches silent layout surprises at compile time.
- [ ] **Packed structs vs field-by-field**
    
    - `#pragma pack(push,1)` / `__attribute__((packed))` removes padding → struct layout == wire layout → single memcpy in/out. Caveats: taking a pointer/reference to a misaligned member is UB; unaligned access is free on x86, potentially faulting on other ISAs.
    - Field-by-field: `memcpy` each field at explicit offsets with byte-order conversion. More lines, zero portability caveats.
    - Either is defensible; packed + static_assert is fine for an x86 LAN project — but be ready to state the caveats.
- [ ] **Endianness on the wire**
    
    - Pick one and document it. Network order (big-endian, htonl/ntohl) is the convention; little-endian is defensible on all-x86 (SBE uses LE for exactly this reason — zero conversion cost on the machines that matter). The interview answer is knowing it's a _decision_, not a default.
- [ ] **memcpy, not reinterpret_cast** _(ties to your casts knowledge)_
    
    - `reinterpret_cast<uint32_t*>(buf + off)` then dereference = strict-aliasing UB (+ potential misalignment). `memcpy(&x, buf+off, 4)` is defined and compiles to the same single load. `std::bit_cast` for whole-object puns in C++20.
- [ ] **Your header design — justify each field**
    
    - Typical: magic/version (1–2B, reject garbage), msg type (1B: DATA/NACK/HEARTBEAT), flags (1B), payload length (2B), sequence number (4–8B), send timestamp (8B, for latency measurement). ~16–24 bytes.
    - Seq width tradeoff: uint32 wraps at 4.3B messages (at 1M msg/s = ~71 min) — either handle wraparound (serial number arithmetic, RFC 1982) or use uint64 and never think about it. Saying this unprompted scores points.
- [ ] **SBE vs protobuf vs JSON (conversation-level)**
    
    - JSON: text, slow parse, allocations — never on a hot path. Protobuf: compact binary but varint decoding + allocations. SBE/FlatBuffers: fixed offsets, zero-copy — _access_ fields in place without a decode step; SBE is the CME/exchange standard. Your custom header is a hand-rolled SBE-style fixed layout; say so.

---

## 10. Protocol Design (your design decisions — own every one)

- [ ] **Sequence numbering** — monotonic per-datagram counter; gap in sequence = loss detected. Decide start value (0 vs random), width (see above), and what a restart of the sender means (epoch/session id field avoids ambiguity — worth mentioning).
- [ ] **Gap detection** — receiver tracks `expected_seq`; arrival of seq > expected → NACK [expected, seq−1], buffer the arrival, advance.
- [ ] **Retransmit buffer** — sender-side ring of last N packets. Size = worst-case loss burst × recovery time. NACK for an evicted seq = unrecoverable → surface to application (callback/flag), don't hide it.
- [ ] **Reorder buffer** — receiver-side holding pen for out-of-order arrivals; drain in-sequence once the gap fills. Bound its size; decide the policy when it overflows (drop new vs declare loss).
- [ ] **Delivery semantics** — ordered (TCP-like, buffer until contiguous) vs immediate-with-gap-flag (app handles it). Configurable = best interview answer; know which mode your benchmarks used.
- [ ] **Heartbeats** — periodic message carrying the latest seq; solves trailing-loss invisibility and doubles as liveness detection. Interval = tradeoff between worst-case trailing-loss detection latency and overhead.
- [ ] **NACK retry timer** — NACKs are datagrams too; they get lost. If a gap persists past the timeout, re-NACK. Cap retries → unrecoverable-loss path.
- [ ] **Duplicate suppression** — retransmits + re-NACKs create dupes; `seq < expected` or already-buffered → discard silently.
- [ ] **Failure modes you handled** — enumerate them proactively in interviews: lost data packet, lost NACK, lost retransmission, lost trailing packet, retransmit-buffer eviction, reorder-buffer overflow, slow consumer (SPSC full — spin vs drop policy).

---

## 11. Benchmarking & Measurement

- [ ] **Clocks** — `std::chrono::steady_clock` / `CLOCK_MONOTONIC`: monotonic, immune to NTP jumps — always this for intervals, never `system_clock`. Reading a clock costs ~20–30ns (vDSO, no syscall on Linux).
- [ ] **Cross-process timestamping** — sender stamps in the packet, receiver stamps on arrival; both clocks are the same machine's on loopback, so the delta is valid. Across machines it isn't (clock offset) — another reason to state "loopback/LAN, same host" in the README.
- [ ] **Kernel timestamps** — `SO_TIMESTAMP` + recvmsg cmsg = arrival time before your process was scheduled; separates network+kernel latency from scheduling jitter.
- [ ] **Percentiles, not averages** — HFT lives in the tail. Report p50/p99/p99.9/max; the mean hides everything interesting. ≥100k samples for p99.9 to be meaningful; sort-and-index is fine at this scale.
- [ ] **Warmup + methodology hygiene** — discard the first thousands of samples (cold caches, page faults, branch predictor training); pin threads (`taskset`/`pthread_setaffinity_np`) to stop the scheduler migrating them mid-run; note CPU frequency scaling (performance governor) — mentioning any of these signals you measure seriously.
- [ ] **Fair TCP comparison** — same payload sizes, same rate, `TCP_NODELAY` on, loopback both. Expect: comparable p50 (loopback is cheap for both), your win at p99+ under loss (no HOL blocking, no cwnd collapse). If TCP wins raw zero-loss throughput, say so — honesty about where you lose reads as engineering maturity.
- [ ] **Loss injection** — drop every Nth (or Bernoulli p) at the sender before sendto; sweep 0.1%/1%/5%; plot recovery-latency distribution vs loss rate. This chart is the centerpiece of the README.

---

## 12. Adjacent Topics That Will Come Up

- [ ] **Threading basics (from your CCiA reading, not the project)** — mutex/lock_guard/unique_lock/scoped_lock distinctions; condition_variable + predicate loop (spurious wakeups); deadlock conditions and lock-ordering discipline; why your project _avoids_ all of this (SPSC invariant) — being able to say "I know the locking toolbox and chose not to need it" is the strongest framing.
- [ ] **OS: what a syscall costs** — mode switch, ~50–100ns+ before any work; why batching (`recvmmsg`) and bypass exist.
- [ ] **OS: context switches & scheduling jitter** — µs-scale cost, cache pollution; why HFT pins and busy-polls.
- [ ] **Cache hierarchy numbers (rough)** — L1 ~4 cycles / ~1ns, L2 ~12 cycles, L3 ~40 cycles, DRAM ~60–100ns, cross-core cache line transfer ~40–100ns. Anchor your queue latency claims to these.
- [ ] **Your allocator project crossover** — why no allocation on the hot path: malloc takes locks, unbounded tail latency, cache pollution; your protocol pre-allocates every buffer at startup. Connects your two projects in one sentence.
- [ ] **BFT research crossover** — consensus = reliable ordered delivery over unreliable networks; sequence numbers, retransmission, and duplicate suppression appear in both; have the one-paragraph bridge rehearsed.

---

## How to use this

1. First pass: tick what you already know cold; the unticked items are your reading targets for the prereq days.
2. During implementation: revisit sections 8–11 — they turn from theory into things you've debugged.
3. Interview prep (Sept): use the _Interview Qs_ under each section as mock-question prompts; answer out loud, 60–90 seconds each.