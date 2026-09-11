# C++ Interview Bank — Questions & Answers

Trimmed to what is actually testable in a Squarepoint-shaped round. 15 questions cut as low-yield (listed at the end).

---

## 1. Object Lifetime & Construction

**1. Walk through everything that happens, in order, when `Derived d;` runs, where `Derived` inherits from `Base` and both have members with constructors.**

- Virtual bases first, constructed by the **most-derived** class, not by whichever intermediate class names them.
- Then non-virtual direct bases, in declaration order.
- Then non-static data members, in **declaration order**, not the order written in the init list.
- Then the constructor body.
- The vptr is repointed at each stage, before that class's member init list runs, so during `Base`'s constructor the object _is_ a `Base`.
- One vptr per polymorphic base subobject, not one per level of a single-inheritance chain.

**2. What is the order of destruction relative to construction, and why that order?**

- Exact reverse: body, members in reverse declaration order, non-virtual bases in reverse, virtual bases last.
- The reason is **dependency**, not stack layout. A member declared later may hold a reference or pointer to one declared earlier, so it must die while its dependency is still alive.

**3. What happens if a constructor throws halfway through? Which destructors run?**

- Only fully constructed subobjects are destroyed. The object was never constructed, so its own destructor never runs and `delete` is never called on it.
- Throw from the **member init list**: only the bases and members completed so far are destroyed.
- Throw from the **body**: all bases and members are complete, so all of their destructors run; the class's own does not.
- Anything the body allocated into a raw pointer leaks. This is the argument for members that own their own resources.

**4. Explain default-initialization, value-initialization, and zero-initialization.**

- **Default-init** (`T obj;`, `new T`): the default constructor runs if there is one; for scalars and trivial types nothing happens and the value is indeterminate.
- **Value-init** (`T obj{}`, `T()`, `new T()`): if the class has a user-provided or deleted default constructor, it just runs. Otherwise the object is **zero-initialized first**, then the defaulted constructor runs.
- **Zero-init**: applied to all static and thread-local storage before anything else, and is the first phase of value-init.
- `int x;` at namespace scope is zero-initialized into `.bss`; inside a function it is indeterminate and **reading it is UB**, not just junk.

**5. What does `T obj();` actually declare, and why is it a classic trap?**

- A function named `obj` taking no arguments and returning `T`. The most vexing parse.
- Worse form: `Widget w(Timer());` declares a function taking a pointer to a function returning `Timer`.
- Fix with braces: `T obj{};`.

**6. When is it legal to call a virtual function inside a constructor, and what does it dispatch to?**

- **Always legal.**
- It dispatches to the most-derived override among classes constructed **so far**, which during `C`'s constructor is `C`'s own override, never a further-derived one.
- Mechanism: the vptr has already been advanced to `C`'s vtable before `C`'s member init list runs. Storage for the whole object was allocated before any constructor ran; constructors only initialize.
- The real trap is calling it from the **member init list**: the vptr already points at `C` but `C`'s members are not yet initialized, so the override reads garbage.

**7. What is placement new? Who calls the destructor?**

- `new (address) T{...}` constructs a `T` in storage you already own. Requires `#include <new>`.
- Storage must be large enough and correctly aligned.
- **You** call the destructor: `p->~T()`.
- Never call `delete` on that pointer; the memory did not come from `operator new`.
- This is exactly what `std::vector` does over `::operator new`.

**8. Explain the static initialization order fiasco and one mitigation.**

- Order of **dynamic** initialization of namespace-scope objects is unspecified **across translation units**. Within one TU it is declaration order.
- If a global in A depends on one in B, it may run first and observe an unconstructed object.
- Mitigation: the **function-local static** (Meyers singleton), initialized on first call, thread-safe since C++11.
- Alternative: `constinit`, which forces constant initialization at compile time so there is no dynamic phase.

---

## 2. Copy & Move Semantics

**9. What are the five special member functions, and when does the compiler generate each?**

- Destructor, copy constructor, copy assignment, move constructor, move assignment. (Default constructor is a sixth special member but not part of the rule of five.)
- **Destructor**: generated unless user-declared.
- **Copy ctor / copy assign**: generated unless that one is user-declared; **deleted** if any move operation is user-declared; still generated but deprecated if a destructor or the other copy operation is user-declared.
- **Move ctor / move assign**: generated only if there is no user-declared destructor, no user-declared copy operation, and no user-declared other move operation.
- **Default ctor**: generated only if no constructor of any kind is user-declared, **including copy and move constructors**.

**10. Writing a user-defined destructor suppresses which special members? What is the consequence?**

- Both move operations.
- Every "move" of the type then silently resolves to a **copy**, because `Type&&` still binds to the copy constructor's `const Type&`.
- This is the most expensive accidental performance bug in the language, and the reason the rule of zero exists.

**11. What does `std::move` actually do? Does it move anything?**

- Nothing at runtime. It is `static_cast<std::remove_reference_t<T>&&>(x)`, an unconditional cast to an rvalue.
- It does not guarantee a move happens: `std::move` into a `const T&` parameter, or on a type with no move constructor, still copies.

**12. After `auto b = std::move(a);` what state is `a` in, (a) for library types, (b) for your own types?**

- Library types: **valid but unspecified**. Every operation with no preconditions still works; you may not assume the value.
- Some types promise more: moved-from `unique_ptr` is guaranteed null, moved-from `shared_ptr` is empty.
- Your own types: whatever you implement, but the contract you owe is the same one, that the destructor and all precondition-free operations still work.

**13. A class has a `const std::string name;` member and otherwise movable members. Move-constructible? Move-assignable?**

- Move-constructible yes, move-assignable no.
- The implicit move constructor initializes `name` from `std::move(other.name)`, whose type is `const std::string&&`. That binds to `const std::string&`, so overload resolution picks the **copy** constructor. The const member is copied; everything else moves.
- Move assignment is not generated because `name` cannot be assigned, and copy assignment is unavailable too, so the type is **not assignable at all**.
- Consequence: unusable with `std::sort`, `std::remove`, `vector::erase`, or any vector reallocation path that assigns.

**14. Why should a move constructor be `noexcept`? Name the library mechanism that depends on it.**

- `std::vector` reallocation uses **`std::move_if_noexcept`**, which selects the copy constructor if the move is not `noexcept` and a copy constructor exists.
- Without `noexcept`, growing a vector copies every element.
- The reason is the **strong exception guarantee**: a throwing move partway through relocation leaves the source elements already gutted with no way back, whereas copies leave the original buffer intact so the vector can free the new one and rethrow.

**15. What is copy elision? Which cases are mandatory since C++17?**

- Constructing the result directly in its destination instead of copying or moving.
- Mandatory since C++17 for **unnamed** temporaries: `T x = T(args)` and `return T(args)`, guaranteed even for types with deleted copy and move constructors.
- **NRVO** (returning a named local) remains optional.
- Consequence: a factory for an immovable type such as `std::mutex` must construct **in the return statement itself**; a named local depends on optional NRVO and fails to compile if it is not applied.

**16. Explain NRVO. Why does `return std::move(local);` usually make things worse?**

- NRVO elides the copy/move of a named local returned by value, constructing it in the caller's storage.
- `return local;` does two things: it first tries NRVO, and if that fails the local is treated as an rvalue so a move is selected.
- `return std::move(local);` makes the returned expression an xvalue rather than the name of a local, which **disqualifies NRVO**. You pay a guaranteed extra move and gain nothing.

**17. When does returning by value beat an output parameter, and when doesn't it?**

- By value wins almost always: elision and moves make it free or near-free, it composes, it works with `const`, it cannot be left uninitialized.
- The output parameter wins when the caller has a buffer whose **capacity should be reused across calls**. A loop calling `fill(vec)` a million times reuses one allocation; returning by value allocates each call.
- Also wins for types that are expensive to move, for filling caller-owned memory, and where a value and a status must be returned separately.

**18. Implement copy-and-swap for `operator=`. What does it buy and what does it cost?**

```cpp
T& operator=(T other) noexcept {   // copy or move happens in the parameter
    swap(*this, other);
    return *this;
}
```

- **Buys**: strong exception guarantee (all allocation happens before `*this` is touched), self-assignment safety for free, one function for both copy and move assignment.
- **Costs**: it **always allocates**, so `a = b` where both already have capacity cannot reuse `a`'s buffer, a real hot-path regression; requires a `noexcept` swap; releases the old resource later than necessary, at parameter destruction; and you cannot additionally declare separate copy- and move-assignment overloads, since the calls become ambiguous.

**19. What is the difference between `T&&` where `T` is a template parameter versus a concrete type?**

- Template parameter deduced from the call: **forwarding reference**, binds to lvalues and rvalues, deduces `T = U&` for lvalues.
- Concrete type, or a class template parameter already fixed, or `const T&&`, or `std::vector<T>&&`: plain **rvalue reference**, binds to rvalues only.

**20. Write the move constructor and move assignment for `class Buffer { char* data_; size_t size_; };`**

```cpp
Buffer(Buffer&& other) noexcept
    : data_(std::exchange(other.data_, nullptr)),
      size_(std::exchange(other.size_, 0)) {}

Buffer& operator=(Buffer&& other) noexcept {
    if (this == &other) return *this;
    delete[] data_;                              // release what we own
    data_ = std::exchange(other.data_, nullptr);
    size_ = std::exchange(other.size_, 0);
    return *this;
}
```

- The constructor **cannot** be swap-based: the members are uninitialized, so swapping puts indeterminate values into the source.
- The assignment must free the existing buffer. A swap-based assignment merely defers the free to the source's destructor, which is a different design and leaks if the source is long-lived.

---

## 3. Value Categories, References & Forwarding

**21. Define lvalue, prvalue, xvalue, with one expression each.**

- **lvalue**: has identity, cannot be moved from. The expression `x` for `int x;`.
- **prvalue** ("pure" rvalue): no identity, initializes something. `42`, or `f()` returning by value.
- **xvalue** ("expiring"): has identity and can be moved from. `std::move(x)`, or a function returning `T&&`.
- The category belongs to the **expression**, not the variable.

**22. `std::string&& r = std::move(s);` — is `r` an lvalue or rvalue in a later expression? Why does it matter for forwarding?**

- lvalue. Any named entity used as an expression is an lvalue regardless of its declared type.
- That is exactly why forwarding needs `std::forward`: inside `f(T&& x)`, `x` is an lvalue even when the caller passed an rvalue, so passing it on unchanged silently copies.

**23. Does binding a temporary to `const T&` extend its lifetime? To `T&&`? Does returning a `T&&` bound to a local extend anything?**

- `const T&`: yes, extended to the lifetime of the reference.
- `T&&`: yes, same rule.
- Returning one: **no**. Extension never crosses a function boundary, and it also does not apply when a temporary binds to a reference **member** in a constructor init list.

**24. What are the reference collapsing rules?**

- `& &` → `&`
- `& &&` → `&`
- `&& &` → `&`
- `&& &&` → `&&`
- Rule of thumb: lvalue reference wins.

**25. What is `std::forward` and why can't `std::move` replace it?**

- `std::forward<T>(x)` is a **conditional** cast: it yields an rvalue only when `T` was deduced as a non-reference (caller passed an rvalue), and an lvalue when `T` deduced as `U&`.
- `std::move` is unconditional, so it would turn caller lvalues into rvalues and steal from objects the caller still owns.

**26. Write a `wrapper` that perfectly forwards any argument to `g`. Explain each token.**

```cpp
template <typename T>
decltype(auto) wrapper(T&& arg) {
    return g(std::forward<T>(arg));
}
```

- `T` is deduced from the call.
- `T&&` is a forwarding reference **because** `T` is deduced here.
- `std::forward<T>` restores the caller's value category.
- `decltype(auto)` preserves `g`'s return type exactly, including references; plain `auto` would strip the reference and copy.
- Variadic form: `template <typename... Ts> decltype(auto) wrapper(Ts&&... a) { return g(std::forward<Ts>(a)...); }`

**27. Why is `auto&&` useful in range-for loops? When over `const auto&`?**

- Same deduction rules as a forwarding reference, so it binds to anything.
- Primary reason: **proxy references**. `std::vector<bool>::operator*` returns a prvalue proxy, which `auto&` refuses to bind and `const auto&` binds only as an unwritable copy. `auto&&` binds it correctly.
- Second reason: when you intend to move elements out of the range.

**28. What does `decltype(x)` give vs `decltype((x))` for a local `int x;`?**

- `decltype(x)` is `int`: for an unparenthesized name it yields the declared type.
- `decltype((x))` is `int&`: the parentheses make it an expression, and for an lvalue expression of type `T` the result is `T&`.

**29. What's the danger in `const std::string& s = cond ? str1 : "literal";`?**

- The conditional operator's common type is `std::string`, so a temporary is materialized in **both** branches, including when `cond` is true.
- The const reference binding extends that temporary's lifetime, so this does **not** dangle.
- The danger is cost and aliasing: a heap allocation on a line that looks like a free reference binding, and `s` does not refer to `str1`, so later mutations of `str1` are invisible through `s`.

**30. `void f(int&); void f(int&&);` — which overload for `f(5)`, `f(x)`, `f(std::move(x))`, `f(x + 1)`?**

- `f(5)` → `int&&`
- `f(x)` → `int&`
- `f(std::move(x))` → `int&&`
- `f(x + 1)` → `int&&` (the result of `x + 1` is a prvalue)

**31. Explain the template deduction rules.**

- Deduce `T`, then substitute and let references collapse.
- `T` **by value**: strips top-level `const`/`volatile` and references; arrays and functions decay to pointers.
- `T&`: keeps `const`; a `const int` argument deduces `T = const int`.
- `const T&`: `T` never picks up the `const`; binds to everything.
- `T&&` **forwarding**: lvalue of type `U` deduces `T = U&`, rvalue deduces `T = U`, then collapsing applies.
- `auto` follows the same rules with one exception: `auto x = {1,2,3}` deduces `std::initializer_list<int>`, which template deduction would reject.

---

## 4. const, constexpr, and Compile Time

**32. What's the difference between `const`, `constexpr`, and `consteval` on a function?**

- `const` (after a member function): does not modify the object, callable on const objects.
- `constexpr`: **may** be evaluated at compile time when called with constant arguments, runs normally otherwise.
- `consteval`: an immediate function, **must** produce a constant expression, so every call is evaluated at compile time and it can never take runtime arguments.

**33. Can a `constexpr` function be called at runtime? Contain a loop? Throw?**

- Runtime: yes, that is the normal case.
- Loops: yes since C++14 (C++11 allowed only a single return).
- Throw: yes, the `throw` may appear. If it is actually reached during constant evaluation the call simply is not a constant expression and you get a compile error; the same call at runtime throws normally.
- Since C++20 they may also allocate, use `try`, and contain virtual calls.

**34. What does `mutable` do? Give a legitimate use case.**

- Allows a member to be modified through a const member function or on a const object.
- Legitimate: a `std::mutex` member locked inside a `const` getter, a memoization cache, a lazily computed value, a hit counter.
- The unifying idea is **logical constness**: observable state is unchanged even though bytes are not.

**35. What does `const` after a member function mean in terms of `this`? What about `&&`?**

- `const` makes `this` a `const T*`: callable on const objects, cannot modify non-mutable members.
- `&` and `&&` are **ref-qualifiers** on the implicit object parameter. `void f() &` is callable only on lvalues, `void f() &&` only on rvalues.
- Practical use: returning a member by move from an expiring object, `std::string get() && { return std::move(s_); }`.

**36. What is `if constexpr` and how does it differ from a regular `if` in a template?**

- Requires a compile-time constant condition and **discards** the untaken branch.
- The discarded branch is uninstantiated only **inside a template**, and only for constructs that depend on a template parameter, so it may be ill-formed for that `T` and still compile.
- A regular `if` requires both branches to be valid for every instantiation and chooses at runtime.

**37. What's the difference between `static constexpr` and `inline constexpr` at namespace scope, for headers?**

- `constexpr` implies `const`, and a namespace-scope `const` variable has **internal linkage** by default, so every TU gets its own copy.
- Fine until something odr-uses it (takes its address, binds a reference) from an inline function or template: different TUs then see different addresses, an ODR violation.
- `inline constexpr` gives external linkage with one merged definition. That is the fix.
- `static` at namespace scope only makes the internal linkage explicit.
- One-liner: **`constexpr` implies `inline` for functions but implies internal linkage for variables**, and that is the root of the problem.

---

## 5. Templates

**38. Explain SFINAE, and show a minimal `enable_if` gating a function on `is_integral`.**

- During template argument deduction the compiler substitutes deduced arguments into the **signature**. If that produces an invalid type or expression **in the immediate context of the signature**, the candidate is silently removed from the overload set instead of erroring.
- Only if no candidate survives do you get a diagnostic.
- An error inside the function **body** is not SFINAE, it is a hard error.

```cpp
template <typename T>
std::enable_if_t<std::is_integral_v<T>, void> f(T x);   // vanishes for non-integral T
```

- `enable_if_t<true, T>` is `T`; `enable_if_t<false, T>` has no `type` member, so substitution fails.

**39. What do C++20 concepts replace, and what do they give you beyond nicer errors?**

- They replace `enable_if`, tag dispatch, and most of the detection idiom.
- **Subsumption**: given two viable constrained overloads, the more constrained one wins automatically. With `enable_if` you had to hand-build mutually exclusive conditions or an int/long tag hierarchy.
- **Named, reusable predicates** that appear in the interface, so the constraint is documentation.
- **Checked before instantiation**, so an unsatisfied concept is reported at the call site rather than deep inside a body.
- **Usable in more positions**: `Numeric auto x`, on non-template members of class templates, and on constructors, where `enable_if` was awkward or impossible.

**40. What is CRTP? Write the skeleton and give the trade-offs vs virtual.**

```cpp
template <typename Derived>
class Base {
public:
    void interface() { static_cast<Derived*>(this)->impl(); }
};
class D : public Base<D> {
public:
    void impl() { /* ... */ }
};
```

- The base knows the derived type as a template parameter, so the `static_cast` is valid and the call resolves **statically**.
- **Benefits**: no vptr, so objects are smaller; no dependent load and no indirect branch; the call inlines, which unlocks everything that follows inlining.
- **Costs**: no runtime polymorphism (cannot store `Base<A>` and `Base<B>` in one container); each derived type instantiates its own base, so code size and compile time grow; a bloated binary pollutes the instruction cache.
- The one-line interview answer: CRTP is how you get polymorphism in a hot path without paying for dispatch.

**41. Why must template definitions live in headers? What is explicit instantiation?**

- Every TU that instantiates a template must see its definition, because instantiation is compile-time code generation and the compiler cannot generate code it cannot see.
- **Explicit instantiation** (`template class Foo<int>;` in one .cpp) forces that specialization to be emitted there; `extern template class Foo<int>;` in the header tells other TUs not to instantiate it themselves.
- Keeps the definition in one .cpp and cuts compile time, at the price of supporting only the argument types you listed.

**42. What is two-phase lookup? Why do you sometimes need `this->` or `typename` inside templates?**

- Names split into **non-dependent** (looked up at definition time, in the context where the template is written) and **dependent** (looked up at instantiation time, with ADL).
- Consequence: a member inherited from a **dependent base** is not found by unqualified lookup at definition time, since the compiler cannot know what `Base<T>` contains. `this->member` or `using Base<T>::member` makes the name dependent so lookup is deferred.
- `typename` is needed before a dependent qualified name used as a type (`typename std::vector<T>::iterator`), because the compiler must decide at parse time whether it names a type or a value. `template` is the analogous keyword before a dependent member template (`obj.template get<int>()`).

**43. What is a variadic template? Write `sum(args...)` with a fold expression.**

```cpp
template <typename... Ts>
auto sum(Ts... args) { return (args + ... + 0); }   // binary right fold; empty pack gives 0
```

- Syntax map: `(... op pack)` unary left, `(pack op ...)` unary right, `(init op ... op pack)` binary left, `(pack op ... op init)` binary right.
- `sizeof...(args)` gives the count.
- The empty-pack case is why you usually supply an init value.

---

## 6. Inheritance & Virtual Dispatch

**44. Describe the memory layout of an object with virtual functions. For single inheritance with a 3-level hierarchy, how many vptrs?**

- One hidden vptr per **polymorphic base subobject**, placed first in the object on the Itanium ABI.
- A 3-level single-inheritance hierarchy gives exactly **one** vptr, pointing at the most-derived class's vtable. What differs per class is the vtable, not the count.
- Multiple inheritance from two polymorphic bases gives two vptrs.
- The vtable holds function pointers, an offset-to-top, and a pointer to the `type_info` for RTTI. (You missed the `type_info` pointer in R2.)

**45. Walk through what happens at machine level for `p->f()` where `f` is virtual. How many loads, and why does it hurt?**

- Load the vptr from the object (one load).
- Load the function pointer from a fixed slot in the vtable (a second load, **dependent** on the first).
- Indirect call.
- Hurts because the two loads serialize (the second address is unknown until the first returns), the vtable may be cold, the indirect branch can mispredict when the target varies across iterations, and above all the compiler **cannot inline** through it and therefore loses every optimization that would have followed.

**46. When is a virtual destructor required? What goes wrong without one?**

- Whenever an object may be deleted through a pointer to a base.
- Without it, `delete basePtr` with a derived dynamic type is **UB**; in practice only the base destructor runs, so derived members leak and derived cleanup never happens.
- Not required if you never delete polymorphically. The modern alternative is a `protected` non-virtual destructor, which makes the mistake impossible.

**47. What does `final` enable, and what is devirtualization?**

- `final` tells the compiler no further override exists, so a call through that static type has exactly one target and can be **devirtualized** and then inlined. On a class it does this for all its virtuals.
- Devirtualization is possible whenever the dynamic type is known: an object used by value or by name, a `final` class or method, a class in an anonymous namespace with one derived type, or under LTO where the whole hierarchy is visible.
- Compilers also do **speculative** devirtualization behind a type check, which pays off when one type dominates.
- This is why a virtual function _can_ be inlined: what matters is whether the target is statically determined, not the `inline` keyword.

**48. Explain virtual inheritance and the diamond problem.**

- Without `virtual`, `D : B, C` where both derive from `A` contains **two** `A` subobjects, so `A`'s members are ambiguous and duplicated.
- With `B : virtual A`, all paths share one `A`.
- **Layout** changes: the shared base is placed separately (typically at the end) and access goes through a virtual-base offset stored in the vtable, so member access costs an extra indirection and casts are no longer simple pointer arithmetic.
- **Construction** changes: the **most-derived** class constructs the virtual base first, and any initializer for it in intermediate classes is ignored.

**49. What is object slicing? Show a two-line example.**

```cpp
void f(Base b);      // by value
Derived d;  f(d);    // the Derived part is silently sliced off
```

- The base copy constructor copies only the base subobject, and the copy's vptr is the base's, so virtual calls dispatch to `Base`.
- Also happens with `Base b = d;` and `vector<Base> v; v.push_back(d);`.
- Prevent by taking base parameters by reference, or by deleting the base's copy operations.

**50. `override` vs `virtual` — what does `override` actually check? Give a bug it catches.**

- `virtual` declares participation in dynamic dispatch. `override` **asserts** that this function overrides a base virtual, and is a compile error if it does not.
- Catches: a missing `const` (`void f()` vs `void f() const`), a different parameter type (`int` vs `long`), a different return type, or a base function that was never virtual.
- Without it, `Derived::f()` becomes a separate overload and every call through a `Base*` silently runs the base version.

**51. Why is calling a pure virtual from a base constructor a runtime error rather than a compile error?**

- Dispatch is dynamic and the vptr is a runtime value. During `Base`'s constructor the vptr points at `Base`'s vtable, whose slot for the pure virtual holds `__cxa_pure_virtual`, which prints a message and aborts.
- The compiler cannot diagnose it in general because the call may be indirect, through another function the constructor calls.
- The standard calls it UB; the implementation gives you the abort.

---

## 7. STL Containers & Internals

**52. Describe `std::vector` growth. Why is `push_back` amortized O(1)? `reserve` vs `resize`?**

- Capacity is multiplied by a constant factor on reallocation: 2 in libstdc++ and libc++, 1.5 in MSVC.
- Growth by a constant **factor** makes total copy work across n pushes O(n), hence amortized O(1). A constant **increment** would make it O(n²).
- `reserve(n)`: changes capacity only, constructs nothing, `size()` unchanged.
- `resize(n)`: changes size, value-initializing new elements or destroying surplus ones.
- Reserving and then indexing `v[i]` is UB; that needs `resize`.

**53. When `vector` reallocates, does it move or copy? What decides?**

- `std::move_if_noexcept`: moves if the move constructor is `noexcept` or if no copy constructor exists, copies otherwise.
- The driver is the **strong exception guarantee**: a throwing move partway through relocation damages both buffers, while copies leave the original intact.

**54. Compare `std::map` and `std::unordered_map`. Which for a latency-sensitive path, and why might the answer be "neither"?**

- `map`: red-black tree of separately allocated nodes, O(log n) with **dependent pointer chases**, iteration in key order, stable references and iterators.
- `unordered_map`: bucket array plus per-bucket chains of separately allocated nodes, O(1) average and O(n) worst, no order, rehash on load-factor growth.
- On a hot path both are bad: both allocate per node and both chase pointers into cold lines. A `map` lookup is roughly 20 dependent misses versus a few hot lines for a flat structure, so 5-10x.
- **Neither**: a sorted `std::vector` with `lower_bound` for lookup-heavy data, a flat open-addressed table, or a **direct-indexed array** when the key space is small and dense (the 6500-symbol slot mapping shape).

**55. How does `unordered_map` handle collisions? What is `load_factor`, and what happens on rehash?**

- In practice separate **chaining**: each bucket holds a singly linked list of nodes.
- `load_factor()` is `size() / bucket_count()`. When an insert would exceed `max_load_factor()` (default 1.0), the container rehashes to a larger bucket count and redistributes everything.
- Rehash invalidates **all iterators** but **not references or pointers**: the nodes do not move, only the bucket array is rebuilt and relinked.

**56. Why is `std::deque` not just "vector with fast front insertion"? Sketch its structure.**

- An array of pointers (the map) to fixed-size **chunks** of elements.
- Indexing costs two indirections plus arithmetic rather than one.
- Growth at either end allocates a new chunk and possibly reallocates the map, but **never moves existing elements**, which is why references survive end insertion while iterators do not.
- Elements are contiguous only within a chunk, so iteration is fast but not vector-fast, and there is no `data()`.

**57. Why is `vector<bool>` notorious?**

- It is a specialization packing one bit per element, so it is not a container of `bool` and violates the container requirements.
- `operator[]` returns a **proxy**, not `bool&`.
- Consequences: `auto x = v[0]` gives a proxy aliasing the vector, not a copy; `bool* p = &v[0]` does not compile; `auto& r = v[0]` does not compile; it cannot feed APIs expecting `bool*`; and concurrent writes to distinct elements are **not** safe, unlike every other vector.
- Use `vector<char>`, `deque<bool>`, or `bitset`.

**58. What does `std::span` give you, what does it not own, and what's the danger?**

- A non-owning view over contiguous memory: a pointer plus a length (or a compile-time extent).
- Owns nothing, reference semantics.
- Dangles the moment the underlying container reallocates, is destroyed, or the temporary it was built from dies. Treat it as a raw pointer with a size: fine as a parameter, dangerous as a member or a return value.

**59. `emplace_back` vs `push_back` — the real difference, and a case where `emplace_back` is a trap.**

- `push_back` takes an object and copies or moves it. `emplace_back` forwards its arguments to construct in place, saving one move when you would otherwise build a temporary.
- **Traps**: it uses **direct** initialization, so it invokes `explicit` constructors `push_back` would refuse (`v.emplace_back(10)` on a `vector<vector<int>>` silently creates a 10-element vector); no narrowing check; for an aggregate with no constructor, `emplace_back(a, b)` does not compile before C++20; and if you already hold an object, it is no faster than `push_back`.

**60. What is small string optimization, and what changes about moves?**

- Short strings are stored **inline** in the string object with no heap allocation. About 15 characters in libstdc++ and MSVC, 22 in libc++.
- Moving a short string **copies the bytes**; it is not a pointer steal, so it is not free.
- More importantly, pointers, references, and iterators into a **short** string are invalidated by a move or swap, whereas for a long string they follow the buffer.
- It also makes `std::string` larger than a pointer plus size, and makes move cost data-dependent.

**61. How is `std::sort` implemented, and why isn't quicksort alone acceptable?**

- **Introsort**: quicksort with median-of-three pivoting, switching to **heapsort** once recursion depth exceeds about 2·log₂(n), finishing with **insertion sort** on small ranges (typically ≤16).
- Plain quicksort has an O(n²) worst case triggerable by adversarial or merely unlucky input, while the standard requires O(n log n) comparisons.
- The heapsort fallback caps the worst case; the insertion-sort tail exploits near-sortedness and cache locality.

**62. What are the guarantees of `std::nth_element`, and when do you use it over `sort`?**

- Rearranges so position n holds the element that would be there if sorted, everything before compares not-greater and everything after not-less. The partitions are otherwise unordered.
- O(n) average via introselect (quickselect with a median-of-medians fallback).
- Use it for the k-th element or an unordered top-k.
- **For the k-closest-to-mid question**: `nth_element` to position k in O(n), then sort only the first k in O(k log k). Beats sorting all n.

**63. Compare `priority_queue`, a sorted `vector`, and `multiset` for top-k with frequent inserts.**

- **Bounded max-heap** (`priority_queue` with the reversed comparator): O(log k) insert, O(1) to test against the current worst, contiguous storage. No ordered iteration, no arbitrary removal.
- **Sorted vector**: O(log k) to find the position, O(k) to shift, but the shift is a contiguous `memmove` so it wins for small k (up to a few dozen) and gives ordered iteration free.
- **`multiset`**: O(log k) insert and erase-anywhere plus ordered iteration, but a node allocation per insert and pointer chasing. Slowest in practice.
- Default to the heap for pure top-k, the sorted vector for small k or when you need order, `multiset` only when you must erase arbitrary elements.

---

## 8. Iterators & Invalidation

**64. State the iterator invalidation rules for `std::vector`: `push_back`, `insert` in the middle, `erase`.**

- `push_back`: if it **reallocates**, everything (iterators, pointers, references) is invalidated; if not, only `end()` is invalidated and existing elements are untouched.
- `insert` in the middle: if it reallocates, everything; if not, iterators and references **at or after** the insertion point are invalidated, those strictly before remain valid.
- `erase`: iterators and references **at or after** the erased element are invalidated, those before remain valid. No reallocation ever occurs on erase, and capacity does not shrink.

**65. Same for `std::deque` — front, back, middle. (References and iterators differ.)**

- Insertion at **either end**: **all iterators** invalidated, but **no references or pointers** to existing elements. The elements never move; only the bookkeeping does.
- Insertion in the **middle**: all iterators and all references invalidated.
- Erase at either end: only iterators and references to the erased elements (plus `end()` for a back erase).
- Erase in the middle: all iterators and references invalidated.
- The end-insert case is the one that surprises people and the one to be able to say aloud.

**66. Same for `std::map` / `std::set` — insert and erase.**

- Insert invalidates **nothing** at all.
- Erase invalidates only iterators and references to the **erased element**.
- Everything survives because each element is a separately allocated node that never moves.
- This is why node-based containers are usable when you must hold long-lived handles, and why `it = m.erase(it)` is the erase-loop idiom.

**67. Same for `unordered_map` — insert without rehash, insert with rehash, erase. What survives a rehash?**

- Insert, no rehash: nothing invalidated.
- Insert triggering a **rehash**: **all iterators** invalidated, **references and pointers stay valid**.
- Erase: only the erased element's iterators and references.
- References surviving a rehash is the surprising part, and the mechanism is that rehashing rebuilds the bucket array and relinks nodes **without moving them**.

**68. What does `erase` return? Write the three erase-all-evens loops.**

- Returns an iterator to the element following the last one removed.

```cpp
// (a) manual
for (auto it = v.begin(); it != v.end(); )
    if (*it % 2 == 0) it = v.erase(it); else ++it;

// (b) erase-remove
v.erase(std::remove_if(v.begin(), v.end(), [](int x){ return x % 2 == 0; }), v.end());

// (c) C++20
std::erase_if(v, [](int x){ return x % 2 == 0; });
```

**69. Why is a missing second argument to `erase` a subtle bug family? What does `std::remove` actually do?**

- `std::remove` **cannot change the container's size**. It moves the elements you are keeping to the front and returns an iterator to the new logical end. The tail holds unspecified moved-from values.
- `v.erase(first, last)` is what actually shrinks the container.
- `v.erase(std::remove(...))` compiles, picks the single-iterator overload, and removes exactly **one** element, leaving the moved-from junk behind and the size wrong by n-1.

**70. This compiles. What's wrong?**

```cpp
for (auto it = v.begin(); it != v.end(); ++it)
    if (*it == 0) v.erase(it);
```

- `erase` invalidates `it`, and the subsequent `++it` on an invalidated iterator is **UB**.
- Even under a forgiving implementation the logic is wrong: erasing shifts the next element into the current position and `++it` then skips it, so consecutive zeros are missed.
- Fix with `it = v.erase(it)` or erase-remove.

**71. Do references survive `vector::push_back`? `map::insert`? An `unordered_map` rehash?**

- Vector: only if there is no reallocation, so treat them as dead.
- Map: **yes, always**.
- Unordered_map rehash: **yes**, references and pointers survive; only iterators die.

**72. `auto& x = v[3]; v.reserve(1000); use(x);` — okay? What if it were `shrink_to_fit`?**

- Not okay unless `capacity()` was already at least 1000. `reserve` reallocates, moving every element, so `x` dangles and `use(x)` is UB.
- `shrink_to_fit` is sneakier: a non-binding request that is **permitted** to reallocate, so it may or may not invalidate.
- Treat both as invalidating.

**73. What iterator category does each container provide, and why does `std::sort` refuse some?**

- `vector`: contiguous (random access, plus the C++20 contiguous tag). `deque`: random access, not contiguous. `list`: bidirectional. `forward_list`: forward. `map`/`set`: bidirectional. `unordered_map`: forward. `istream_iterator`: input, single-pass.
- `std::sort` needs **random access** because pivoting and partitioning require `it + n` and `it2 - it1` in constant time. Hence `list` provides its own `sort` member, and input iterators cannot be sorted at all since they cannot be traversed twice.

**74. Dangling vs singular iterator. Is comparing an invalidated iterator to `end()` defined?**

- **Singular**: not associated with any container, e.g. default-constructed. Only assignment and destruction are defined.
- **Dangling / invalidated**: did refer to an element, but a container operation destroyed the association.
- Comparing an invalidated iterator to `end()` is **UB**, not merely unspecified. There is no defined way to test validity, which is why the rules have to be memorized.

**75. You hold iterators into a `std::vector` order book while another path inserts. Enumerate your options and their costs.**

- **Indices instead of iterators**: survive reallocation, cost one addition per dereference, still broken by erase-in-the-middle shifting.
- **Reserve the maximum up front**: no reallocation ever, simple and fast, but caps capacity and is a latent bug if exceeded.
- **A stable container** (`list`, `deque` for end insertion, or a `map` of price levels): buys stability, costs an allocation per node and pointer chasing.
- **Slot array plus free list**: preallocate N slots, hand out indices, never move an element. This is the pool allocator answer and what real order books do.
- **Generation counters / handles**: index plus a generation stamp validated on dereference; catches stale handles for the cost of a branch.
- **Copy the values out**: correct by construction, costs a copy.
- **Intrusive lists**: nodes embedded in the order objects, O(1) cancel with no allocation.
- The interview move is to name the trade-off axis (stability vs locality vs allocation) and then pick the slot array, because it is stable, contiguous, and allocation-free.

---

## 9. Memory, Allocators & Smart Pointers

**76. What does `new T[10]` do beyond allocating? Why must it pair with `delete[]`?**

- Allocates the array plus, for types with a non-trivial destructor, a hidden **cookie** storing the element count, then default-initializes each element in order.
- `delete[]` reads the cookie, destroys elements in reverse, and frees from the correct base address (offset by the cookie).
- Plain `delete` runs one destructor and frees the wrong address: UB, typically heap corruption rather than a clean crash.
- If a constructor throws partway, already-constructed elements are destroyed and the memory is freed.

**77. Explain `shared_ptr` layout and control-block mechanics. What does `make_shared` change, and what is the weak-pointer caveat?**

- A `shared_ptr` is **two pointers**: one to the object, one to the control block.
- The control block holds an atomic **strong** count, an atomic **weak** count, the deleter, and the allocator.
- Strong reaching 0 destroys the **object**; weak reaching 0 frees the **control block**. All `shared_ptr`s together count as **one** weak reference.
- Only `shared_ptr` touches the strong count; `weak_ptr` merely holds a pointer to the same block. (Your R2 miss.)
- `make_shared`: **one allocation** instead of two, control block adjacent to the object, and it closes the exception-safety window in `f(shared_ptr<T>(new T), g())`.
- **Caveat**: the fused block cannot be freed until the **weak** count hits zero, so one lingering `weak_ptr` pins the object's storage. Bad for large objects. Also: no custom deleter, and no `new T[]`.

**78. What does a `shared_ptr` copy cost in a multithreaded program, mechanically? Why is by-value in a hot path a smell?**

- Copy: an atomic increment of the strong count (relaxed suffices). Destructor: an atomic decrement with acquire-release, plus a branch on zero.
- The cost is not the instruction, it is the **cache line**. The count is shared, so every thread touching it takes exclusive ownership of that line and invalidates it in every other core. Under contention this is a line ping-pong costing hundreds of cycles, scaling negatively with core count.
- Pass `const shared_ptr&`, or better `T&`/`T*` when the callee does not participate in ownership.
- By-value in a hot signature forces a refcount round trip per call for nothing.

**79. What is `weak_ptr` for? Walk through `lock()` — can it race with the last `shared_ptr` dying?**

- A non-owning observer that can safely ask whether the object is still alive.
- `lock()` returns a `shared_ptr`, null if the object is gone. Test the returned `shared_ptr`, not the `weak_ptr`.
- It **cannot** race, and that is why it is not `if (!expired()) return shared_ptr(*this);` — between the check and the increment the count could hit zero.
- It is a **CAS loop**: load the strong count, bail if zero, otherwise `compare_exchange_weak` to count+1, retry on failure. An atomic increment-if-nonzero.
- Uses: breaking cycles, caches, observer lists, async callbacks (`weak_from_this()`).

**80. `unique_ptr<T>` vs a raw pointer: size and overhead? What about a lambda vs function-pointer deleter?**

- Default deleter: exactly the size of a raw pointer, zero runtime overhead. The deleter is an empty type stored via **empty base optimization**, and `delete` inlines into the destructor.
- Stateless lambda or functor deleter: still one pointer, since the closure type is empty.
- **Function pointer** deleter: two pointers, and the call cannot be inlined.
- Hence prefer a functor or captureless lambda type over `void(*)(T*)`.

**81. What does `enable_shared_from_this` solve, and what goes wrong in a constructor?**

- It lets a member function obtain a `shared_ptr` to `this` **without** creating a second, independent control block (which would double-free).
- Mechanism: the object stores a `weak_ptr` to itself, which the **first** `shared_ptr` constructed from a raw pointer populates.
- In the constructor no `shared_ptr` owns the object yet, so the internal weak reference is empty and `shared_from_this()` throws `std::bad_weak_ptr`. Same on a stack object.
- Prefer `weak_from_this()` in callbacks: it returns an empty `weak_ptr` rather than throwing.

**82. What is an allocator? What is `pmr::monotonic_buffer_resource` and when is it a big win?**

- The container-facing interface for obtaining and releasing **raw storage**, deliberately separated from construction and destruction. That separation is why a vector can have capacity beyond size.
- The real interface is `std::allocator_traits`; a minimal allocator needs only `value_type`, `allocate`, `deallocate`, `operator==`.
- `monotonic_buffer_resource`: a **bump-pointer arena** over a buffer you supply. Allocation is a pointer increment, `deallocate` is a no-op, everything is released at once.
- Big win when a phase has many short-lived allocations that all die together (per-message, per-tick), since it removes all `malloc` calls, all frees, and all fragmentation from that phase.

**83. Sketch a fixed-size pool allocator. Why do trading systems use them instead of the heap?**

- Preallocate N slots of `sizeof(T)`, thread a singly linked **free list through the unused slots themselves** (each free slot stores the next free index), so bookkeeping is free.
- `allocate` pops the head, `deallocate` pushes back. Both O(1), a couple of instructions, no size-class branching.
- The general heap has a **variable, unbounded tail**: it may take an arena lock, walk free lists, fragment, or go to the OS and take a page fault.
- A pool gives constant predictable cost, keeps objects contiguous and cache-friendly, and can be per-thread so it needs no synchronization.
- Related: an **arena** frees everything at once, which is O(1) and eliminates fragmentation entirely, at the cost that one long-lived object keeps the whole arena alive.

**84. What is memory alignment? What does `alignas(64)` do and why 64?**

- An object of type `T` must sit at an address that is a multiple of `alignof(T)`, because hardware loads are cheaper or only legal at natural boundaries and aligned SIMD requires it.
- `alignas(64)` forces 64-byte alignment. 64 is the **cache line size** on x86-64, so an aligned, padded member starts a fresh line and shares it with nothing.
- That is the false-sharing fix. Portable spelling: `std::hardware_destructive_interference_size`.

**85. `malloc` + placement new vs plain `new`. When do you separate allocation from construction deliberately?**

- Plain `new T` does two things: `operator new(sizeof(T))` then constructs a `T`, and throws `std::bad_alloc` on failure.
- `malloc` + placement new splits them, returns null instead of throwing, ignores over-aligned types (use `aligned_alloc`), and requires a manual destructor call.
- Separate them whenever **storage lifetime and object lifetime differ**: vector capacity beyond size, an object pool reusing slots, a ring buffer of slots, a small-buffer-optimized type, deserializing into a preallocated arena.

**86. Double-delete, use-after-free, leak: one tool and one structural prevention each.**

- **Double-delete**: ASan or a debug allocator. Prevent with `unique_ptr` (single owner, no way to delete twice).
- **Use-after-free**: ASan, which quarantines freed memory and reports the access. Prevent by making ownership explicit and using `weak_ptr` or handles for non-owning references; never return references to locals.
- **Leak**: LeakSanitizer or Valgrind. Prevent with RAII: every resource owned by an object whose destructor releases it.
- One line: all three are **ownership bugs**, and RAII plus single-owner discipline removes the class.

---

## 10. Integer Semantics, Casts & UB

**87. `for (int i = 0; i < v.size(); ++i)` — what happens at the type level, when does it misbehave, and your standing fix?**

- `v.size()` is `std::size_t`, unsigned and usually 64-bit. The usual arithmetic conversions convert `i` to `size_t`.
- Misbehaves when `i` is negative: it converts to a huge unsigned value, so the comparison is false and the loop stops or never starts. Also emits `-Wsign-compare`, which people learn to ignore.
- Fixes: range-for, an index of type `std::size_t`, or `std::ssize(v)` (C++20) which returns a signed size.

**88. What happens on signed overflow? Unsigned? Which does the compiler exploit, and for what?**

- Signed overflow is **UB**. Unsigned wraps modulo 2ⁿ, fully defined.
- The compiler exploits the signed case: because `i + 1 > i` is assumed, it can prove loops terminate, promote a 32-bit `int` induction variable into a 64-bit register with no wrap checks, and simplify `i*2/2` to `i`.
- Concrete: `for (int i = 0; i <= n; ++i)` has a computable trip count and can be vectorized; with `unsigned i` the compiler must handle wraparound and may refuse.

**89. `int a = -1; unsigned b = 1; if (a < b)` — what and why?**

- **False**. The usual arithmetic conversions convert `a` to `unsigned` (same rank, unsigned wins), so `a` becomes 4294967295, which is not less than 1.
- Same mechanism behind `i < v.size()` bugs and behind `if (v.size() - 1 >= 0)` always being true.

**90. What are the four named casts, one use each? Which can fail at runtime, and how do you observe it?**

- `static_cast`: related-type conversions checked at compile time (numeric conversions, up-casts, down-casts you know are safe, `void*` back to `T*`).
- `dynamic_cast`: checked down-cast or cross-cast in a polymorphic hierarchy. **Pointer form returns `nullptr`, reference form throws `std::bad_cast`.** This is the one that can fail at runtime, and it costs an RTTI lookup. (Your R2 miss: you had it right first, then talked yourself into "throws" for both.)
- `const_cast`: adds or removes `const`/`volatile`, legitimately to call a legacy C API that takes non-const but does not modify. Writing through it to an object actually **defined** const is UB, and such objects may live in read-only pages.
- `reinterpret_cast`: bit-pattern reinterpretation, legitimately for pointer-to-integer round trips and hardware addresses.

**91. What does `reinterpret_cast<float*>(&myInt)` then dereference violate? What are the sanctioned ways to type-pun?**

- Violates **strict aliasing**: accessing an object through a glvalue of an unrelated type is UB, so the compiler may assume the `int` and `float` accesses never touch the same memory and reorder or elide them. May also violate alignment.
- Sanctioned: `std::bit_cast` (C++20); `std::memcpy` into a properly typed object, which optimizers recognize and compile to a single move; or accessing bytes through `char`, `unsigned char`, or `std::byte`, which may alias anything.
- A union is defined for this in C but is technically UB in C++, though every major compiler supports it.

**92. UB vs unspecified vs implementation-defined, with an example each.**

- **UB**: no requirements at all, the whole program is meaningless. Signed overflow, dereferencing null, data races.
- **Unspecified**: several behaviors allowed, no documentation required. Order of evaluation of function arguments; the value of a moved-from library object.
- **Implementation-defined**: unspecified but the implementation must **document** its choice. `sizeof(int)`, whether `char` is signed.

**93. `i = i++ + 1;` — status in C++17? What changed about evaluation order generally?**

- **Well-defined since C++17**, UB before.
- In a simple assignment the **right** operand is sequenced before the left, and the assignment is sequenced after both. So `i++` yields the old value, `+1` makes old+1, and the assignment writes it: `i == old + 1`.
- C++17 also fixed order for `<<`, `>>`, subscripting, `.`, `->`, and made a call's postfix expression sequenced before its arguments. Argument evaluation order **relative to each other** is still unspecified.

**94. Why is `size_t` unsigned, why is that considered a mistake, and what is `ssize`?**

- Unsigned because a size cannot be negative, and in the 16-bit era the extra bit doubled the addressable range.
- Considered a mistake because subtraction misbehaves: `a.size() - b.size()` wraps to an enormous number instead of going negative, mixed comparisons silently convert, and the compiler loses the overflow assumptions that make signed loops optimizable.
- `std::ssize(c)` (C++20) returns a signed size, enabling `for (auto i = 0; i < std::ssize(v); ++i)` with no conversion trap.

**95. What does strict aliasing let the compiler assume? Show a snippet that breaks it.**

- That two pointers of unrelated types never refer to the same object, so it can keep values in registers across a store through the other pointer and reorder freely.

```cpp
int i = 1;
float* f = reinterpret_cast<float*>(&i);
*f = 2.0f;
std::cout << i;    // may print 1: i was cached
```

- Exceptions: `char`, `unsigned char`, `std::byte`, and the object's own type or a signed/unsigned variant of it.

**96. Narrowing: `int x = 3.7;` vs `int y{3.7};` — and where else does brace init save you?**

- `int x = 3.7;` compiles and truncates to 3. `int y{3.7};` is a **compile error**: brace initialization forbids narrowing conversions.
- Brace init also saves you from the most vexing parse (`T obj{}` is a variable), from uninitialized members (`int x{}` is zero), and from lossy implicit conversions.
- Its own trap: an `initializer_list` constructor is preferred when one exists, so `vector<int> v{10}` gives one element and `v(10)` gives ten.

---

## 11. Concurrency & the Memory Model

**97. What is a data race, precisely, and what is the consequence?**

- Two threads access the **same memory location**, at least one access is a **write**, the accesses are **not atomic**, and they are **not ordered by happens-before**.
- Consequence: **undefined behavior for the whole program**, not merely a stale or torn value.
- Say this exactly. "You get a wrong value" understates it: the compiler is permitted to assume no races exist and optimize on that basis.

**98. `mutex`, `recursive_mutex`, `shared_mutex` — when would you justify the latter two?**

- `std::mutex`: plain mutual exclusion; UB if the owner locks again.
- `recursive_mutex`: the owner may lock repeatedly and must unlock as many times. Justified almost never; needing it usually means a public function calls another public function that also locks, and the fix is a private unlocked implementation function.
- `shared_mutex`: many readers or one writer. Justified only when read critical sections are **long** and writes rare, because a shared acquire is still an **atomic RMW on one counter**, so all readers contend on the same cache line and it is frequently **slower than a plain mutex**. Also no shared-to-unique upgrade, and prone to writer starvation.
- The HFT read-heavy answer is usually a **seqlock**: an odd/even version counter, wait-free readers who never write.

**99. `lock_guard` vs `unique_lock` vs `scoped_lock` — capabilities and costs.**

- `lock_guard`: locks in the constructor, unlocks in the destructor. Zero overhead, no early unlock, not movable.
- `unique_lock`: adds deferred locking, timed locking, early `unlock()`, move, try-lock. Required by `condition_variable::wait`. Costs one bool for the owns-lock flag and a branch in the destructor.
- `scoped_lock` (C++17): variadic, locks multiple mutexes with a deadlock-avoiding algorithm. The default choice; with one mutex it is equivalent to `lock_guard`.

**100. What is a deadlock? Give the four Coffman conditions and the two standard C++ tools.**

- Threads each holding a resource while waiting on one another, so none proceeds.
- Coffman, all four required: **mutual exclusion**, **hold and wait**, **no preemption**, **circular wait**.
- Tools: `std::scoped_lock(m1, m2)` / `std::lock(m1, m2)`, which acquire several mutexes atomically with respect to deadlock; and a documented global **lock ordering**, which removes circular wait.
- Weaker third option: `try_lock` with backoff.

**101. What does `cv.wait(lock, pred)` expand to? Why is the predicate loop non-negotiable? Spurious vs lost wakeup?**

- It expands to `while (!pred()) wait(lock);`. **The predicate form IS the while loop.**
- Needed for three reasons: a **spurious wakeup** (the OS may wake a waiter with no notify at all, a real consequence of futex and signal delivery), a **stolen** wakeup (another thread consumed the item first), and re-checking after reacquiring the lock.
- A **lost wakeup** is the opposite failure: the producer notifies while the consumer has checked the condition but not yet started waiting, so the notification hits nobody and the consumer waits forever.
- Fix: the state must be modified **while holding the same mutex** the waiter uses. Notifying without holding the lock is only safe when the state change itself was made under it.

**102. Explain relaxed, acquire, release, acq_rel, seq_cst, one sentence each.**

- **relaxed**: atomicity only, no ordering or visibility guarantees relative to other variables.
- **acquire** (loads): nothing in this thread can be reordered before it, and it sees everything released by the writer.
- **release** (stores): nothing in this thread can be reordered after it, and everything written before it becomes visible to an acquiring reader.
- **acq_rel** (RMW only): both, for the load and store halves.
- **seq_cst**: acquire/release plus a single total order over all seq_cst operations that all threads agree on. The default, because it is the only one where naive reasoning is sound.

**103. Write release/acquire message passing. Why does relaxed on the flag break it?**

```cpp
// Thread A
data = 42;                                        // ordinary write
flag.store(true, std::memory_order_release);      // publishes everything above

// Thread B
while (!flag.load(std::memory_order_acquire)) {}  // pairs with the release
assert(data == 42);                               // guaranteed
```

- The release store and the acquire load that reads it form a **synchronizes-with** edge, so `data = 42` happens-before the read.
- With `relaxed` there is no edge: the compiler or hardware may reorder, and B may see `flag == true` with stale `data`. The access to `data` is then a genuine data race, so UB, even though on x86 it would appear to work.
- **What it compiles to**: on x86 (TSO) an acquire load and a release store are **plain `mov`**, zero extra instructions; only a seq_cst store needs `xchg` or `mfence`. On ARM they are `ldar`/`stlr`. Note TSO still allows **store-load** reordering, so x86 is not "sequentially consistent".

**104. What does happens-before mean, and how do release/acquire pairs establish it?**

- A partial order combining **sequenced-before** within a thread and **synchronizes-with** between threads. If A happens-before B, A's effects are visible to B.
- A release store synchronizes-with the acquire load that reads its value, linking the two threads' sequenced-before chains into one order.
- Everything the writer did before the release is therefore visible to everything the reader does after the acquire.
- Absence of happens-before between conflicting accesses is exactly the definition of a data race.

**105. Is `volatile` a threading tool? What is it actually for?**

- **No.** No atomicity, no ordering, no cross-thread visibility.
- It tells the compiler the value may change outside the program's control, so it must not elide, cache, or reorder those accesses **with respect to other volatile accesses**.
- Real uses: memory-mapped hardware registers, variables touched by a signal handler (`volatile sig_atomic_t`), `setjmp`/`longjmp` locals.
- The confusion comes from **Java**, where `volatile` really does mean sequentially consistent atomic.

**106. Write a spinlock with correct memory orders, plus the two standard refinements.**

```cpp
class Spinlock {
    std::atomic<bool> locked_{false};
public:
    void lock() {
        for (;;) {
            if (!locked_.exchange(true, std::memory_order_acquire)) return;
            while (locked_.load(std::memory_order_relaxed))   // test-and-test-and-set
                _mm_pause();                                  // or yield()
        }
    }
    void unlock() { locked_.store(false, std::memory_order_release); }
};
```

- Acquire on the successful exchange, release on the store: that is what gives the critical section its ordering.
- **Test-and-test-and-set**: an `exchange` is a write, so spinning on it takes exclusive ownership of the line on every attempt and starves the holder. Spinning on a plain **relaxed load** lets the line stay shared.
- **`_mm_pause`**: hints the core to back off, saves power, and avoids a memory-order violation penalty on release.
- Use a spinlock only when the critical section is shorter than a context switch (a few hundred ns) and the holder cannot be preempted.

**107. `compare_exchange_weak` vs `strong`. Why does the CAS loop tolerate spurious failure, and what does `expected` become on failure?**

- Both compare against `expected` and store `desired` if equal; otherwise they **load the actual value into `expected`** and return false.
- `weak` may fail **spuriously** (return false even when values matched), because on LL/SC architectures (ARM, POWER) an intervening cache event breaks the reservation. `strong` retries internally to hide that.
- The **loop** tolerates it because it re-reads and retries anyway, and `weak` compiles to a tighter loop. So: `weak` inside a loop, `strong` for a single-shot attempt.
- `expected` being updated is what lets the next iteration compute from fresh state.

**108. What is the ABA problem? Give a lock-free stack scenario and two mitigations.**

- A thread reads A, is preempted, the value changes to B and back to A, and the CAS then succeeds even though the world changed.
- Stack: T1 reads `head = A`, plans `CAS(head, A, A->next == B)`, stalls. T2 pops A, pops B, pushes A back, so `head == A` again but `A->next` is now C. T1's CAS succeeds and sets `head = B`, which has been freed.
- **Tagged pointer**: a version counter incremented on every update, so ABA becomes A1/B/A2 and the CAS fails. Needs a double-width CAS (`cmpxchg16b`).
- **Hazard pointers or epoch-based reclamation**: a node is never freed while any thread may reference it.
- Third option: avoid node reuse entirely with an index-based pool plus generation counters.

**109. Sketch an SPSC ring buffer: who owns what, which indices are atomic, which orders, where does false sharing bite, and the full-vs-empty trick.**

- The producer owns `tail` (write index) and only reads `head`; the consumer owns `head` and only reads `tail`. Both indices atomic.
- Producer writes the slot, then **release**-stores `tail`. Consumer **acquire**-loads `tail`, which guarantees the slot contents are visible. Symmetrically the consumer's release store to `head` pairs with the producer's acquire load when checking for space. Each side's read of **its own** index can be relaxed.
- **False sharing**: if `head` and `tail` share a line, every producer write invalidates the consumer's copy and back. Pad each with `alignas(std::hardware_destructive_interference_size)`.
- After padding, what still ping-pongs is **each side's read of the other's index**. Fix with a **cached copy** of the other index, reloaded only when the buffer looks full or empty.
- **Full vs empty**: with plain wrapping indices `head == tail` is ambiguous. Either leave one slot unused (store at most capacity-1) or use free-running counters masked only when indexing, so `tail - head == capacity` means full.

**110. What is false sharing? How do you detect and fix it?**

- Two threads write distinct variables that occupy the same cache line. The line is the unit of coherence, so each write invalidates the other core's copy and the line **ping-pongs under MESI** even though nothing is logically shared.
- Detect: `perf c2c`, HITM counters (cache hits in a line modified on another core), or the symptom of throughput **dropping** as you add threads.
- Fix: pad or align each hot variable to its own line (`alignas(64)`); give each thread its own counter and sum on read; keep hot state in locals and write back once.

**111. `thread_local` — what is it, when initialized and destroyed, and one HFT use?**

- One instance per thread, with its own storage. Initialized on first use in that thread (or before first use for constant initialization), destroyed at thread exit in reverse order of construction.
- Uses: a per-thread pool or arena so allocation needs no locking; per-thread statistics counters avoiding the contended-atomic ping-pong; a per-thread scratch buffer for formatting; per-thread RNG state.
- Cost worth knowing: access may go through a TLS lookup, and in dynamically linked code that can be a function call rather than an offset from the thread pointer.

**112. Lock-free vs wait-free. Is a CAS retry loop wait-free?**

- **Lock-free**: system-wide progress guaranteed, at least one thread makes progress in bounded steps, though an individual thread may starve.
- **Wait-free**: **every** thread completes in a bounded number of its own steps.
- A CAS retry loop is **lock-free but not wait-free**: a given thread can lose the race indefinitely.
- Wait-free examples: a relaxed `fetch_add` counter, an atomic load or store, a seqlock reader.

---

## 12. Performance, Cache & Low-Latency

**113. From memory: latency in cycles for L1, L2, L3, main memory. Plus a branch mispredict and an uncontended mutex.**

- L1 hit ~4 cycles (~1 ns)
- L2 hit ~12 cycles (~4 ns)
- L3 hit ~40 cycles (~15 ns, worse across sockets)
- Main memory ~200-300 cycles (~80-100 ns)
- Branch mispredict ~15-20 cycles (~5 ns), the pipeline depth
- Uncontended mutex lock+unlock ~20-25 ns (two atomic RMWs, no syscall)
- Context for follow-ups: contended mutex with a futex sleep ~1-3 µs; syscall ~100-300 ns; context switch ~1-5 µs including cache damage; kernel bypass wire-to-app ~1-2 µs; datacenter round trip ~100 µs.
- The **shape** matters more than the digits: L1 to DRAM is ~100x, and one DRAM miss costs more than a hundred arithmetic instructions.

**114. What is a cache line? Why is iterating a `vector<T>` fast and a `list<T>` of the same data slow?**

- A cache line is the unit of transfer and coherence, **64 bytes** on x86-64.
- `vector<int>`: 16 ints per line, so 15 of every 16 accesses hit L1, and the hardware **prefetcher** recognizes the sequential stride and fetches ahead, hiding even the misses.
- `list<int>`: each node separately allocated (24-32 bytes with the payload), scattered by the allocator, so each step is a fresh line for one useful int; the next node's address is not known until the current one arrives (a **dependent load**, so misses serialize rather than overlap); and the prefetcher cannot predict the pattern.
- 10-50x on traversal for identical asymptotic complexity.

**115. Row-major 2D array: why is `[i][j]` vs `[j][i]` a massive difference? Estimate for a 10,000×10,000 `int` matrix.**

- Row-major means consecutive `j` are adjacent. `[i][j]` walks memory sequentially: one miss per 16 ints, prefetcher engaged.
- `[j][i]` strides by the row length (40,000 bytes here), so every access is a new line, a new page, and a likely **TLB miss**, and nothing useful of the fetched line is reused before eviction.
- 400 MB, far beyond any cache: expect roughly **10-30x**.
- Fix: loop interchange or tiling. This is the concrete version of "layout follows access pattern".

**116. What is branch prediction? Why does sorting an array speed up a branchy loop over it? What are `[[likely]]`/`[[unlikely]]`?**

- The CPU speculatively executes past a conditional using a predicted direction; a misprediction flushes the pipeline, ~15-20 cycles.
- On a **sorted** array a data-dependent branch like `if (v[i] > threshold)` takes the same direction for long runs, so the predictor is nearly always right; on shuffled data it approaches 50% wrong, which at ~15 cycles dominates the loop. Hence the famous 3-6x.
- `[[likely]]`/`[[unlikely]]` affect **code layout** (which side is on the fall-through path and stays in the instruction cache), not the dynamic predictor. Justified only for genuinely lopsided branches such as error paths, and only after measuring.

**117. Latency vs throughput of an instruction. Why do dependency chains matter more than instruction count?**

- **Latency**: cycles from issue to result available. **Throughput**: how many can start per cycle across the pipelined units.
- A multiply might be 5-cycle latency, 1-per-cycle throughput: ten independent multiplies take ~14 cycles, ten dependent ones take 50.
- Modern cores are wide and out-of-order, so they run out of **independent work** long before execution units. The critical path through the dependency graph sets the time.
- Practical: unroll with multiple accumulators, avoid long pointer chases (each load's address depends on the previous load), prefer reassociable computation.

**118. What does `inline` mean to the compiler vs the linker? What actually controls inlining, and what does a non-inlined call cost?**

- To the **linker**: an ODR relaxation. The entity may be defined identically in multiple TUs and the linker merges them, keeping external linkage. That is its only guaranteed meaning, and it is why definitions can live in headers.
- To the **compiler**: at most a weak hint. Functions defined in a class body or marked `constexpr` are implicitly `inline` in the ODR sense.
- What actually controls it: visibility of the definition (same TU, or LTO), size, call-site count, `always_inline`/`noinline`, PGO data.
- Cost of a non-inlined call: argument setup, call and return, stack frame, register spills, and far more important, the **lost optimizations**: no constant propagation into the body, no dead code elimination, no vectorization across the boundary, and the compiler must assume the callee clobbers memory.
- Related: **LTO** defers codegen so the optimizer sees the whole program (cross-TU inlining, devirtualization); **PGO** feeds real execution counts back so hot code is laid out contiguously and branches predicted correctly, typically 5-15% on branchy code.

**119. Why are exceptions often banned in hot paths? What is the real cost model of "zero-cost" exceptions?**

- **What is zero**: the non-throwing path adds no instructions. There is no runtime bookkeeping, only static unwind tables in a separate section.
- **What is not**: the tables inflate the binary (often 10-15%), pressuring the instruction cache; the possibility of a throw constrains the optimizer, since every call becomes a control-flow edge and objects must be destructible at each point; and the **throw itself costs microseconds**, allocating the exception object, walking DWARF unwind tables, running RTTI type matching per handler, and in many implementations taking a **global mutex** in the unwinder, which serializes all throwing threads.
- Model: free when not thrown, catastrophic and non-deterministic when thrown. Fine for exceptional conditions, unacceptable for control flow in a latency-bounded path.

**120. Why might `std::function` be banned in a hot path? Give at least two alternatives.**

- It **type-erases**, so a call is an indirect call through a stored pointer that cannot be inlined.
- It holds a small buffer (typically 16-32 bytes) with a **heap allocation at construction** when the callable does not fit. It is also larger than a pointer and its copy may allocate.
- Mechanism to state correctly: it **owns a pointer to an `Impl<F>`** implementing an abstract call interface. It does not "inherit from" the callable.
- Alternatives: (1) a **template parameter** for the callable, keeping the concrete type so the call fully inlines, at the cost of code bloat and header-only definitions; (2) `function_ref`-style non-owning views, a pointer pair with no allocation; (3) a function pointer plus `void*` context, no allocation but no inlining; (4) `std::variant` over a closed set plus `std::visit`, turning the indirect call into an inlinable switch; (5) CRTP or a concrete type.

**121. What is the cost of a syscall, and name three techniques trading systems use to avoid them on the fast path.**

- Roughly **100-300 ns**, and the register save/restore is the smaller half.
- The rest: **pipeline serialization** on the mode switch, **KPTI page-table switching and the TLB flush** that came with Meltdown, **cache and branch-predictor pollution** from kernel code, and the chance of being rescheduled on return.
- **Kernel bypass** (DPDK, Onload) so packets arrive in userspace with no `read`.
- **vDSO** for timing, so `clock_gettime` is a userspace read of a shared page, or `rdtsc` directly.
- **Preallocated and locked memory** (`mlock`, huge pages, arenas) so no page fault or `brk`/`mmap` happens on the path.
- Plus busy-polling instead of `epoll_wait`, and batching (`sendmmsg`, io_uring) so one syscall carries many operations.

**122. What is NUMA, and why does thread/memory pinning matter? What do `taskset`/`isolcpus` accomplish?**

- On a multi-socket machine each socket has its own memory controller. Remote memory crosses the interconnect at roughly **1.5-2x** the local latency with lower bandwidth.
- Pinning a thread to a core, with its memory allocated on that node (first-touch), keeps the working set local and stops the scheduler from migrating it, which would leave every cache line behind on the old core.
- `taskset` restricts which CPUs a process may run on. `isolcpus`/`nohz_full` remove cores from the general scheduler and timer tick entirely, so nothing else is ever scheduled there.
- Conceptually all of it is about making the memory hierarchy and cache state **predictable**, which is what p99.9 is made of.

**123. You measure p50 = 800 ns and p99.9 = 40 µs on the same path. Five plausible causes and how you'd investigate each.**

- **Page fault** on first touch or after reclaim, ~1-5 µs (more if major). `perf stat -e page-faults`, minor/major counts in `/proc`. Fix: prefault, `mlockall`, huge pages.
- **Context switch or preemption** by the scheduler, an interrupt, or a kernel thread, ~1-10 µs including cache damage. `perf sched`, voluntary/involuntary switch counts. Fix: `isolcpus`, IRQ affinity, `nohz_full`, RT priority.
- **Allocation hitting the slow path**, `malloc` going to `mmap` or taking an arena lock. Count allocations on the path (a counting `operator new` works), heap profiler. Fix: pools and arenas.
- **Cache and TLB misses from cold data**, e.g. a rarely used branch pulling in evicted code and data. `perf stat -e cache-misses,dTLB-load-misses`, `perf record` on the tail. Fix: layout, prefetch, huge pages, keep the hot path small.
- **Lock contention** where a normally uncontended mutex occasionally blocks into a futex, ~1-3 µs plus the wait. `perf lock`, `try_lock` failure counters.
- Also name: CPU frequency scaling and C-state exit, NUMA remote access after a migration, false sharing spikes under load, logging doing I/O, periodic work (stats, timers, stale-entry cleanup) landing inline.
- **Method** matters as much as the list: histogram not average, correlate the tail with timestamps, and establish whether it is **periodic** (a timer) or **load-correlated** (contention) before touching code.

---

## 13. Systems & OS Adjacent

**124. What happens between typing `./a.out` and `main` running?**

- The shell `fork`s; the child `execve`s the binary.
- The kernel tears down the old address space, parses the ELF headers, and maps the segments (text r-x, data rw-, bss zero-filled) as **lazy** mappings backed by the page cache.
- It sets up the stack with argv, envp, and the auxiliary vector.
- If dynamically linked, it loads the interpreter named in `PT_INTERP` (`ld.so`) and jumps **there**, not to the entry point.
- `ld.so` maps the needed shared libraries, performs **relocations** (filling the GOT; function symbols usually bound lazily through the PLT), runs `DT_INIT`/init_array constructors, then jumps to `_start`.
- `_start` sets up the C runtime, runs static constructors, calls `main`.
- First execution of any page faults it in from the page cache, which is why cold start is slow and why trading processes pre-touch and `mlock`.

**125. Static vs dynamic linking: costs at build, load, and run time. What is PLT/GOT overhead?**

- **Static**: bigger binary, slower link, but no load-time relocation, no interposition, direct calls that inline across the whole program under LTO, no runtime dependency.
- **Dynamic**: smaller binaries, shared text across processes, updatable without relinking; load time pays for mapping and relocating, and cross-library calls are indirect.
- The **GOT** is a table of resolved addresses. The **PLT** is a stub per external function that jumps through its GOT entry, initially pointing back into the resolver so the symbol binds on first call (lazy binding).
- Cost: an extra indirect jump and a load per cross-library call, plus loss of inlining across the boundary. `-fno-plt`, `-Bsymbolic`, prelinking reduce it; static linking removes it.

**126. What is virtual memory? Minor vs major page fault, and why does a trading process pre-fault and lock memory?**

- Each process sees its own virtual address space; the MMU translates through page tables, cached in the **TLB**.
- A **page fault** is a trap taken when the translation is missing or permissions disallow the access.
- **Minor**: the page is already in RAM (page cache, a copy-on-write share, or a fresh anonymous page needing zeroing), so the kernel just fixes the mapping, ~1 µs.
- **Major**: the page must come from disk or swap, ~100 µs to milliseconds.
- Pre-faulting (touching every page after allocation) plus `mlockall` guarantees no fault on the hot path; **huge pages** shrink the number of TLB entries the working set needs.
- Related: a **TLB shootdown** is when a mapping change forces an IPI to every core in the mask to invalidate and acknowledge, with the originator waiting. Hence: pin threads, never unmap on the hot path, use huge pages.

**127. TCP vs UDP in five sentences. Why do market data feeds use UDP multicast, and what does that force on the receiver?**

- TCP is connection-oriented, reliable, ordered, with retransmission, flow control (receiver window) and congestion control (sender window), delivering a **byte stream** with no message boundaries.
- UDP is connectionless datagrams: no retransmission, no ordering, no congestion control, **message boundaries preserved**, and it supports multicast.
- TCP's guarantees cost latency, most visibly **head-of-line blocking**, where one lost segment stalls delivery of everything behind it even though that data has arrived.
- Feeds use **UDP multicast** because one send reaches every subscriber at line rate with no per-subscriber state, and because a late message is worthless so retransmit-by-default is the wrong policy.
- The receiver must then handle: **gap detection** by sequence number, buffering of out-of-order messages, **retransmission requests** to a recovery server or a switch to the redundant B feed, A/B arbitration, and a drop policy when the queue backs up.
- MoldUDP64 specifically: session id plus sequence number per packet with a message count, so the receiver tracks the expected next sequence, detects a gap when a packet arrives ahead of it, and requests the missing range.

**128. What does `epoll` solve over `select`/`poll`? Edge vs level triggered? Where does busy-polling fit?**

- `select`/`poll` pass the entire interest set into the kernel on every call and the kernel scans all of it: **O(n)** per call plus the copy.
- `epoll` keeps the interest set in the kernel (`epoll_ctl`) and returns only the ready descriptors: **O(ready)**, no copy.
- **Level-triggered**: reports readiness while the condition holds, so a partial read still reports next time. Easier to get right.
- **Edge-triggered**: reports only on a transition, so you must drain until `EAGAIN` or you hang. Fewer wakeups.
- **Busy-polling** replaces the mechanism entirely: spin on `recv` or on the NIC ring rather than sleeping, burning a core to remove the wakeup and syscall path.

**129. What is kernel bypass conceptually? What does it eliminate from the receive path?**

- The NIC DMAs packets directly into userspace-mapped buffers and the application polls those rings itself. The kernel is out of the data path.
- **Eliminated**: the interrupt and softirq, `sk_buff` allocation and the copy into it, protocol stack traversal, the socket buffer, the copy to userspace, the syscall and mode switch, and the scheduler wakeup.
- **Taken on**: a userspace TCP/UDP stack, a permanently spinning core, loss of kernel tooling.
- Result: ~1-2 µs wire-to-application versus 10-20 µs through the kernel, and far tighter tails.
- Distinguish from DMA generally: the kernel path uses DMA too. Bypass removes the **software layers**, not the DMA.

**130. Process vs thread at the kernel level. What does a context switch cost, and what gets flushed?**

- On Linux both are **tasks**; the difference is what they share. `fork` copies the address space (copy-on-write) with its own fd table and signal handlers. `clone` with `CLONE_VM|CLONE_FILES|CLONE_FS|CLONE_SIGHAND` shares all of that, leaving only stack, registers, and TLS private.
- **Thread switch**: save and restore registers and stack pointer, re-enter the scheduler, ~1-2 µs. Nothing in the memory hierarchy is architecturally flushed, but the new thread's working set evicts the old one's from L1/L2, and that is the real cost.
- **Process switch**: additionally swaps page tables (CR3), which **flushes the TLB** unless PCIDs are in use. More expensive, and with KPTI even a syscall pays part of it.
- Related trap: `fork` in a multithreaded process copies **only the calling thread**, but copies mutexes in whatever state they were in. A mutex locked by another thread stays locked forever in the child. Only async-signal-safe calls are legal between `fork` and `exec`; use fork-then-exec immediately, `pthread_atfork`, or `posix_spawn`.

**131. What makes `std::cout << x` slow, and why do `printf`-style or hand-rolled formatting win?**

- Each `<<` goes through stream buffer machinery and **locale/facet** dispatch (`num_put` does a virtual call per numeric insertion), checks and updates format state, and by default `cout` is tied to `cin` and synchronized with C `stdio`.
- It is not inlinable, and each operator returns a stream reference, defeating batching.
- Alternatives win because: (1) no locale or virtual dispatch, the conversion is a direct integer-to-chars loop, and `std::to_chars` has no allocation and no locale at all; (2) formatting into a preallocated buffer with **one** write at the end instead of many small operations; (3) a hot-path logger can **defer entirely**, writing raw binary arguments plus a format id into a ring buffer and formatting on a background thread, dropping the in-path cost to a memcpy.
- Cheap fixes if you must use iostreams: `sync_with_stdio(false)` and `'\n'` instead of `std::endl` (which flushes).

---

## 14. Code Reading — state the output, or the bug

**132.**

```cpp
struct S { int x = 1; S() : x(2) { x = 3; } };
S s;
```

- `s.x == 3`.
- The default member initializer `= 1` is **ignored**, because the constructor's init list initializes `x` explicitly.
- So two writes happen: initialized to 2, then assigned 3 in the body. The `= 1` never executes.

**133.**

```cpp
auto f = [=]() { return counter_++; };   // inside a member function
```

- `counter_` is a member, so it is not captured directly. `[=]` captures **`this` by value** (the pointer, not the object), and `counter_` means `this->counter_`.
- **Lifetime trap**: the lambda holds a raw pointer, so if it outlives the object (posted to a thread pool, stored in a callback list) every call is a use-after-free.
- It also mutates the original object's member, surprising anyone who read `[=]` as "copies everything".
- C++20 deprecates implicit `this` capture via `[=]`. Write `[this]`, `[*this]` to copy the object, or `[c = counter_]` to copy the member.

**134.**

```cpp
const int& r = std::max(1, 2);
int use = r;
std::string&& s = std::string("hi") + "!";
s += "?";
```

- First: **UB**. `std::max` takes and returns `const T&`; the arguments materialize temporaries that die at the end of the full expression. Lifetime extension does **not** apply because `r` binds to the result of a **function call**, not directly to a temporary. `r` dangles.
- Second: **fine**. `s` binds directly to the temporary returned by `operator+`, so its lifetime is extended to `s`'s scope, and it is non-const so mutation is allowed.

**135.**

```cpp
std::map<std::string, int> m;
if (m["key"] > 0) { }
```

- `operator[]` **default-inserts** when the key is absent, so this creates `{"key", 0}` and compares 0 > 0.
- The map is now non-empty and later `size()` or iteration sees the phantom key.
- `operator[]` is a **write** operation. In a read path use `find`, `at`, `count`, or `contains`. It also does not compile on a `const map`, which is the compiler telling you the same thing.
- This pattern has now appeared in three of your sessions.

**136.**

```cpp
std::vector<std::string> v;
v.reserve(2);
auto& first = v.emplace_back("a");
v.emplace_back("b");
v.emplace_back("c");
std::cout << first;
```

- **UB**. The third `emplace_back` exceeds capacity, so the vector reallocates, moves the elements, and destroys the old buffer. `first` dangles: use-after-free.
- `emplace_back` returning a reference (C++17) makes this easy to write accidentally.
- Fix: reserve enough, or hold an **index** rather than a reference.

**137.**

```cpp
unsigned n = v.size();
for (unsigned i = 0; i <= n - 1; ++i) { }
```

- Explodes when `v` is **empty**: `n` is 0, `n - 1` wraps to `UINT_MAX`, and the loop runs ~4 billion iterations indexing out of bounds.
- Rule: never subtract from an unsigned value that can be zero. Write `i < n`, or use signed sizes.

**138.**

```cpp
struct B { virtual void f() { std::cout << "B"; } };
struct D : B { void f() override { std::cout << "D"; } };
void call(B b) { b.f(); }
D d; call(d);
```

- Prints **`B`**.
- `void call(B b)` copies only the `B` subobject out of `d`, and the copy's vptr is `B`'s, so `b.f()` dispatches to `B::f`.
- Fix: `void call(B& b)` or `const B&`.

**139.**

```cpp
std::shared_ptr<int> p(new int(5));
std::shared_ptr<int> q(p.get());
```

- Two **independent control blocks** now manage the same `int`, each with a strong count of 1.
- Whichever is destroyed first deletes the object; the second deletes it again: **double free**, with a use-after-free in between if the other is dereferenced.
- Rule: raw-pointer construction **takes ownership**, so a raw pointer may be given to at most one `shared_ptr`.
- Correct: `auto q = p;` to share, or `make_shared` from the start, or `enable_shared_from_this` when a member function needs one.

**140.**

```cpp
std::atomic<bool> ready{false};
int data = 0;
// T1: data = 42; ready.store(true, relaxed);
// T2: while (!ready.load(relaxed)) {}  std::cout << data;
```

- **Not guaranteed 42.**
- Relaxed gives atomicity for `ready` but establishes **no happens-before**, so there is no synchronizes-with edge and the write to `data` is unordered with respect to the read.
- Formally the access to `data` is then a **data race**, so the program has UB. Practically, the compiler may hoist or sink either operation, and on ARM or POWER the reader can see `ready == true` with stale `data`.
- x86 hardware would not reorder these two stores, but the **compiler** still might, so "works on my machine" proves nothing.
- Fix: `release` on the store, `acquire` on the load.

**141.**

```cpp
std::string_view sv = std::string("temporary");
std::cout << sv;
```

- **UB**. The temporary `std::string` dies at the end of that full expression, and `string_view` is a non-owning pointer plus length, so `sv` immediately dangles.
- Lifetime extension does not apply because `sv` is **not a reference**.
- Same shape as a function returning a `string_view` into a local. The most common `string_view` bug.

**142.**

```cpp
struct P { std::unique_ptr<int> u; };
P a{std::make_unique<int>(1)};
P b = a;
```

- Does **not** compile. `unique_ptr`'s copy constructor is deleted, so `P`'s implicit copy constructor is deleted too.
- The one-token change that makes it compile: `unique_ptr` → `shared_ptr`.
- **Is it wise?** Only if shared ownership is what you mean. If the intent is a single owner, the right change is `P b = std::move(a);`. Reaching for `shared_ptr` to silence a compile error is how ownership becomes unclear, refcount traffic appears on hot paths, and cycles start leaking.

---

## Cut from the original 156

Removed as low-yield for this round: delegating-constructor semantics, `std::launder`, UB during constant evaluation, `typename`/`template` disambiguation as a standalone question, CTAD, full vs partial specialization, compile-time factorial, `std::void_t` detection idiom, virtual-function inlining (folded into devirtualization), LTO/PGO (folded into inlining), physical vs logical const (folded into casts), `int x;` at namespace vs function scope (folded into initialization), function template vs instantiation (folded into headers), memory arena as a separate question (folded into pool allocator), and the `initializer_list` range-for puzzle.

## The repeat offenders

These have cost you marks in more than one session. Read these aloud rather than re-reading:

- **Q67** unordered_map references survive a rehash, only iterators die
- **Q135** `operator[]` inserting in a read path (third occurrence)
- **Q103** what acquire/release compiles to on x86 (plain `mov`, zero extra instructions)
- **Q130** `fork` with threads: one thread copied, mutexes copied locked
- **Q14 / Q53** `move_if_noexcept` and the strong guarantee behind it
- **Q35** const-qualifying accessors
- **Q77** only `shared_ptr` touches the strong count
- **Q90** `dynamic_cast`: pointer returns null, reference throws