```cpp
bool increment_if_nonzero(std::atomic<int>& n) { 
	int m = n.load(std::memory_order_relaxed); 
	bool changed = m.compare_exchange_weak())
}
```