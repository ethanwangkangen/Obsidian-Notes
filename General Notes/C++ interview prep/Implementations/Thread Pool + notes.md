- Notes to remember
	- Remember - all accesses to the queue must be protected by mutex, **including .empty()**
```cpp
// Write your solution here
// C++23 using GCC 14.2
// Debug with std::cerr or std::clog.
// !!! IMPORTANT !!!
// 99% of headers are pre-compiled for you server-side.
// If your submission fails to compile due to a missing header, add it to your submission.

class ThreadPool
{
public:

    ThreadPool() 
    {
        int N = std::thread::hardware_concurrency();
        for (int i{}; i < N; ++i) {
            std::thread t{&ThreadPool::Work, this};
            threads_.push_back(std::move(t));
        }
    }

    ThreadPool(std::size_t count) 
    {
        for (int i{}; i < count; ++i) {
            threads_.emplace_back(std::thread{&ThreadPool::Work, this});
        }
    }

    ThreadPool(const ThreadPool&) = delete;
    ThreadPool (ThreadPool&&) = delete;
    ThreadPool& operator=(const ThreadPool&) = delete;
    ThreadPool& operator=(ThreadPool&&) = delete;

    ~ThreadPool()
    {
        // inform threads 
        finished_.store(true, std::memory_order_relaxed);

        // join all the threads
        for (auto& t : threads_) {
            if (t.joinable()) t.join();
        }
    }

    size_t ThreadCount() const 
    {
        return threads_.size();
    }

    template <typename Work>
    void submit(Work work)
    {
        // Pass callback to workers.
        std::scoped_lock lk(mtx_);
        queue_.push(std::function(work));
    }

private:

    // threads run this function
    void Work()
    {   
        // Perform actual work.
        while (true) {
            // break condition
            if (finished_.load(std::memory_order_relaxed)) break;

            // try pop
            std::unique_lock lk(mtx_);
            std::function<void()> f{};
            bool popped{false};
            if (!queue_.empty()) {
                popped = true;
                f = queue_.front();
                queue_.pop();
            }
            lk.unlock();
            try {
                if (popped) f();
            } catch (...) {
                // do something
            }
            
        }
    }
    std::atomic<bool> finished_ {false};
    std::mutex mtx_;
    std::queue<std::function<void()>> queue_;
    std::vector<std::thread> threads_;
};
```

# CV version
- cv predicate must also contain the stop checker
	- The stop checker change in destructor must be protected by a mutex
		- Once a thread is blocked in cv.wait(), **the only thing that can retrieve it is a notify on that same cv**
	- If stop checker change is not protected by mutex
		- Possible that stopped_ is changed (in the destructor) right after worker checks stopped_. In that case, it isn't actually notified, and since destructor has already run, nothing else will notify it ever again.
- Basically
	- Every cv.wait() failure causes thread to sleep, and something has to wake it so the function can return and join() can be called.
	- If that thing is in the predicate itself, make sure its protected by the same mutex