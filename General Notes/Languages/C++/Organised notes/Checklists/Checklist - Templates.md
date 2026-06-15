- When are templates instantiated?
    - Compile time
- What are the implications on cache and binary size?
	- Binary size: more instantiations, larger size
	- Cache: Each instantiation is a separate function body at different address in memory -> more potential cache misses

### Function Templates
- How do you prevent a function template from being instantiated for a given type `T`?
	- `foo<int> = delete;`
- What is partial template specialization?
```cpp
template <typename T, int N>
class A {
}

template<T, >
A <T, 0> {.. };
```    
- Can functions be partially specialized?
	- No.

- What is CTAD, when was it introduced?
	- Class Template Argument Deduction    
- What are template deduction guides and when are they needed?
	- To give hints to comipler when constructor parameters don't map to template parameters
```cpp
template <typename T>
struct Wrapper {
    T value;
    Wrapper(T v) : value(v) { }
};

// Problem: Wrapper w{"hello"} deduces Wrapper<const char*>
// Guide overrides the deduction:
Wrapper(const char*) -> Wrapper<std::string>;
```
        
- What are the three types of template parameters?
    - Type, Non-type, template
- When would you use non-type template parameters?
	- When need to pass compile-time argument to function
- When would you use template template type parameters?
```cpp
template <typename T, template <typename, typename> typename Container = std::vector>
class Stack {
    Container<T, std::allocator<T>> data;
};

Stack<int> s;               // uses vector<int>
Stack<int, std::deque> s2;  // uses deque<int>
```
- Just like a function pointer says "give me a function and I'll call it," a template template parameter says "give me a template and I'll instantiate it with the types I choose."

- When presented with an equivalent free function or specialization, when will the compiler choose the templated version? 
	- Overload phase first.
		- For **base** templates vs non-template overloads
		- Try to deduce T, and see which works
		- Prefer the non-template version (if equal)
	- Specialisation phase
		- Choose the most specialised version based on partial ordering
		- If ambiguous, compilation error

1. Rank all candidates by conversion quality 
	- (exact match > promotion (int to long) > conversion(int to double) > user defined conversion)
	- for pointers and references, const is considered (low level const)
2. If a template and non-template are tied at the same rank → non-template wins
3. If the template is a better match → template wins
eg.
```cpp
// Base
template <typename T>
T max(T a, T b) {
    return a < b ? b : a;
}

// Spec
template <>
double max(double a, double b) {
    return 0;
}

// Non-template
double max(double a, double b) {
    return a < b ? b : a;
}

int main() {
    double a = 0.1, b = 0.2;
    auto maximum = max(a, b);
}
```
- First, base compared with non-template -> non-template immediately wins

eg.
```cpp
template <class T> 
void foo(T &i) { std::cout << 1; }

template <>
void foo(const int &i) { std::cout << 2; }

int main(){
    int i = 42;
    foo(i);
}
```
- T deduced as `int`
- Base chosen over specialisation, as conversion to `const int` is worse
### Class Templates

- How do templates interact with the static keyword, in particular, static member variables?
    
- Can class templates only be declared in header files?
    
    - If a template is declared in a header file and defined in an implementation file, what do you need to do to allow the compiler to generate templates for the types that you'd like instantited, and why?
        
- Can class templates have virtual methods?
    
- When a class template contains a method that is never called, will the compiler generate code for it? For example, when the class below is instantiated, is code for `foo` generated?
    

```cpp
struct A {
    void foo() { }
    void bar() { }
};

int main() {
    A a;
    a.bar();
}
```

- Which special member functions can be templated?
    
    - When can a templated constructor come in-handy?
        
- How do we specialize a non-templated member function that belongs to a template class?