Formula
```cpp
T curr = a.load(std::memory_order_acquire);
while (CONDITION(curr)) {
	if (a.compare_exchange_weak(curr, f(CURR)))
		return WON;
}
return EXIT;
```
- CONDITION = am i able to write at all, given the value is curr?
- f() = what do i want to write into the atomic, if it is curr?
- WIN = this thread made the change.

- If no need a loop, just use strong 
```cpp
bool try_reserve(std::atomic<uint64_t>& seq, uint64_t expected_gen) {
	return seq.compare_exchange_strong(expected_gen, expected_gen+1);
}
```