# Reading 
- **OSTEP** — ch. 6 (syscalls/context switches), 13–16 (address spaces, quick reads), 18–20 (paging/TLB), 21–22 (swapping/page faults). ~8 hrs. Do first — rebuilds forgotten CS2106 and gentles you into Drepper.
- **Drepper** — §3 (caches; work one set/way/tag example by hand) and §4 (virtual memory, hardware side). Skip §2, §5. ~6–8 hrs.
- **Preshing posts** — memory barriers, acquire/release. ~4–6 hrs. Do after Drepper — lands better with the cache picture rebuilt, and it's your coldest topic.
- **Cloudflare networking posts** — interleave with MoldCast sessions. ~2–3 hrs.


# Learning
- STL containers and iterators
	- eg. set, map, deque
	- Iterator vs pointer invalidation
	- Methods
- Exceptions
- Std::function
- Std::variant
- Std::optional
- Variadic templates
- C++20 template changes 
	- Sfinae, concepts, constrained auto
- Alignment
	- Alignof, alignas..
- Template
	- Recap on specialisation preferences
	- Static members
- PIMPL idiom
- ring buffer, SPSC queue, circular queue
	- https://getcracked.io/problem/53/rate-limited-pubsub?language=Cpp
	- circular queue in answer here

# Doing
- Mistake ledger
- Start practising LLD/OOP questions.
- Recap on the alloc/malloc questions
- Recap concurrency primitives: reader-writer lock,...
- Circule queue, ring buffer, SPSC...

# Misc
- https://myntbit.com/training/cpp-zero-copy-span-parsing
	- How to work with low-level pointers, memcpy, spans, arrays etc (recap moldcast as well)
- https://myntbit.com/training/fpga-mmio-order-gateway
	- Something about reinterpret cast to volatile?
- String parsing 




