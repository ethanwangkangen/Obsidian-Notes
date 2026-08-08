- Can a constructor throw an exception? If so, is it considered constructed? Will its destructor run?
    - Yes. Object is **not** considered constructed if so. Not constructed so destructor is not called too.
    - Member variables that were fully constructed before the throw **do** have destructors called

- Can a destructor throw an exception? If so, is it considered destroyed?
    - **Can** throw but should **never** (see below)

- What happens when a destructor throws an exception while another is being handled?
	- std::terminate is called and program dies
	- **NO mechanism** to handle **two** exceptions at once
	- Destructors are `noexcept` since C++11
    
- What happens if you call an object's destructor manually before it goes out of scope?
	- Destructor runs again when out of scope -> **undefined behaviour**
    
- When should the user manually call an object's destructor?
	- When **`placement new`** has been used to construct an object into pre-allocated storage
    
- In what order are member variables constructed and destroyed?
	- Constructed in order of **declaration**, destroyed in **reverse**
    
- What is placement new (or the C++20 alternative, construct_at), and when is it useful?
    
    - Which types can be used as the underlying buffer for pacement new?
        
- What is the strict aliasing rule?
	- Compilers allowed to assume that pointers to **unrelated types** **do not alias (point to same memory)
		- Optimiser can generate better code, can cache values in registers without worrying that a write through one pointer invalidaets read through another
	- Exceptions:
		- `char*`, `unsigned char*`, `std::byte*`
		- Can alias any type, so they work as placement new buffers
	- eg.
```cpp
int x = 42;
float* f = reinterpret_cast<float*>(&x);
*f = 3.14f;   // undefined behavior — int and float are unrelated types
```

- Explain copy-elision, making reference to RVO and NRVO.
	- Copy/move construction skipped entirely, object is **constructed directly in final location**
	- RVO
		- Applies when returning a prvalue (temporary)
		- In C++ 17 this is **manatory**
	- NRVO
		- Applies when **returning** a **named** local variable
		- Not mandatory
		- If does not fire, the return falls back to a **move** (not copy!)
```cpp
// RVO
Widget makeWidget() {
    return Widget(42);   // mandatory elision since C++17
}
Widget w = makeWidget(); // no copy, no move — one construction total

// NRVO
Widget makeWidget() {
    Widget w(42);
    return w;   // NRVO may elide; if not, w is moved
}
```
    
- What are the five special member functions?
    - Move constructor, move assignment, copy constructor, copy assignment, constructor, destructor
    - What are the following rules as it relates to special member functions:
        - Rule of 3
	        - If you define **any** of destructor, copy constructor or copy assignment, you should define **all** three (managing a resource)
        - Rule of 5
            - Extends to **move** constructor and assignment
	    - When you define special-member function X, what does the compiler no longer implicitly generate for you?
                
        - Rule of 0
	        - If a class does not directly manage a resource, it should **not** have any of the special functions above
	        - Instead, should delegate resource management to specialised tpyes that already handle their own lifecycle like containers (`std::vector`) or smart pointers
            
- What are the four categories of storage duration?
	- Automatic - local variables, scoped lifetime
	- Static - globals, static locals/members, program lifetime
	- Thread - `thread_local`, one per thread, thread lifetime
	- Dynamic - `new/delete`, manual lifetime
        - How would you **prevent** a class from being allocated on the **heap**?
	        - Delete `operator new`
		        - `void* operator new(size_t) = delete;
		        - `void* operator new[](size_t) = delete;`            
        - How would you **prevent** a class from being allocated on the stack**?**
	        - Make the **destructor private** , provide a factory returning a pointer/`shared_ptr`
		        - Compiler can't place object on the stack if it can't call the destructor at scope exit