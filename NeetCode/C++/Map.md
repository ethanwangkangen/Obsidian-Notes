map? unordered map?  

- Example from [[Arrays and Hashing]], Group Anagrams
	- Wanted to map `vector<int>` to strings
	- unordered_map is a hash table
		- Needs a hash function and equality operator
		- No hash provided for `vector<int>
	- map is a red-black tree
		- To insert/find a key, must compare keys with `<`
		- `vector<int>` DOES have a `<` defined