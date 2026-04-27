### Virtual
- What is the **virtual** keyword?
	- It **marks** a method so the call is **dispatched** based on **runtime type** of the object, not the compile time type of pointer/reference

- What is the **size** implication of adding even one virtual method to a class?
	- **8 bytes to store pointer** to the vtable
	- However, adding more virtual methods doesn't increase size of object instance, only the vtable grows

- What are its implications in terms of **runtime** costs?
	- **2 extra** indirections: Need to access load the vtable from the vptr, then retrieve the function
	- **Branch** **prediction**: CPU can't target easily, pipeline stalls
	- **No inlining**
		- Note: virtual functions still can be made inline, 

- What is the **vtable**?
	- Static, per-class (not per-instance!) array of function pointers, one entry per virtual method

- What is a **vtable pointer**?
	- Pointer that is a data member of each instance of a class that has at least one virtual method
	
- What are its implications in terms of the object's size and type?
	- Every instance carries one extra pointer
	- It is essentially the **runtime type identity** of the object
		- `dynamic_cast` and `typeid` consult this

- What happens if you call a virtual method in a parent-object's constructor, why?
	- Doesn't work the way you expect it to, because the parent is constructed first, the **derived vtable** **does not exist yet**
	- End's up calling the **parent/base function**

- When do you need to mark an object's destructor as virtual, and why 
	- Whenever a class is **designed to be used polymorphically**
		- ie. **Base class** destructor should be **virtual**
	- So that calling `delete` on the **base pointer** calls the **derived** virtual destructor

- How do you **override** a **destructor**, and how does its behavior differ from overriding a method?
	- Override a destructor by declaring a **base class destructor virtual**
	- Destructors **chain automatically**, the **base** version is **automatically called as well**

- Does virtual apply to the static or **dynamic** type of the pointed at object?
	- Dynamic

 - What are pitfalls of using **default** parameters with virtual methods?
	- **Default** parameters are **bound at compile time**, the function call itself is replaced with the default parameter using the static type

- Is **override** mandatory, why is it useful to use?  
	- Not mandatory, has no actual function
	- Useful to verify that you are actually overriding a virtual method

- What is the **diamond inheritance** problem?
	- Two classes inherit from the same base, a fourth class inherits from them both

```cpp
Animal 
/    \ 
Dog Cat
 \   / 
 DogCat ← inherits Animal twice!
```


 - How do you solve the diamond inheritance problem?
	 - **Virtual inheritance**
	 - Make Dog and Cat virtual (in the above example)
   
- What if you didn't want to use virtual inheritance to solve the diamond inheritance problem?
	- Composition
  
- How do you prevent a virtual method from being overriden?
	- **Final** keyword

- How do virtual methods interact with access modifiers (public, private, protected)? For instance, can you override a private virtual method from a child class?    


- How do you shift the cost of dynamic polymorphism to compile-time?

- What are the implications?