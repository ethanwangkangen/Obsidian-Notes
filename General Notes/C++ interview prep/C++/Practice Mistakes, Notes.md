# Container operations
- Erasing
	- `std::erase_if(container, predicate)`
- Erasing while iterating
```cpp
for (auto it = v.begin(); it != v.end();) {
	if (invalid(*it)) it = v.erase(it);
	else it++;
}
```
- Erasing through iterator
	- `container.erase(it)`
- Initialising vector with a size
	- `v(5)` instead of `v{5}` 
- Map
	- Trying to do `[]` without default constructor
		- Instead, `m.insert{k, v}`
	- Forgetting that `[]` inserts
- Iterators
	- `rbegin()` is a REVERSE iterator! Not a normal one

# Reference lifetime/invalidation
- Careful with reference invalidation
	- Eg. object owns and `id` string, `id` string used after object deleted
# Common bugs, things to note
- Be careful with comparators
	- >, >=, and also **what you're comparing against** (size, capacity, 0)
	- Tie breakers, be careful
- Dropped constraints
	- **Read question carefully first**
- Uninitialised variables
- Accidentally casting to wrong thing
- Careful with typing
- Missing returns
- Timestamps - consider the current time!

- How to use `std::max_element(v.begin(), v.end(), worse)` where worse returns true if object1 worse than object 2
	- Take note of tie breakers!
- How to do `std::lower_bound properly`