- What are smart pointers, when were they introduced?
    - What preceded smart pointers, why was it deprecated in C++11 and removed in C++17?
    - `auto_ptr`
	    - Removed because it did move semantics when copying

- What is a `std::shared_ptr`?
	- What is it
		- Wrapper over a raw pointer, with pointer to a control block that stores reference count. Deletes pointer when reference count hits 0
    - Is it threadsafe?
	    - The **reference count** operations are **atomic** - can safely copy/destroy `shared_ptr` instances across threads
		    - However, the **pointed-to-object** is not protected
    - Why do some people consider using `std::shared_ptr` _slow_ or _expensive_?
	    - Atomic reference count increments and decrements are more expensive
	    - Also the control block heap allocation (unless `make_shared`) is used
	- Given this, how should shared pointers be used in code?
		- Pass `shared_ptr` by `const&` when a function just needs to use the object
		- Only pass by **value** when function needs to take shared ownership
    - What is its size and, as a follow-up, what's its data members?
	    - 2 pointers, one to data and one to control block
        - What's in its control block?
	        - Contains reference count, weak reference count, deleter, allocator, and if make_shared was used, the object itself is embedded in control block
            
-  What are the three advantages of using `std::make_shared` instead of passing in a raw pointer to `std::shared_ptr`'s constructor?
	- Single allocation - allocated object and control block together in one `malloc` call, instead of 2
	- Exception safety - due to argument evaluation order issues
		- `foo(std::shared_ptr<A>(new A), std::shared_ptr<B>(new B))`
		- Eg. `new B` throws, the memory from `new A` leaks 
	- Less redundant code

- When would you use `std::shared_from_this`?
	- When an object managed by `shared_ptr` needs to obtain a `shared_ptr` to itself

- What is `std::unique_ptr`?
    - What are some of its characteristics?

- Can memory leak with shared pointers?
    - What is a weak pointer, when is it used?
    
- How do you provide a customer deleter to a smart pointer, and when should you do so?
	- unique_ptr: through type template
	- shared_ptr: just pass in constructor
	- Use when managing resources that aren't freed with `delete`

---

Be familiar with all smart pointer's APIs. For example, `release` and `reset` on unique pointer, `lock` and `expired` on weak pointer, etcetera.

**`unique_ptr`:**

- `get()` — raw pointer, no ownership change
- `release()` — returns the raw pointer and relinquishes ownership (sets internal pointer to null). Caller is now responsible for deletion. This does _not_ delete anything.
- `reset(ptr)` — deletes the currently owned object, takes ownership of `ptr` (or nullptr)
- `swap(other)` — swap managed pointers
- `operator bool` — true if non-null
- `operator*`, `operator->` — dereference
- `operator[]` — for `unique_ptr<T[]>` only

**`shared_ptr`:**

- `get()` — raw pointer
- `reset(ptr)` — decrements ref count on current object, takes ownership of `ptr`
- `use_count()` — current strong reference count (mainly for debugging; not reliable for synchronization)
- `unique()` — deprecated in C++17, removed in C++20; was `use_count() == 1`
- `swap(other)` — swap
- `operator bool`, `operator*`, `operator->` — same as unique
- No `release()` — you can't extract exclusive ownership from shared ownership

**`weak_ptr`:**

- `lock()` — attempts to create a `shared_ptr`. Returns a valid `shared_ptr` if the object is still alive, or an empty one if it's been destroyed. This is the safe way to access the object.
- `expired()` — returns `true` if the object has been destroyed (`use_count() == 0`). Don't use this to check-then-lock — that's a TOCTOU race. Just call `lock()` and check the result.
- `use_count()` — strong ref count (for debugging)
- `reset()` — releases the weak reference
- `swap(other)` — swap