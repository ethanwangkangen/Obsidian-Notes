### Functions

- What is a function signature, and what are its components? Start with a free-function.
	- Method name, parameter types and count. **NOT** return type
- What is the difference between high-level const and low-level const?
	- High level const: **object** itself is const
	- Low level const: what the **pointer/reference** points to is const
- Which one is included in a function signature?
	- **Low level const** is included
		- Think about it, foo(const int&) is clearly diff from foo(int &) 
			- The former can bind to rvalues
- How do the components of a function signature change when we are referring to a **class** method?
	- Additions
		- The **class** the method belongs to
		- **cv**-qualifiers ont he implicit `this` parameter (const, volatile)
			- Determines whether method can be called on const object, and can overload on it
		- **ref**-qualifiers (& and &&)
- What components become part of the function signature when templates are involved (hint: constraints).
	- constraints
```cpp
template <std::integral T>
void process(T val);

template <std::floating_point T>
void process(T val);      
```

### Lambdas
- What is a lambda?
	- Anonymous function
- What are its three components?
	- `[](){}`
- Which component can be excluded if empty?
    - `()`
- If you wanted to capture a move-only type, such as `std::unique_ptr`, how would you do that?
	- init capture
		- Use `std::move()` inside the capture group, and **create** a new variable 
		- eg.
```cpp
auto ptr = std::make_unique<T>(a);
auto lambda = [p = std::move(ptr)] () {
	cout << *p <<"\n";
};
```
        
- Is a lambda const, or non-const by default?
	- const
	- The operator() is const

- Which modifiers can be used between the parameter list and the function body. 
	- `mutable`, `constexpr/consteval` (C++17, 20), `noexcept`, `-> ReturnType`, `requires` (C++20)

- What is an immediately invocable lambda, and when is it useful?
	- Lambda that's called at the point of definition by appending `()`
	- Usefor for **complex intialisation of const variables**
```cpp
const auto val = [&] {
    if (condition) return computeA();
    else return computeB();
}();  // <-- invoked immediatel
```

- What is the difference between capturing `this` pointer and `*this`?
    - `this pointer: capture a pointer by value, pointer to the object
    - `*this`: captures a **copy of the object**
- When might you use `*this` instead of `this`?
	- When the lambda might outlive the object        
- How is a lambda implemented under-the-hood?
	- Compiler generates a unique, unnamed class (closure type) with member variables for each captured variable, operator() containing the body, `const`-qualified by default (unless mutable)
- What is the size of a lambda?
	- Stateless: 1
	- Captures by value:  sum of captured members' sizes plus padding/alignment
	- Captures by reference: each is a pointer typically
- What is a lambda's type?
	- Only compiler knows, unique, unnamed closure type
- Are two equivalent lambdas the same type (`std::is_same_v`)?
	- No  

### std::function

- What is `std::function`?
    
    - What technique does `std::function` make use of to achieve the ability to contain any callable given a generic interface?
        
        - How might you implement a similar type, such as `std::any`?
            
        - How large is `std::function` (based on how it might be implemented)?
            
- What is the difference between `std::function` and `std::lambda`?
    
    - When might you use `std::function` instead of `std::lambda`?