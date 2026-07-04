# Questions
- Benchmarked as 'measured over loopback' -> what do these numbers **not** include
	- NIC hardware path
		- DMA, interrupt handling, driver processing (loopback packets never touch the NIC, they U-turn inside the kernel)
	- Real network effects
		- Serialisation onto wire, switch forwarding, queuing jitter
	- What it **does still measure**
		- Syscall overhead, kernel protocol stack, memory copies, protocol logic
```
**A real packet's receive path** (transmit is the mirror image):

wire → NIC receives and validates the frame → DMA copies it into a ring buffer in RAM → interrupt fires (or NAPI poll picks it up) → driver wraps it in an `sk_buff` → IP layer → UDP layer → socket buffer → your `recvfrom` copies it to userspace.

**A loopback packet's path:**

your `sendto` → kernel builds the packet, routing table says destination is `127.0.0.1` → **U-turn**: the packet is handed straight back up the receive side of the stack, in software — IP layer → UDP layer → socket buffer → receiver's `recvfrom`.

No NIC. No wire. No DMA, no interrupt, no driver. The packet is bytes in kernel memory the entire time; "transmission" is essentially a function call.

```

