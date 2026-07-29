# Associativity
- Direct mapped
	- Each address maps to exactly one slot
- N-way set-associative
	- Each address maps to one set with N candidate slots
- Fully associative
	- A line can go anywhere
	- No conflict misses, but look up requires comparing against every slot in parallel
		- Expensive in hardware - only used for tiny structures like TLB

How address is encoded
- Tag | set index | offset
	- Offset = byte within the line
	- Set index: set
	- Tag: remaining bits to identify which address it holds
- Lookup
	- Compute set from index bits
	- Compare the tag against all N ways in the set in parallel
	- Match = it, no match = miss, one way is evicted to make room

Types of misses
- Compulsory/cold miss
	- First-ever access to a line
- Capacity miss
	- Working set is larger than cache
	- Lines get evicted because cache can't hold anything
- Conflict miss
	- The set that a line maps to is full, even though other sets have room
	- Does not apply for fully associative cache

# Write behaviour - cache coherency
- Write-through
	- Simplest way - if cache line is written into, processor immediately writes it into main memory
	- Slow
- Write-back
	- Cache line marked as dirty
	- When cache line is dropped from the cache at some point in the future, dirty bit will instruct processor to write the data back at the time instead of just discarding
	- Problem: 
		- Must assure that multiple processors see the same memory content at all times
		- If cache line is dirty on one processor, second processor must read from that processor's cache line instead

Solution to write-back
- Transfer cache content to other processor in case it is needed
- MESI cache coherency protocol
	- M (Modified): this core has the only copy, and it's dirty (differs from memory). Must write back on eviction.
	- E (Exclusive): this core has the only copy, clean (matches memory). Can be written without notifying anyone (silently upgrades to M).
	- S (Shared): clean copy that may also exist in other cores' caches. Read-only.
	- I (Invalid): line is not valid here; access = miss.
- Key transition
	- Core reads, no one else has it -> E. If another core has it -> S
	- Core writes: need line in M. If S, broadcasts invalidation -> all other copies go I, it goes M. If in E -> silently M
	- Another core reads a line you hold in M -> you supply data, both go S
- Just remember
	- **Writes require exclusive ownership**

Instruction cache - no writes
- L1i is separate cache from L1d (data), effectively read only
- No stores -> no dirty state, no write-back logic, simpler
- Optimised for sequential fetch + branch prediction feeding

# Paging
- Virtual address translation
	- Virtual page -> Physical page
- Page tables
	- One per process
	- PTE : one virtual page -> physical page mapping
- Multi level paging
	- Savings come from unmapped regions -> don't have to materialise table entries for them.
	- Process does not use entire virtual memory address space, no need to map for them
	- 4 levels -> requires 4 levels of sequential lookup in order to compute. Cannot be parallelised too.
- TLB
	- Acts as a cache for page table entries
	- Instead of storing the PTE, which would still require 4 lookups to compute the result, cache the result directly