- Why C is unsafe
	- Say i have a pointer i pass into a function
		- Don't know whether the function will modify the memory at that pointer address
	- Examples
		- forget to free(), leak memory
		- free() to early, get use-after-free bugs
		- free() twice, corrupt the heap (double free)

eg.  Use-after-free
```c
int *p = malloc(sizeof(int));
free(p);

int* q = malloc(sizeof(int)); // Since the memory used by p was freed, likely to reuse it now
*q = 32;

printf("%d", *p)); // Outputs 32, q leaked. p still holds same address
```


Rust principles
- Immutability: by default everything immutable
- Ownership: at any time, single owner of the value
- Borrowing: make sure borrowed values have the expected semantics of borrowing

# Ownership

In C, multiple pointers can point to the same memory location
eg. 
```c
char *s1 = malloc(6);
strcpy(s1, "hello");

char *s2 = s1;   // two pointers to same memory
free(s1);
printf("%s\n", s2); // 💥 use-after-free

```

- Both s1 and s2 think they "own" the same memory, so freeing one leaves the other dangling

In Rust:
```c
let s1 = String::from("hello");
let s2 = s1;   // ownership moves
println!("{}", s1); // compile error, avoids dangling pointer
```

What is ownership?
- **Ownership**: one variable is the “sole responsible party” for memory.
- **Borrowing (`&`)**: temporary permission to access it without taking over responsibility.
- **Mutable borrow (`&mut`)**: temporary permission to modify, but only if no one else is reading at the same time.

# Rust principles:
- All variables assumed to be immutable by default, unless explicitly marked `mut`
- Ownership
	- each value has a single owner, when the owner goes out of scope, the value is dropped (freed)
- Borrowing
	- instead of copying/moving ownership, you can "borrow a reference"
	- immutable borrow (&T) allows multiple readers
	- mutable borrow (&mut T) allows single writer, enforced at compile time
- Lifetimes
	- every reference has a lifetime that must not outlive the data it points to

Examples:
```c

fn change (p : &mut i32) { // Need to specify &mut to modify!
	*p = 36;
}


fn main() {
	let p = 42; // cannot modify this since no mut
	p = 21; // ERROR!
	
	let q = p; // COPY not MOVE. q is a new variable.
	// This is because p is a Copy type. No ownership transfer here.
	// Different from the s1 s2 example above, since String is not Copy (it owns heap memory)
	
	change(&mut p); // Need to pass in a mutable reference too
}
```

Rules for references
- Reference is always valid, no null, no dangling (checked via lifetimes)
- *Can have either **many immutable borows (&T)** OR **ONE mutable borrow (&mut T)** but not both. (to prevent race condition)*
- References cannot outlive the value they point to (borrow checker checks lifetimes)

Lecture examples:
![[Pasted image 20250909100230.png]]
- Why this doesnt work?
	- Pass country into function -> object is copied into local variable of function
	- Once function ends, actually the object is no longer in scope, reference destroyed

How does this work?
- Statically tracked ownership
	- Rust assigns each data object an exclusive owner at a given program point
- Rust can track owner of all objects **statically**
	- Local and Globals known at compile-time
	- **Heap objects owned by local/global variables which are on the stack**
		- Allocate on the heap, but the pointer to the heap data is owned by some variable
		- let s = String::from("hi");
			- s is on the stack, owns a pointer to heap data
	- Heap objects can then own other heap variables
	- Transitive chain of all objects start at local/global (statically known) variables