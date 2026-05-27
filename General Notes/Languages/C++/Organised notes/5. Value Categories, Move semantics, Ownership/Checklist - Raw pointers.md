- What is a raw pointer, when is it preferred over smart pointers?
	- When needing a **non-owning** reference to something,
		- eg. **function parameters**
			- If function just needs to read/modify an object but doesnt take control when it dies

- Deleting pointers `delete[]` versus `delete`
	- `delete[]` for array deletion
	- For non-trivial types, `new[]` typically **stores** the **arry size before the returned pointer**
		- `delete[]` reads it to know the count

- What happens if you try to delete stack-allocated variables?
	- UB

- Can we call `free` on a new-ed up pointer, or `delete` on a malloc'ed pointer, why or why not?
    
- Understand when array-decay-to-pointer happens (`arr[]` -> `arr*`).
	- Passing array to **function taking pointer**, assigning to pointer, or using in most expressions
	- Does **not** happen with `sizeof`, `&arr` (gives `int(*)[5]`), or `decltype`, or binding reference to array `int (&ref) [5] = arr`

- Understand the placement of `const` as it relates to a pointer (east-and-west const, `const*` versus `*const`).
	- `const int* p` // pointer to const int
	- `int const* p` // same thing
	- `int* const p` // const pointer to int
	- `const int* const p` // const pointer to const int
	- Just think about if `const` is to the **left** or the **right** of the `*`
		- If left, is pointer to const
		- If right, is const pointer
   
- When should you use `dynamic_cast` versus `static_cast`?
	- `static_cast` performs compile-time checked conversions
		- For downcasting (base->derive), trusts you, no runtime check
	- `dynamic_cast` 
		- Uses RTTI to verify cast at runtime
		- If cast invalid, returns `nullptr` for pointers or throws `std::bad_cast` (for references)
   
- What are the advantages and disadvantages of `dynamic_cast` over `static_cast`?
	- `dynamic_cast`
		- advantages: safety, catches bugs
		- disadvantages: requires RTTI, has runtime cost, and **polymorphic base** requirement


- What is a co-variant return type?
	- Change return type to a derived pointer instead of base pointer
```cpp
struct Base {
    virtual Base* clone() const { return new Base(*this); }
};

struct Derived : Base {
    Derived* clone() const override { return new Derived(*this); }
    // ^ returns Derived* instead of Base* — legal and useful
};
```


- Understand pointers to pointers to pointers (multiple levels of indirection, e.g. `int***`).
```cpp
int x = 42;
int* p = &x;       // points to x
int** pp = &p;      // points to p
int*** ppp = &pp;   // points to pp

***ppp = 100;       // modifies x through three dereferences
```


- Understanding `const char*`, where is it stored?
	- `const char* s = "hello";`
	- `"hello"` is stored in a read-only section of binary 

- What sort of exceptions can be thrown on a call to new?
```cpp
int* p = new int(5);                // throws std::bad_alloc on failure
int* q = new(std::nothrow) int(5);  // returns nullptr on failure
```

- What is the difference between `new` and `placement new`?
	- Regular `new` does two things: allocates memory, then constructs the object. 
	- Placement `new` skips the allocation — you give it memory you've already obtained
		- Don't call `delete` on placement `new`, must destroy manually

```cpp
// Regular new
int* p = new int(42);  // allocates + constructs

// Placement new
char buffer[sizeof(int)];
int* p = new (buffer) int(42);  // constructs in existing memory, no allocatio
```