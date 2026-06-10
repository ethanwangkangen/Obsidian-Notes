- What are the four _unique_ types of casts in C++?
	- `static_cast`
		- Compiler can verify at compile time: numeric promotions/narrowing, up/downcast in known hierarchy (no runtime check), void* to typed pointer
	- `dynamic_cast`
		- **Runtime check** for polymorphic class hierarchies
			- Must have at least one virtual function
		- On failure
			- Pointers: **nullptr** returned
			- References: **throws std::bad_cast**
		- Inspects vtable/RTTI
		- Cost: runtime lookup
	- `reinterpret_cast`
		- Reinterprets the bit pattern of a pointer/reference as a diferent type
	- `const cast`
		- Adds/removes `const` or `volatile`

- What is _type-punning_?
	- Trying to convert bit representation of one object to another
	- Recommended to use `std::bit_cast<>` (C++20), which uses `memcpy` to copy source bits into destination buffer
```cpp
// Works, but UB
uint32_t second = *reinterpret_cast<uint32_t*>(&pi);

// Recommended way (C++20)
const uint32_t pii = std::bit_cast<uint32_t>(pi);
```


- When is reinterpret_cast useful?      
    - What are the advantages of C++20's std::bit_cast utility over reinterpret_cast?
        - When is it safe to access a data member of a reinterpret_cast'ed pointer?
    - `std::memcpy` versus `reinterpret_cast`.
        
- What order are these casts attempted in a C-style cast?
    
- When is const-cast useful, what are its dangers?
	- When calling a legacy C APi that takes in a non-const pointer but doesnt actually modify data
	- Implementing the const/non-const member function deduplication pattern — the non-const version casts away const from the result of calling the const version (**Scott Meyers' idiom**), avoiding code duplication.
	- Danger:
		- If **underlying object** originally declared **const**, modifying it through const_cast'd pointer/ reference is undefined behvaiour
```cpp
const int a = 10;
const int *b = &a;
 
int c = 20;
const int *d = &c;
 
int *bad = const_cast< int * >( b );
*bad = 12;  // Invalid; undefined behavior!
 
int *good = const_cast< int * >( d );
*good = 42; // Valid!
```

- What is `std::move` (r-value cast), and how is it implemented?
    - Casts to rvalue reference
    - Strips reference qualifiers using `remove_reference_t`, then casts to `T&&`
```cpp
template <typename T>
constexpr std::remove_reference_t<T>&& move(T&& t) noexcept {
    return static_cast<std::remove_reference_t<T>&&>(t);
}
```

   - What happens when you try to `std::move` a const object?
	   - `const T&&` produced. Copy constructor is called since `const T&` parameter can bind to rvalues. So aa copy is made

- What implications does adding `const` to a class data member have?
	- Compiler **cannot generate** move assignment operator 
		- Moving requires overwriting the target, but can't assign to const member
	- Implicitly generated copy assignment also deleted
	- Class if effectively **non assignable** normally

- What are the two _primary_ value categories (l-value, r-value)?
    - What are their differences?
	    - lvalue: has an address you can take
	    - rvalue: temporary/value that's about to expire
    - If we were to expand our definition of value categories, what other three _extended_ value categories are there?
	    - xvalue (expiring value)
	    - gl value (lvalue and xvalue)
	    - prvalue (pure rvalue)
        
        - What is the difference between pr-values and r-values?
	        - pr-value has no identity at all
	        - rvalue encompasses xvalues as well. xvalues do have identity but has been marked as expiring
            
        - What guarantees, if any, are there related to x-values. For instance, can they be reused?
	        - x-values guarantees that you can safely destroy it and call operations with no preconditions on it
		        - ie **destructor can run**, can **reassign** it
	        - Cannot assume anything about its values
		        - Valid but unspecified state.

- What is an implicit cast, when might it happen?
    - Numeric promotions/conversions, boolean contexts, pointer conversions, user-defined conversions
    - Most **dangerous** : signed to unsigned
	    - `int(-1) < unsigned(0)` -> -1 converts to a very large unsigned value, so **false**
    - How do you safely compare signed and unsigned integers?
	    - In C++20: use **std::cmp_less**, **std::cmp_greater**, etc.

- How do you prevent implicit casts for user-defined types?
	- **explicit** keyword