```cpp

int take_up_to(std::atomic<int>& n, int k) {
	int curr = n.load(std::memory_order_relaxed);
	int taken{};
	while (curr > 0 && taken <= k) {
		if (n.compare_exchange_weak(curr, curr-1)) {
			taken++;
		}
	}	
	return taken;
}