# Poisson distribution 
- Discrete
- Probabilities for **number of events** in a fixed time period
	- λ is mean number of events within a time interval
	- **Poisson** distribution
	- Average = λ
	- Variance = λ
- **Time between** events
	- **Exponential** distribution
	- Average = 1/λ, SD = 1/λ
![[Pasted image 20260912224616.png|446]]

- Cumulative calculations:
	- P(x<=2) = P(0) + P(1) + P(2)
- Remember to adjust λ based on the time interval

# Splitting and pooling
![[Pasted image 20260912230507.png|462]]

# Classification
![[Pasted image 20260912231044.png|464]]

# Solving Markov models
- Identify all possible system states to be modelled
- Determine transition rates between states
- Abstract linear equations and solve steady-state probabilities of being in each state
- Use steady-state probablilities to find other metrics of interest

# General birth-death process
- Only 2 transition types
	- Births - increase state variable by one
	- Deaths - decrease state variable by one
- Model - jobs arrive one at at ime, S is number of jobs in system

![[Pasted image 20260912231346.png|484]]
- Assumptions
	- Exponential inter-arrival time and service time
- Given i persons, 
	- Time until next arrival is exponentially distributed with mean 1/ λi
	- Time until next departure is exponentially distributed with mean 1/μi

![[Pasted image 20260912231545.png|588]]
![[Pasted image 20260912231647.png|582]]

General process formula:
- For state Sn-1:
![[Pasted image 20260912232118.png|578]]

# Traffic intensity - M/M/1 system
- ρ = λ/μ
- Product of arrival rate and mean service time
![[Pasted image 20260912232917.png|479]]
- P(N ≥ k) = Σ_{n≥k} (1 − ρ)ρⁿ = (1 − ρ)·ρᵏ/(1 − ρ) = ρᵏ
eg.
- Probability of buffer overflow if system has 12 buffers
	- = P(13 or more packets in system)
	- = ρ^13

eg.
![[Pasted image 20260912233420.png|583]]

# M/M/1 cheatsheet
 ![[Pasted image 20260912235747.png|372]]
![[Pasted image 20260912235810.png|407]]
Adjusting from one M/M/5 queue to 5 M/M/1 queues
- Divide λ by 5, ρ also changes based on that
- μ remains the same
# M/M/m system - more than one servers

![[Pasted image 20260912235834.png|430]]
![[Pasted image 20260912235909.png|436]]
# M/M/m/K system
- After K buffers are full, **all arrivals are lost**
![[Pasted image 20260912235517.png|417]]
![[Pasted image 20260912235927.png|369]]![[Pasted image 20260912235944.png|551]]