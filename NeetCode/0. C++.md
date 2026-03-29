```CPP
# vector
vector<int> v;  
vector<int> v2(5); // size 5, default 0  
vector<int> v3(5, 10); // size 5, all 10
v.push_back(x);  
v.pop_back();  
v.size();  
v.empty();  
v.clear();  
v[i]; // O(1)  
v.at(i); // bounds-checked
sort(v.begin(), v.end());  
sort(v.rbegin(), v.rend(); //descending

# hash set
unordered_set<int> seen = {};
s.insert(x);  
s.erase(x);  
s.count(x); // returns 1 or 0  
s.find(x); // iterator  
s.size();

# hash map
unordered_map<int, int> mp;
mp[key] = value;  
mp[key]++; // auto-initialises to 0
mp.count(key);
mp.find(key);
for (auto &p : mp) {
    cout << p.first << " " << p.second;
}

# map (ordered)
map<int, int> mp;

# set (ordered)
set<int> s;

# multiset (allow duplicated)
multiset<int> ms;
ms.insert(5);
ms.erase(ms.find(5));   // erase ONE occurrence

# queue
queue<int> q;
q.push(x);
q.front();
q.pop();
q.empty();

# stack
stack<int> st;
st.push(x);
st.top();
st.pop();
st.empty();

# priority queue (heap)
priority_queue<int> pq;
pq.push(x);
pq.top();
pq.pop();

# dequeue
deque<int> dq;
dq.push_front(x);
dq.push_back(x);
dq.pop_front();
dq.pop_back();

# pair
pair<int,int> p = {1,2};
p.first;
p.second;

# string
string s = "hello";  
s.length();  
s.size();  
s.substr(i, len);  
s.find("abc");  
stoi(s);  
to_string(x);
```

