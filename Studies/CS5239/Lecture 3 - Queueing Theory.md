# Stochastic process
- Random variables, indexed by time
	- Discrete or continuous
- Important groups
	- Markov process ->**Future** states of process **independent of past**
		- Birth-death process (subset of Markov) -> discrete-space markov, transitions restricted to neighbor states
			- Poisson (Subset of birth-death) ->Inter-arrival times **independently** and **exponentially** distributed
- eg. N(t) - number of jobs at CPU time t


# Markov process
- Probability of each event depends only on curent event
- Don't care about the past

# Queueing Theory
- Open vs Closed
- Open
	- New jobs arrive and depart
	- Can have finite or infinite buffer
		- Finite buffer: jobs that arrive after full buffer are **dropped**
	- Probabilistic/ non-probabilistic routing
- Closed: jobs stay within system, recycled



![[Pasted image 20260912220020.png|323]]
- Source
	- Can be finite or infinite
- Arrivel process
	- Diff between 2 consecurive arrivals of customers
- Service time distribution
	- Time each customer spends at service center
- Number of servers
	- Can be 1 or **m**
- Service discipline/queue discipline
	- eg. FCFS, RR

# Kendall notation
- A/S/m/K/N/Sd
	- *A: inter-**Arrival** time distribution*
	- *S: **S**ervice time distribution*
	- *m: number of servers*
	- K: system capacity
		- Number of places in queue + number of servers (m)
		- Queue size = K-m
	- N: number of customers in source/ population size
	- SD: **S**ervice **D**iscipline 
		- eg. FCFS
- Short hand
	- A/S/m (just the first 3)
	- Infinite queue
	- Infinite source
	- SD is FCFS

- Inter-arrival time and Service time distribution denoted by
	- D: deterministic (**constant**)
	- M: exponential
	- Ek: Erlang-K
	- Hk: Hyper-exponential
	- G: general (any probability distribution)
	
![[Pasted image 20260912220946.png]]
![[Pasted image 20260912221004.png]]

- λ = mean arrival rate
- μ = mean service rate per server
	- = `1/ E[s]` for one server
- w = waiting time in queue
- s = service time per job
- r = response time 
	- = w + s

- `E[w]` = mean waiting time
- `E[s]` = mean service time
- `E[r]` = mean response time in system
	- = `E[w]` + `E[s]`

- `Nq `= number of jobs in queue
- `Ns` = number of jobs receiving service
- `N`= total jobs in system
	- = Nq + Ns
- `E[N]` = `E[Nq`] + `E[Ns]`

>each worker can process on average 10 tasks per second 
> == arrival rate per worker λ = 10, since one tasks enters as one leaves
# Little's law
Stable:
 - System stable if mean arrival rate less than mean service rate
	 - λ < mμ
	 - mean arrival rate (λ) < number of servers(m) * mean service rate(μ)
- Finite population & finite buffer system => stable
- Condition necessary for bounded queue but does not guarantee acceptable response time
```
Little's law: 
E[n] = λE[r] - mean number of jobs in system - mean arrival rate * mean response time

E[Nq] = λE[w] - mean number of jobs in queue = mean arrival rate * mean waiting time

E[Ns] = λE[s] - mean number of jobs receiving service = mean arrival rate * mean job service time
```
- Little's law apply to any stable system, as long as number of jobs entering system = number of jobs completing service

# Traffic Intensity and Utilisation
- Traffic Intensity ρ
	- = λ/μ
	- Percent measure of load on overall system
		- If < 1, system is steady
		- If >=1, system cannot keep up with demand
	- For one server
		- When jobs are not lost and system is in steady state
		- = λ`E[s]` 
		- = `E[Ns]` from Little's law
- Utilisation U
	- Proportion of time the server is busy
		- Max out at 100% (saturated)
		- If jobs never lost, U = ρ
		- If jobs can be lost, U <= ρ