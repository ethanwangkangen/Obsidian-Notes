Previously, 
# Systematic search
- Typically complete and optimal
- But intractable, not solvable in reasonable amount of time

- State: corresponds to **actual/specific** situation or configuration of problem domain

# Local search
- Traverse state space by considering only the best locally-reachable state
- **FORGET** other neighbouring states after each choice, instead of adding to frontier
- Typically incomplete and suboptimal
- Solution is the **final state after a certain number of steps**
	- **NOT the path taken to reach it**
- Any-time property
	- Longer runtime, often better solution

- State: represend different configurations or candidate solutions, **may or may not map to actual problem state**
- Successor function: function that generates neighboring states by applying modifications to current state

Path finding example
- States are the **paths** form A to B
- Then successor function is replace a sub-path between any 2 cities with other valid paths

Evaluation function
- Test quality of a state
- eg. n-queens: number of safe queens

## Hill climbing / greedy local search /steepest ascent esearch
- Just keep finding the highest eval successor state of current state
- If can't find any, return state as answer

- Global vs Local Maximum
	- Overal highest points vs point that is higher than immediate neighbours
- Shoulder
	- Flat region, hard for algo to progress

# Adverserial search
- Competitive multi-agent problems
	- Opposing goals. 
	- Adverserial games
		- Compete againste each other 
		- eg. Zero sum game
- Terminal state
	- State where game ends
- Utility function
	- Output the value of a state from perspective of ***OUR*** agent (eg. agent A)
		- Wants to maximise this
	- eg. Outcome is win loss or draw, assign utility values of +1, -1 and 0
	- Utility of non-terminal states is typically not known in advance, must be computed

How to decide?
- A must   think about the moves of player B too

Expand function
- For each action, compute: next_state=transition(state, action)
- Returns a set of (action, next_state pairs)

Terminal function
- Returns True if state is terminal

Utility function
- Returns a score or state from pov of A

Minimax(state):
- Return action that has max_value(state)
	- Assuming we have the values for each new state

Max_value(state):
- If terminal(state); return utility(state)
- for next_state in expand(state):
	- v=max(v, min value for A in next_state)
- return v   

Min_value(state):
- If is_terminal(state): return utility(state)
- for next_state in expand(state)
	- - v=min(v, max value for A in next_state)

Minimax analysis
- Time completxity: exponential, max depth O(b^M), b is branching factor, max depth m
- Space complexity: polynomial
- Complete: yes if tree finite
- Optimal: yes against optimal opponent
	- Compute optimal strategy for both players
	- Nash equilibrium: Player B cannot reduce player A's utility by playing sub optimally
- Problem: minimax explores entire tree, but sometimes unnecessary
	- eg. A has option 1 and 2. Option 1 gives A +10, Option 2 lets B choose something that makes A -20. 
		- Already we know A should choose option 1, don't need to evaluate the rest of B's choices
	- ie. Pruning

## Alpha-Beta pruning
- At A's turn, if action leads to better oucome for player A, higher than the lowest value of A so far, then the rest of actions irrelevant
- ![[2026-02-09_19-14.png]]
- Ordering becomes important now

- Cutoff
	- Don't explore full game tree 
- ![[Pasted image 20260209191949.png]]
- ![[Pasted image 20260209192124.png]]