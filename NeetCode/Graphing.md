# Templates
```CPP
# adjacency list 
unordered_map<int, vector<int>> graph;

for (auto &edge : edges) {
    graph[edge[0]].push_back(edge[1]);
}

# dfs
unordered_set<int> visited;  
  
void dfs(int node) {  
	if (visited.count(node)) return;  
		visited.insert(node);  
  
	for (int nei : graph[node]) {  
		dfs(nei);  
	}  
}


# bfs
queue<int> q;  
unordered_set<int> visited;  
  
q.push(start);  
visited.insert(start);  
  
while (!q.empty()) {  
	int node = q.front(); q.pop();  
  
	for (int nei : graph[node]) {  
		if (!visited.count(nei)) {  
			visited.insert(nei);  
			q.push(nei);  
		}  
	}  
}
```