- Things to note
	- How to use std::max_element (write a worse() function)
	- Erasing
		- Erase **at the end**
		- Remember to erase the bid if it's complete, **and the Pair<Price, Vector> if it's empty!**
		- std::erase_if(v, pred)
			- But if we have the iterator, can just do v.erase(it)
```cpp
std::map<Price, std::vector<Order>> bids;
std::map<Price, std::vector<Order>> asks;
```