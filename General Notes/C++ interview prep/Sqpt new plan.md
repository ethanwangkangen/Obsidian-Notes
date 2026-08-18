Backlog
- OOD
	- AVL tree/map (planned)
	- Order book recap (planned)
	- Vector (planned)
	- Range module (planned)
	- Thread pool recap (planned)
	- SPSC queues recap (planned)
	- LRU cache recap (planned)
	- File system recap (planned)
	- Pool allocator (lock free version) (planned)
	- std::function wrapper
	- Hashmap (planned)
- Currency exchange - djikstra/bfs
	- Leetcode review
- C++
	- Banks
	- Getcracked checklists
	- Review modern C++ tools, features
- OS
	- Banks
	- Ostep chapters(?)
- CA
	- Banks
	- Getcracked checklist
- Posix/Linux


# Aug 17 Mon
- Implementation
	- Hashmap (2h)
	- Ranges (30 min)
	- Vector recap (30 min)
- Leetcode recap (all) (2h)
- C++ all banks and checklists (2h)

# Aug 18 Tue
- Implementation
	- std::function (?) (2h)
	- std::function wrapper? (30 min)
	- SPSC caching + recap (30 min) - DONE
	- Map blank review (30 min)
	- Ranges - DONE
- Blockgraph and moldcast recap (resume)
	- recvmmsg (30 min)
- OS bank, networking recap (2h)
- C++ modern tools, features you've used.. (30 min)

# Aug 19 Wed
- Implementation
	- Pool allocator (2h)
- Review everything
	- All implementation stuff (2h)
- C++
	- Compilation deep (15 min)
- Comp Arch Bank (2h)
- Posix and linux recap + cheatsheet  (1h)


# Aug 20 Thur
- Behavioural prep
	- Why squarepoint, resume stuff, etc.
- Interview day
  
  ```cpp
  struct TimeEntry {
	  string key;
	  string value;
	  int timestamp;
  };
  class TimeMap {
  public:
	  TimeMap() {};
	  void set(string key, string value, int timestamp) {
		  entries[key].emplace_back(key, value, timestamp);
	  }
	  
	  string get(string key, int timestamp) {
		  auto p = entries.find(key);
		  if (p == entries.end()) return "";
		  // largest timestamp <= timestamp.
		  // timetamp > timestamp, then subtract one
		  auto& v = p->second;
		  auto it = std::upper_bound(
			  v.begin(), 
			  v.end(),
			  timestamp,
			  [&](const auto& timestamp, const auto& elem){
				  return elem.timestamp > timestamp;
			  }
			);
		  if (it == v.begin()) return "";
		  return (--it)->value;
	  }
  
  private:
	  std::unordered_map<std::string, std::vector<TimeEntry>> entries;
  };
  ```