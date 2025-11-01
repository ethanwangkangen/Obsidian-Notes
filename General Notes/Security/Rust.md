# Syntax

## Variables

`let x: i32 = 5`
- x is name of variable
- i32 is the type

If i want the variable to be mutable, ie. variable can be reassigned
`let mut x: i32 = 5;`
`x = 6; // allowed`


## Reference types
- These are **TYPES**
- Immutable reference
	- `let x = 5;
	- `let r1: &i32 = &x;`
	- `let r2 = &x;`
	- Can have many immutable reference to the same object at the same time.
	- Question: what happens when x is reassigned?
		- Ans: r1 and r2 change what value they see to. 
		- r1 and r2 are essentially references to the region of memory where x lives.
- Mutable reference
	- `let mut x = 5;`
	- `let r: &mut i32 = &mut x;`
	- `*r += 1;`
	- Can only have one immutable reference at a time, and no mutable references at the time.
- Raw pointers
	- Unsafe, so skip this for now
- Smart pointers
	- `Box<T>`
		- Heap allocation, single owner.
		- Memory automatically freed when out of scope
		- `let b: Box<i32> = Box::new(42);`
	- `Rc<T>`
	- `Arc<T>`
	- `RefCell<T>`

Arrays
- `[T; N]` is a stack-allocated array of length N, where each element has type T
	- `let a: [i32; 3] = [0; 3];` 

Functions
```c
fn () -> T {

} 
```

- Functions return the last expression by default, no need return statement
- 


# Thought process
- Eg. when writing function definitions
	- eg. `fn add_user(db: &mut UserDatabase, mut user: UserStruct)`
	- UserDatabase is passed in as a mutable reference, UserStruct passed by value
	- Q1: Does caller want to give up ownership?
		- For UserDatabase, the caller still wants to be able to have ownership of UserDatabase. So we pass in a reference instead.
		- For UserStruct, caller can relinquish ownership, since after this we access User through the database functions. So can pass by value
	- Q2: If reference, do we want to modify the object?
		- If yes, mutable reference &mut T
		- If no, immutable reference &T
	- Q3: If value, do we want to modify this variable? 
		- Note: different from Python, where reassignment and mutability are diff things
		- Here, even if we want to mutate the object that this variable stores, need to put the 'mut' label

The same thought process goes into what the function returns 

- Does the caller need ownership of the object?
	- If yes → return by value.
	- If no → return a reference (&T or &mut T).
- Does the function need to modify the object?
	- If yes → mutable reference (&mut T).
	- If no → immutable reference (&T).
- Do you need to handle the case where it might not exist?
	- If yes → wrap in Option<T> (like NULL in C).





Option and Some
- `Option<T>` -> can either be Some(T) or None
- 2 ways to check this
	- eg. 

```c
let maybe_user : Option<User> = Some( User {id: 1, username : ..})`

// Option A: match
match maybe_user {
	Some(user) => println!("User");
	None() =? prntln!("No user found");
}

// Option B: if let
// if let PATTERN = EXPRESSION
if let Some(user) = mayber_user {
	..
} else {
	..
}
```
