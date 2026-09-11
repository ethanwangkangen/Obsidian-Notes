Backlog
- OOD
	- AVL tree/map (planned)
	- Order book recap (planned)
	- Vector (planned)
	- Range module (planned)
	- Thread pool recap (planned)
	- SPSC queues recap (planned)
	- LRU cache recap (planned)
	- File system recap (planned)
	- Pool allocator (lock free version) (planned)
	- std::function wrapper
	- Hashmap (planned)
- Currency exchange - djikstra/bfs
	- Leetcode review
- C++
	- Banks
	- Getcracked checklists
	- Review modern C++ tools, features
- OS
	- Banks
	- Ostep chapters(?)
- CA
	- Banks
	- Getcracked checklist
- Posix/Linux


# Aug 17 Mon
- Implementation
	- Hashmap (2h)
	- Ranges (30 min)
	- Vector recap (30 min)
- Leetcode recap (all) (2h)
- C++ all banks and checklists (2h)

# Aug 18 Tue
- Implementation
	- std::function (?) (2h)
	- std::function wrapper? (30 min)
	- SPSC caching + recap (30 min) - DONE
	- Map blank review (30 min)
	- Ranges - DONE
- Blockgraph and moldcast recap (resume)
	- recvmmsg (30 min)
- OS bank, networking recap (2h)
- C++ modern tools, features you've used.. (30 min)

# Aug 19 Wed

- Implementation
	- Pool allocator
	- Review everything else, especially Map + Orderbook (actually implement them)
- Recap notes
	- OS, Comp arch read through
	- C++
	- Posix + Linux commands
	- C++ - compilation
- Moldcast and Blockgraph 


# Aug 20 Thur
- Behavioural prep
	- Why squarepoint, resume stuff, etc.
- Interview day

# Aug 24
https://leetcode.com/problems/subarray-sums-divisible-by-k/
https://claude.ai/chat/a764a66b-7353-41cb-8a10-816b54cf4337
- CS3210 tutorial 0 notes
- More leetcode
- At least 1 implementation question

# Aug 31
- Networking
## Sept 1
- OS / Comp architecture
- Leetcode - categories, dp + greedy practice, ordered set implemtation stuff

## Sept 2

## Sept 3

## Sept 4 interview day
- Review resume + touchup

-------------------------------------------------------------
Final sprint list:
- Implementations
	- AVL tree/map
	- Order book recap (1x, review bugs)
		- Bugs
		- Deletion inside loop - dangling reference
		- Trying to insert with `[]`
	- Shared pointer (1x)
		- Take note of what release() does
	- Unique pointer, make_unique (1x)
	- Vector  (1x)
	- Sorted vector thing
	- Range module (1x, need recap)
	- Thread pool recap (1x)
		- Remember, threads are **move only**
	- Median from data stream
		- **Revisit**
		- Remember to push into one, then pop the top into the other to maintain invariant
		- Check the sizes between the 2 heaps
	- Thread safe bounded queue (1x)
	- LRU cache recap (1x)
	- Sliding window median
		- **Revisist!**
		- std::next(), multiset single element erase needs iterator erase, insert before changing mid iterator and erase after, > for insert and <= for erase etc.
	- LFU cache
	- File system recap 
	- Mutex (1x)
		- Acquire and release semantics, **NOT RELAXED**
	- Reader-writer lock
	- Pool allocator 
		- https://getcracked.io/problem/215/implement-a-pool-allocator
	- std::function wrapper
	- Hashmap 
	- Arcade high score board
		- https://getcracked.io/problem/214/arcade-high-score-board
		- Try it with multiset
	- Detecting endianness
- C++ notes and other stuff to learn
	- String parsing
	- Binary serialisation, endianness, allocators and memory
	- Numerical, floating point
	- Header memorisation
	- Utilities and constructs (tuple, std::function)
	- Memory and pointer stuff
	- uint32t etc.
- Resume, moldcast and blockgraph review
- Behavioural 
- Leetcode
	- 973 - closest points to origin : priority queue use.
	- 658 - same as above.
	- 75 - similar to what i did in round 2
	- 215 - again, priority queue
	- 715 - range module : **REVISIT THIS!**
	- 539 - string parsing: added to to-do list
	- 362 - design hit counter: binary search on sorted vector. easy
	- 295 - median from data stream
	- 239 - sliding window maximum (done before)
	- 146 - lru cache
	- 56, 57
	- 179
	- 399
	- 25
	- 122, 123
	- 1235
	- 1642
	- 621
	- 300, 354
	- Review pattern master list
- Concurrency
	- https://getcracked.io/problem/252/concurrent-stock-positions
		- CAS loop, thread local, shared mutex, atomicity, ordering
- Networking
- OS
	- Go through notes
	- Go through getcracked roadmap, while looking at notes
- CA 
- Optimisation practice
	- Get alude to generate
# Sept 8
- Completed:
	- Leetcode until 362
	- Orderbook recap
	- LRU recap
	- Range module recap

# Sept 9
- Completed:
	- Make unique
	- Shared ptr
	- Unique ptr
	- Mutex
	- Thread pool
	- Sliding window median
	- Median from data stream


# Sept 10
- Completed:
	- Vector
	- Bounded thread pool
	- Reading of C++ notes

# Sept 11
Plan
- Implementations
	- Review: pattern cheatsheet
		- LRU cache (done)
			- Bugs: capacity check happens whether or not put inserts or makes a new one
			- Forgot to update value in put()
		- LFU cache (done)
			- Bug: inside put(), evict FIRST before putting in
			- Use map() instead of unordered_map()
			- Remember how rehashing works for unordered_map
				- No rehashing in map() since its a tree
				- Map -> iterators invalidated, but **references still valid**
		- Interval set
			- Same pattern, **upper_bound** then **std::prev** for all 3: query, add and remove
			- Add: >=/<=, Remove: >/<
			- **Insert does nothing if already exists, DOEST NOT OVERWRITE!**
				- In remove(), when adding the left and right segments for prev, erase first then insert.
		- Most Frequent Ids
			- Simple ordered set
			- Use std::greater<> to compare a pair
				- If not, **must ensure that you compare both elements in the pair, not just one**
				- If not set may falsely make 2 diff pairs equivalent based on one element only
		- My Calendar II
			- Sweep line
			- My solution: order by endpoints, then one pass
				- Bug: early termination based on starting (which map is NOT sorted by!)
				- Bug: used map instead of multimap.
					- Remember, no matter what, **insert never overwrites**
						- Map, set -> dropped
						- Multimap, mutliset -> insert new pair
		- Sliding window median (todo)
		- Vector emplace back, std algorithm review (todo)
		- Free List (todo)
		- Heap + lazy deletion (don't need?)
		- File system
		- CRTP
			- Done
			- Remember base does all the work, including being templated. Derived just inherits
- C++ trivia master list read through + quiz myself with AI

# Sept 12 
Plan
- Implementations
	- Map
	- Hashmap
- Concurrency day
- Std::optional, std::function, std::variant
# Sept 13
Plan
- Implementations
	- File system
	- Pool allocator
	- Allocator on stack, malloc etc
- OS, CA day
- 3210 Assignment

# Sept 14
- Leetcode review
- String stuff
- Trie
# Sept 15
- Resume
- Behavioural
- Networking
- 

# Sept 16 Interview day

