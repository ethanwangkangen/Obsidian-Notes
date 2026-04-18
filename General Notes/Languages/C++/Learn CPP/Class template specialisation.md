Eg.
```cpp
template <typename T>
class Storage8 {
private:
    T m_array[8];

public:
    void set(int index, const T& value) {
        m_array[index] = value;
    }

    const T& get(int index) const {
        return m_array[index];
    }
};
```
- `Storage8<bool>` will end up being inefficient
	- All variables must have an address, and CPU can't address if less than a byte, so all variables must be at least one byte in size
	- But `bool` only needs 1 byte

# Class Template specialisation
```cpp
//... (the code on top)

template <> // no parameters
class Storage8<bool> { // specializing Storage8 for bool
private:
    std::uint8_t m_data{};

public:
    void set(int index, bool value) {
        // Figure out which bit we're setting/unsetting
        // This will put a 1 in the bit we're interested in turning on/off
        auto mask{ 1 << index };

        if (value)  // If we're setting a bit
            m_data |= mask;   // use bitwise-or to turn that bit on
        else  // if we're turning a bit off
            m_data &= ~mask;  // bitwise-and the inverse mask to turn that bit off
	}

    bool get(int index) {
        // Figure out which bit we're getting
        auto mask{ 1 << index };
        // bitwise-and to get the value of the bit we're interested in
        // Then implicit cast to boolean
        return (m_data & mask);
    }
};
```
- Specialised class template starts off with `template<>`
- When we instantiated any `Storage<T>` where T is not a bool, it will create a new version
	- But when T is bool, it will use this version here

## Specialising member functions
- What if we just want to customise a **member function** to be used for a specific type, but retain the rest of the class?
	- Define it with the specified type outside the class

```cpp
template<>
void Storage<double>::print() {
    std::cout << std::scientific << m_value << '\n';
}
```
- Explicit function specialisation
	- Not implicitly inline, make inline if put in header file

## Where to define class template specialisations
- Compiler must see full definition of both the non-specialised class
	- So just define it in the same header file, just below the non-specialised (generic) class
- If specialisation only required in a **single** translation unit
	- Then just put it there
	- Other translation units will not see definition hence continue to use the non-specialised version
- Be wary of putting specialisation in its own header file then include the header in any translation unit where it's needed
	- Dangerous to change behaviour based on presence/absence of a header file

# Partial template specialisation
- Allows us to specialise **classes** (but not individual functions!) where some (not all) of template parameters have been explicitly defined

## Function template overloading
(Not doing partial template specialisation yet)
- Overloading print(), making it take in a `StaticArray<char, size>`

```cpp
template<typename T, int size>
class StaticArray {
//....
}

// This only works for StaticArray of size 14
template<>
void print(const StaticArray<char, 14>& array) {
}


// This works for any size
template<int size>
void print(const StaticArray<char, size>& array) {
//...
}
```
- The first one will only work on StaticArrays of size 14, while the second will work for any length.

## Partial template specialisation
- However, the above example, print() was not a member function.

```cpp
// now a member function
template<typename T, int size>
void StaticArray<T, size>::print() const {..} 

// CANNOT do this!
template<int size>
void StaticArray<double, size>::print() const {..}
```
- Cannot partially specialise a function
- 