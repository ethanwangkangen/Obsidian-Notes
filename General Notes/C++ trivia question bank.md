# C++ Interview Question Set — Trip Edition

Write your answers by hand or in a notes file. Number them to match. No looking things up until you've committed to an answer — mark any answer you weren't sure of with a `?` so we can weight the review. We'll check everything at the end of the trip.

Some questions deliberately re-test things you've gotten wrong before. They are not marked.

---
**Note: the "answers" here are not necessarily the correct answers. This is my attempt at each question before i correct the mistakes**
## 1. Object Lifetime & Construction (10)

Walk through everything that happens, in order, when `Derived d;` is executed, where `Derived` inherits from `Base` and both have members with constructors.
- Everything in the base is constructed first before the derived is constructed. Members are initialised first before constructor runs. So, in base first. Members are created in the order they are declared in. Then the constructor of base runs. Then the same for derived.
- ✅ Things to add
	- Virtual bases come first before any non-virtual base and are constructed by the most-derived class, not by whichever intermediate class lists them
	- The vptr is repointed at each stage.
		- Remember that a class instance, no matter how derived, only holds one vpointer


What is the order of destruction relative to construction? Why is it that order?
- Destruction happens in the reverse order to construction. Why that order, possibly because it makes sense in terms of memory-layout? Eg. similar to how a stack grows up and down.
- ⚠️ Half credit - things to add
	- Nothing to do with stack layout. Has to do with **dependency order**. A member declared later may hold reference to one declared earlier

What happens if a constructor throws halfway through? Which destructors run, and which don't?
- The stack unwinds, only the destructor for the sub-classes whose constructors have fully run will be called. An object is only considered 'created' when the constructor has run fully, so if it has not, its destructor is not called.
- ✅ Things to add
	- Where it throws changes what leaks
	- Throws in member init list -> only subobjects completed so far are destroyed
	- Throws in body -> all bases and members fully constructed, so all of their destructors run, but class's own destructor doesn't.

What is a delegating constructor? If a delegating constructor's body throws, is the object considered constructed?
- A constructor that calls another constructor. Unsure about if it's considered constructed. 
- ⚠️ Half credit - things to add
	- Yes, since object is fully constructed. Because the target constructor has already finished, so object is created.


Explain the difference between default-initialization, value-initialization, and zero-initialization.
- Not to sure about this.
- **X** -missing or wrong
	- Default init - `T obj;`, `new T;`. Indeterminate value. 
	- Value init - `T obj{}`, `T()`, `new T()` - If class has user-provided or deleted default constructor, it runs (no zeroing). Class without user provided default constructor -> object is **zero-initialised first**, then the constructor runs. 
	- Zero-init - applies to all static/thread-local storage before any other initialisation, and is the first step of value-init (see above)

1. What does `int x;` give you at namespace scope vs inside a function?
- Namespace scope: zero initialised and put in the BSS. Inside function, indeterminate, posisbly junk.
- ✅ Things to add
	- It's not just 'junk', its UB.

1. What does `T obj();` actually declare? Why is this a classic trap?
- This is the most vexing parse. It actually declares a function called obj that returns T, and is a classic trap as one might be trying to default initialise an obj of type T.
- ✅ Things to add
	- `Widget w(Timer())` -> this declares a function taking a pointer to a function returning timer

1. When is it legal to call a virtual function inside a constructor, and what will it dispatch to? Why?
- Calling a virtual function is legal if the function exists in a base class (of higher hierarchy than this). It will then dispatch to the base class version. This is because the current class has not been fully created yet so it won't dispatch to this version of the function.
- **X** -missing or wrong
	- **Always legal**
	- Dispatches to most-derived override among classes constructed so far.
	- Completely wrong about dispatching to base class instead. It will dispatch to the CURRENT class since the ones further down in the hierarchy aren't created yet.
	- **MISCONCEPTION** WRONGGG "during C's construction running, C isnt created so C::f() can't be called"
		- Memory is allocated before constructors run. Constructors just do the initialising.
		- During c's creation, everything in C has already been initialised
		- To be precide, the vptr is advanced to C's vtable before C's member init list even runs.
	- Virtual calls from member initialiser list are a trap
		- Members have not been fully initialised yet, but vptr has already advanced. So if `C::f()` uses C's member var and is called within the member init list, will get junk.

1. What is placement new? Who is responsible for calling the destructor, and how do you call it?
- Used to create an object within some memory that already has been allocated with std::alloctor, or within some sort of buffer. Syntax is `new (address) T{..}` and `p->~T();`
- Destructor has to be called by the programmer.
- ✅ Things to add
	- Must `#include <new>`, storage must be aligned correctly and large enough, must never call `delete` on that pointer.

1. What is `std::launder` for, at a high level? When does the problem it solves arise?
- Unsure
- **KIV: COME BACK TO THIS**

2. Explain static initialization order fiasco. Give one standard mitigation.
- C++ gives no guarantee on the order in which static/global namespace variables are created. This can cause issues where there's dependency based on the order. 
- Mitigation: use static variables inside functions instead, and instantiate them by calling the functino (they are instantiated at the first function call, to my knowledge).
- ✅ Things to add
	- The fiasco is **betwen translation units**. Within one TU, order guaranteed.
	- Specifically **dynamic** initialisation that's unordered
	- `constinit` is the other tool
## 2. Copy & Move Semantics (12)

11. What are the five special member functions? For each, state the conditions under which the compiler implicitly generates it.
12. Writing a user-defined destructor suppresses implicit generation of which special members? What is the practical consequence?
13. What does `std::move` actually do? Does it move anything?
14. After `auto b = std::move(a);` what state is `a` guaranteed to be in (a) for standard library types, (b) for your own types?
15. A class has a `const std::string name;` member and otherwise movable members. Can objects of this class be move-constructed? Move-assigned? Explain exactly what happens to each member in each case.
16. Why should a move constructor be marked `noexcept`? Name the specific standard-library mechanism that changes behavior based on it.
17. What is copy elision? Which cases are _mandatory_ since C++17?
18. Explain NRVO. Why does `return std::move(local);` usually make things worse?
19. When does returning by value beat returning by output parameter, and when doesn't it?
20. Implement the copy-and-swap idiom for `operator=`. What guarantees does it buy you, and what does it cost?
21. What is the difference between `T&&` in `void f(T&& x)` where `T` is a template parameter, versus where `T` is a concrete type?
22. Write the signature of a move constructor and move assignment operator for `class Buffer { char* data_; size_t size_; };` and implement both correctly, including self-assignment handling.

## 3. Value Categories, References & Forwarding (10)

23. Define lvalue, prvalue, and xvalue. Give one expression that is each.
24. `std::string s = "hi"; std::string&& r = std::move(s);` — is `r` an lvalue or rvalue when used in a subsequent expression? Why does this matter for forwarding?
25. Does binding a temporary to `const T&` extend its lifetime? Does binding a temporary to `T&&` extend its lifetime? Does _returning_ a `T&&` bound to a local extend anything?
26. What are the reference collapsing rules? Show all four combinations.
27. What is `std::forward` and why can't you just use `std::move` in its place inside a forwarding function?
28. Write a function template `wrapper` that perfectly forwards any argument to a function `g`. Explain each token.
29. Why is `auto&&` sometimes called a "universal reference" in range-for loops? When would you use it over `const auto&`?
30. What does `decltype(x)` give you vs `decltype((x))` for a local `int x;`?
31. What's the danger in `const std::string& s = cond ? str1 : "literal";`?
32. Given `void f(int&); void f(int&&);` — which overload does each of these call: `f(5)`, `f(x)`, `f(std::move(x))`, `f(x + 1)`?

## 4. const, constexpr, and Compile Time (8)

33. What's the difference between `const`, `constexpr`, and `consteval` on a function?
34. Can a `constexpr` function be called at runtime? Can it contain a loop? Throw?
35. What does `mutable` do, and give a legitimate use case that isn't "cheating."
36. What does `const` after a member function signature mean, precisely, in terms of the `this` pointer? What about `&&` after a member function?
37. Explain physical const vs logical const. How does `const_cast` interact with objects that were _defined_ const?
38. What is `if constexpr` and how does it differ from a regular `if` inside a template?
39. What's the difference between `static constexpr` and `inline constexpr` at namespace scope for avoiding ODR problems in headers?
40. Why can `constexpr` functions still exhibit UB, and what happens if UB is reached during constant evaluation?

## 5. Templates & Metaprogramming (12)

41. What is the difference between a function template and a template function instantiation? When does instantiation happen?
42. Explain SFINAE in one paragraph, and show a minimal `enable_if` example gating a function on `is_integral`.
43. What do C++20 concepts replace, and what advantages do they have over SFINAE beyond nicer errors?
44. What is CRTP? Write the skeleton and explain how the base can call derived methods without virtual dispatch. What is the cost/benefit vs virtual functions?
45. Why must template definitions typically live in headers? What is explicit instantiation and how does it change this?
46. What is two-phase lookup? Why do you sometimes need `this->` or `typename` inside templates?
47. When is `typename` required before a dependent name? When is `template` required?
48. What is template argument deduction for class templates (CTAD)? Give an example with `std::pair` or a deduction guide.
49. What's the difference between full specialization and partial specialization? Which is allowed for function templates?
50. Implement a compile-time `factorial` two ways: recursive template and `constexpr` function. Which would you use today and why?
51. What is a variadic template? Write `sum(args...)` using a fold expression.
52. What is `std::void_t` useful for? Sketch a detection idiom checking whether `T` has a member `size()`.

## 6. Inheritance & Virtual Dispatch (10)

53. Describe the memory layout of an object of a class with virtual functions. Where does the vptr live? For single inheritance with a 3-level hierarchy, how many vptrs does one object contain?
54. Walk through what happens at the machine level when you call `p->f()` where `f` is virtual. Roughly how many memory loads is that, and why does it hurt in hot paths?
55. When is a virtual destructor required? What exactly goes wrong without one?
56. What does `final` on a class or virtual function enable the compiler to do?
57. What is devirtualization and when can the compiler perform it?
58. Explain virtual inheritance and the diamond problem. What changes about object layout and construction order?
59. Can virtual functions be inlined? Under what circumstances?
60. What is object slicing? Show a two-line example that silently slices.
61. `override` vs `virtual` on a derived class method — what does `override` actually check? Give an example bug it catches.
62. Why is calling a pure virtual function from a base constructor a runtime error rather than a compile error?

## 7. STL Containers & Internals (12)

63. Describe the growth strategy of `std::vector`. What is the amortized complexity of `push_back` and why? What does `reserve` change vs `resize`?
64. When `vector` reallocates, does it move or copy its elements? What determines which?
65. Compare `std::map` and `std::unordered_map`: underlying structure, complexity, iteration order, memory layout, and cache behavior. Which would you default to in a latency-sensitive path and why might the answer be "neither"?
66. How does `std::unordered_map` handle collisions in typical implementations? What is `load_factor` and what happens on rehash?
67. Why is `std::deque` not just "vector with fast front insertion"? Sketch its actual memory structure.
68. Why is `vector<bool>` notorious? What breaks?
69. What does `std::span` give you and what does it _not_ own? What's the danger?
70. `emplace_back` vs `push_back` — what's the actual difference, and give a case where `emplace_back` is a trap.
71. What is small string optimization? Roughly how many chars fit inline in typical implementations, and what operation semantics change because of it (hint: think about moves)?
72. How would you implement `std::sort`? Name the algorithm typically used and why quicksort alone isn't acceptable for the standard.
73. What are the complexity guarantees of `std::nth_element` and when would you reach for it over `sort`?
74. Compare `std::priority_queue`, a sorted `vector`, and `std::multiset` for maintaining a top-k structure with frequent inserts. Trade-offs?

## 8. Iterators & Invalidation (12)

75. State the iterator invalidation rules for `std::vector` for: `push_back`, `insert` in the middle, `erase`. Distinguish invalidation of iterators _before_ vs _at/after_ the modification point, and the with/without-reallocation cases.
76. Same question for `std::deque` — front insertion, back insertion, middle insertion. (Careful: references and iterators behave differently here.)
77. Same question for `std::map` / `std::set` — insert and erase.
78. Same question for `std::unordered_map` — insert (no rehash), insert (rehash triggered), erase. What stays valid across a rehash that surprises people?
79. What does `erase` return, and write the canonical loop for erasing all even numbers from a `vector<int>` (a) manually with iterators, (b) with the erase-remove idiom, (c) with C++20 `std::erase_if`.
80. Why is `v.erase(std::remove(...))` (missing the second iterator argument, or missing `v.end()`) a subtle bug family? What does `std::remove` actually do to the container?
81. This code compiles. What is wrong with it?
    
    ```cpp
    for (auto it = v.begin(); it != v.end(); ++it)    if (*it == 0) v.erase(it);
    ```
    
82. Do references into a `std::vector` survive `push_back`? Do references into a `std::map` survive `insert`? Do _references_ into `std::unordered_map` survive rehash?
83. `auto& x = v[3]; v.reserve(1000); use(x);` — is this okay? What if `reserve` was `shrink_to_fit`?
84. What iterator category does each of these provide: `vector`, `list`, `forward_list`, `deque`, `map`, `unordered_map`, an input stream iterator? Why does `std::sort` refuse some of them?
85. What is a dangling iterator vs a singular iterator? Is comparing an invalidated iterator to `end()` defined behavior?
86. Design question: you hold iterators into a `std::vector` order book while another code path inserts orders. Enumerate your options for making this safe and their costs (stable containers, indices, generation counters, etc.).

## 9. Memory, Allocators & Smart Pointers (12)

87. What does `new T[10]` do beyond allocating memory? Why must it be paired with `delete[]` and what actually goes wrong with plain `delete`?
88. Explain the layout and control-block mechanics of `std::shared_ptr`. What does `make_shared` change about allocation, and what is the weak-pointer-keeps-memory-alive caveat?
89. How much does a `shared_ptr` copy cost in a multithreaded program, mechanically? Why is passing `shared_ptr` by value in hot paths a smell?
90. What is `std::weak_ptr` for? Walk through `lock()` — can it race with the last `shared_ptr` dying?
91. `unique_ptr<T>` vs raw pointer: what is the size and runtime overhead? What about `unique_ptr<T, CustomDeleter>` where the deleter is a lambda vs a function pointer?
92. What does `std::enable_shared_from_this` solve, and what goes wrong if you call `shared_from_this()` in a constructor?
93. Explain what an allocator is in the STL. What is `std::pmr::monotonic_buffer_resource` and when is it a big win?
94. Sketch how a fixed-size pool allocator works. Why do trading systems use them instead of the general-purpose heap?
95. What is memory alignment? What does `alignas(64)` do and why 64 specifically?
96. What's the difference between `malloc`+placement-new and plain `new`? When would you separate allocation from construction deliberately?
97. What is a memory arena? Why does freeing "everything at once" change fragmentation and latency characteristics?
98. Double-delete, use-after-free, and leak: for each, name one tool or technique you'd use to catch it, and one coding pattern that structurally prevents it.

## 10. Integer Semantics, Casts & UB (10)

99. `for (int i = 0; i < v.size(); ++i)` — walk through exactly what the comparison does at the type level, when it misbehaves, and your standing fix.
100. What happens on signed integer overflow? Unsigned overflow? Why does the compiler exploit one of these for optimization — give a concrete optimization it enables.
101. `int a = -1; unsigned b = 1; if (a < b)` — what does this evaluate to and why?
102. What are the four named casts, and one legitimate use of each? Which of them can fail at runtime and how do you observe the failure?
103. What does `reinterpret_cast<float*>(&myInt)` followed by a dereference violate? What are the sanctioned ways to type-pun in modern C++?
104. What is UB vs unspecified vs implementation-defined behavior? Give one example of each.
105. `i = i++ + 1;` — status in C++17 and later? What changed about evaluation order in C++17 generally?
106. Why is `std::size_t` unsigned, why do many style guides consider that a mistake, and what is `ssize()`?
107. What does the strict aliasing rule allow the compiler to assume? Show a snippet that breaks it.
108. Narrowing: `int x = 3.7;`, `int y{3.7};` — what happens in each? Where else does brace-init save you?

## 11. Concurrency & the Memory Model (16)

109. What is a data race, precisely, in the C++ standard's terms? What is the consequence of one existing?
110. Explain the difference between `std::mutex`, `std::recursive_mutex`, `std::shared_mutex`, and when you would (rarely) justify each of the latter two.
111. `lock_guard` vs `unique_lock` vs `scoped_lock` — capabilities and costs.
112. What is a deadlock? Give the four Coffman conditions and the two standard C++ tools for lock-ordering problems.
113. What does `std::condition_variable::wait(lock, pred)` expand to, and why is the predicate loop non-negotiable? What is a spurious wakeup? What is the lost-wakeup bug?
114. Explain `memory_order_relaxed`, `acquire`, `release`, `acq_rel`, and `seq_cst`. For each, one sentence on what it guarantees.
115. Write the classic release/acquire message-passing pattern: thread A writes data then a flag, thread B reads the flag then the data. Annotate which memory orders are needed and why relaxed on the flag breaks it.
116. What does "happens-before" mean and how do release/acquire pairs establish it?
117. Is `volatile` a threading tool in C++? What is it actually for?
118. Write a spinlock using `std::atomic_flag` or `atomic<bool>`, with correct memory orders, and add the two standard refinements (test-and-test-and-set, and a pause/yield).
119. Explain compare_exchange_weak vs compare_exchange_strong. Why does the CAS loop idiom tolerate spurious failure, and what does `expected` get set to on failure?
120. What is the ABA problem? Give a concrete scenario in a lock-free stack and two mitigations.
121. Sketch the design of an SPSC ring buffer: what does each thread own, which indices need atomics, what memory orders, and where does false sharing bite? What is the +1 capacity (or wrap-flag) trick for distinguishing full from empty?
122. What is false sharing? How do you detect it and how do you fix it in code?
123. `thread_local` — what is it, when initialized/destroyed, and one HFT-relevant use.
124. What is a lock-free vs wait-free algorithm? Is a CAS retry loop wait-free?

## 12. Performance, Cache & Low-Latency (12)

125. From memory, no notes: approximate latency in cycles for L1 hit, L2 hit, L3 hit, main memory. Add rough numbers for a branch mispredict and a mutex lock/unlock (uncontended).
126. What is a cache line? Walk through why iterating a `vector<T>` is fast and iterating a `list<T>` of the same data is slow, in terms of lines and prefetching.
127. Row-major 2D array: why is looping `[i][j]` vs `[j][i]` a massive difference? Estimate the difference for a 10,000×10,000 `int` matrix.
128. What is branch prediction? Why does sorting an array before a branchy loop over it famously speed the loop up? What are `[[likely]]`/`[[unlikely]]` and when are they justified?
129. What is the difference between latency and throughput of an instruction? Why do dependency chains matter more than instruction count in hot loops?
130. What does `inline` mean to the modern compiler vs to the linker? What actually controls inlining, and what is the cost of a function call that _isn't_ inlined?
131. What is LTO? What is PGO? One sentence each on what they enable.
132. Why are exceptions often banned in hot paths? What is the actual cost model of a modern "zero-cost" exception implementation — what's zero and what's not?
133. Why might `std::function` be banned in a hot path? What are the alternatives (at least two) and their trade-offs?
134. What is the cost of a system call, roughly, and name three techniques trading systems use to avoid syscalls on the fast path (think: networking, timing, memory).
135. What is NUMA and why does thread/memory pinning matter? What do `taskset`/`isolcpus`-style techniques accomplish conceptually?
136. You measure p50 = 800ns and p99.9 = 40µs for the same code path. List at least five plausible causes of the tail and how you'd investigate each.

## 13. Systems & OS Adjacent (8)

137. What happens, at a syscall/kernel level, between typing `./a.out` and `main` running? (High level: fork/exec, loader, dynamic linking, relocations.)
138. Static vs dynamic linking: costs at build, load, and run time. What is PLT/GOT overhead conceptually?
139. What is virtual memory? What is a page fault, and what's the difference between a minor and major fault? Why does a trading process pre-fault and lock its memory?
140. TCP vs UDP in five sentences, then: why do market data feeds use UDP multicast, and what does that force the receiver to handle? (You've built around this — be precise about gap detection and recovery.)
141. What does `epoll` solve over `select`/`poll`? What is edge-triggered vs level-triggered? Where does busy-polling fit instead?
142. What is kernel bypass networking conceptually (DPDK/Onload-style)? What does it eliminate from the receive path?
143. What is the difference between a process and a thread at the kernel level? What does a context switch cost and what gets flushed?
144. Explain what happens on `std::cout << x` that makes iostreams slow, and three reasons `printf`-style or hand-rolled formatting wins in hot paths.

## 14. Code Reading — What Does It Do / What's Wrong (12)

For each: state the output, or identify the bug/UB, and explain precisely.

145. ```cpp
    std::vector<int> v{1, 2, 3};
    for (auto x : {v.begin(), v.end()}) {}   // does this compile? what is x?
    ```
    
146. ```cpp
    struct S { int x = 1; S() : x(2) { x = 3; } };
    S s;  // what is s.x, and which of the three writes happened?
    ```
    
147. ```cpp
    auto f = [=]() { return counter_++; };   // inside a member function
    // what is actually captured, and what's the lifetime trap?
    ```
    
148. ```cpp
    const int& r = std::max(1, 2);
    int use = r;   // ok?
    std::string&& s = std::string("hi") + "!";
    s += "?";      // ok?
    ```
    
149. ```cpp
    std::map<std::string, int> m;
    if (m["key"] > 0) { /* ... */ }   // what did this do to m?
    ```
    
150. ```cpp
    std::vector<std::string> v;
    v.reserve(2);
    auto& first = v.emplace_back("a");
    v.emplace_back("b");
    v.emplace_back("c");
    std::cout << first;   // ok?
    ```
    
151. ```cpp
    unsigned n = v.size();
    for (unsigned i = 0; i <= n - 1; ++i) { /* ... */ }   // when does this explode?
    ```
    
152. ```cpp
    struct B { virtual void f() { std::cout << "B"; } };
    struct D : B { void f() override { std::cout << "D"; } };
    void call(B b) { b.f(); }
    D d; call(d);   // prints?
    ```
    
153. ```cpp
    std::shared_ptr<int> p(new int(5));
    std::shared_ptr<int> q(p.get());   // what happens eventually?
    ```
    
154. ```cpp
    std::atomic<bool> ready{false};
    int data = 0;
    // T1:
    data = 42;
    ready.store(true, std::memory_order_relaxed);
    // T2:
    while (!ready.load(std::memory_order_relaxed)) {}
    std::cout << data;   // guaranteed 42?
    ```
    
155. ```cpp
    std::string_view sv = std::string("temporary");
    std::cout << sv;   // ok?
    ```
    
156. ```cpp
    struct P { std::unique_ptr<int> u; };
    P a{std::make_unique<int>(1)};
    P b = a;   // compiles? if not, what one-token change makes it compile, and is that change wise?
    ```
    

---

## Scoring plan for the review

When we check: full credit needs the _mechanism_, not just the conclusion (e.g. "UB" alone is half credit — say which rule is violated). Anything you marked `?`, anything wrong, and anything right-but-fuzzy goes to the ledger with topic, your answer, and the correction.

156 questions. Safe travels.