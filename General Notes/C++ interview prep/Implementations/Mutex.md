Things to note
- Copy deleted
- `flag.exchange(1, ..)`
	- 1 = locked
	- exchange return old. if exchange returns 0, it means it wasn't locked, therefore **this thread** is the one that locked it.
- `flag.load(..)` -> this is purely an optimisation
```cpp
class Mutex {
public:
	Mutex() = default;                        // needed: the deleted copy ctor below
    Mutex(const Mutex&) = delete;             // is user-declared, which suppresses
    Mutex& operator=(const Mutex&) = delete;  // the implicit default ctor
    void lock() {
        for (unsigned int i{0};
             flag_.load(std::memory_order_relaxed) ||
             flag_.exchange(1, std::memory_order_acquire);
             ++i) {

            if (i % 8 == 0 && i != 0) {
                std::this_thread::yield();
            }
        }
    }
	void unlock() {
		flag_.store(0, std::memory_order_release);
	}

private:
    std::atomic<unsigned int> flag_{ 0 };
};
```
