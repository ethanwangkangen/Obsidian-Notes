# Writing classes
- Public vs private split
- Members default initialised? `{}`, `{nullptr}`
- Rule of 0 first — write none of the five unless you own a raw resource
    - Any user constructor kills the implicit default ctor
    - A user destructor kills both implicit moves → every `std::move` silently deep-copies
    - A user move op deletes the copies
- Rule of 5, if you do own one
    - Move ctor / move assign
        - `noexcept` — this is the one that matters
        - Move ctor **initialises** (`std::exchange` in the init list); only assignment may swap
    - Copy ctor / copy assign
        - Acquire the new before releasing the old. If you can't, add the self-check
        - Return `*this`
    - Copy-and-swap `operator=(T other)`
        - Parameter non-const, non-reference
        - `noexcept` iff the copy can't throw (allocates → no; atomic increment → yes)
        - Don't also declare `operator=(T&&)` — ambiguous for rvalues
    - Destructor
        - Every member released, on every path
- Constructor(s)
    - Single-argument → `explicit`?
    - Member init list, in declaration order
    - If it acquires twice, does the first leak when the second throws?
- Conversions out
    - `operator bool` and friends → `explicit`
- Accessors
    - const + non-const pair for anything returning a reference into owned storage
- Missing surface? `get`/`data`, `begin`/`end`, `size`/`empty`, `swap`

# Writing functions
- Signature
    - Return type correct? Does every path return?
    - Parameters — `const`? `&` / `&&`? by value for sinks?
    - Qualifiers — `const`? `noexcept`?
- `noexcept` — mark it only if all three are "no"
    - Allocates?
    - Calls unknown user code (T's ctor, comparator, hash)?
    - Throws by design or calls something that can (`at`, `lock`)?
    - True for all T, or does it need `noexcept(expr)`?
- What invariant does this function uphold? One sentence, written down
- Pointers dereferenced — null-checked?
- Comparisons — direction right? `<` vs `<=`?
- Arithmetic — overflow? unsigned subtraction? whole range of values?
- Old memory released when assigning? 
	- Eg. copy/move assignment operator - remembered to tryRelease()
- Templates
    - `std::forward<Args>(args)...` — never `forward<T>`, never `move`
    - Named rvalue-ref parameter → `std::move` when passing it on
- Conditions
	- Did you drop any conditions given by the question? Or invariants not properly handled?

# State and ordering 
- Degenerate value of every member — list them
    - Include the moved-from state: swap/exchange moves leave the source at your defaults, so `nullptr` and `0` reach every function
- Invariant in one sentence, then check every site that establishes it — they must agree
- Commit point
    - All fallible work first, into fresh storage
    - Commit with operations that cannot fail
    - A counter describing other state updates only once the description is true
- Update rules — fixed points? `c = c * k` can never leave 0
- Two-object functions — point at each member access: mine or theirs?
- `catch` — names what it restores, then rethrows. Otherwise delete it

# Threads
- Refcount / shared counter → `std::atomic`, not a mutex
- Increment relaxed; decrement `acq_rel` (the deleter must see every prior owner's writes)
- Is the object thread-safe, or only the block it points to?

# Final check
- Pick the degenerate value of what the function reads. Hand-execute it.
- Say one tradeoff out loud — why 2× growth, why atomic over mutex. Interviewers grade the narration.