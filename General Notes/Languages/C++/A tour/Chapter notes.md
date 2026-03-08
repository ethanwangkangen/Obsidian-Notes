# Chapter 1
- Scope vs lifetime
	- Scope - eg. global vs local scope
		- Block scope {}: variables only visible inside here
	- Lifetime
		- Runtime concept
		- When does this object obccupy a valid memory address
		- Automatic (static) lifetime, static lifetime, dynamic (heap) lifetime
- `const` vs `constexpr`
	- const
		- 'i promise not to change this'.
		- Can be evaluated at runtime
	- constexpr
		- 'compiler calculate this before i run program'
	- constexpr function
		- If you give it inputs known at compile-time: It runs during compilation, and the result is replaced in your code (effectively becoming a hardcoded value).
		- If you give it inputs only known at runtime: It runs just like a normal function at runtime.
- References vs pointer


# Chapter 2
- Structs
	- Same as class, but all members public
- Class
	- Public, private and protected members
	- Member initialisation list
		- Use this for constructors
		- `Vector(int s): elem{new double[s]}, sz{s} {}`
			- Initialises both the `elem` and `sz` variables
			- More efficient than doing it in the body. This is direct initialisation
			- Necessary in cases, eg.
				- `const` members, cannot change garbage value if not initialised in list
				- `reference` members, must be bound to an object the moment they are creates
				- `no default constructor`
- Union
	- Struct, but all members allocted **same address** so union occupies only as much space as its largest member
	- Only holds avlue for one member at a time
- Enum
	- Enum classes
		- `enum class Color {red, blue, green}`
		- Red is in the scope of Color, so Color:: red is different from another red.
		- Can define operators
			- **Overloading** of default operator method
``` 
Color& operator++ (Color&) t{
	return t=Color::red
}

Color red = ++blue;
```

- Pass by value 
	- In C++, objects passed by value are **copied**
		- This is a shallow copy
		- So if an object holds a pointer to another object, only that pointer to that inner object is copied. No deep copy
	- If don't want to copy, need to pass a reference to the original object
		- `&cat samecat = cat;`
- 
# Chapter 3

# Chapter 4
- this
	- Refers to the **pointer** that holds the object
	- So if in the class `complex`, I define (overload) the += operator like such
		- `complex& operator += (const complex& z) {this->real += z.real; this->im += z.im; return *this;}
			- `*this` dereferences the pointer to return the object itself, so it fits the type of complex& (reference to the object)
			- Take note of difference between `*` and `.`
				- `this` is a pointer, so we need to dereference to get the object. `->` is the same as `(*this).`
				- `z` is the object reference, so can directly access member with `.`
- Can seperate class function definitions from declaration
	- Just declare in class body and define outside
- Destructors
	- `~Vector() {delete[] elem;}`
		- elem was inis\tialised with `new double[s]`, so when the Vector gets freed, we need to free this too with `delete[]`
	- Automatically called when variable goes out of scope
	- Rule of 3
		- If we need to write a custom **destructor**, we almost certainly need a **custom copy constructor** and **copy assignment operator**
- RAII
	- Acquire resources in a constructor and release them in a destructor
	- Try to avoid naked new and delete line
- Abstract classes
	- `virtual double& operator[](int)=0;` -> pure virtual function
	- Virtual function means **may be redefined later in a class dreved from this one**
	- `=0` means **pure virtual** -> **MUST** bedefined by the class derived from Container
		- A class with a pure virtual function becomes an abstract class
			- Same like Java
	- Can implement functions that make use of the abstract class interface without knowing implementation
		- eg. `c[i]` used in a function that takes in `Container& c`
				- This is function resolution. The class that inherits from Container must define the `operator[]()` method, which will be put into the virtaul funciton table (vtbl)
	- Commonly have no constructor, but have destructor
	- Inheritance and overriding
		- `override` keyword - eg. `void draw() const override;` 
			- Not necessary, but can use to be explicit
		- Same as in Java
		- Destructor of subclass overrides base class destructor
		- Calling destructor of a class (eg. `vector_container`) implicitly calls destructor of a member (eg. `vector`)
			- But this only applies if vector is a MEMBER of vector_container
			- If vector_container holds a **pointer**, those objects need to be freed
	- Typing rules are **not the same as Java**
		- Cannot do `Shape circle1 = new Circle()`
			- Get slicing. It only retains the elements of `Shape`
- 
