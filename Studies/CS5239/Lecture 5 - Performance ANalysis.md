# Operational laws
- **Mean** values of performance metrics in queueing networks
- **No assumption** on distribution of service times/inter-arrival times
- Operational = directly measured
- Operational quantities directly measured during a finite observational period
- Operational laws are relationships among operational quantities 

![[Pasted image 20260913203413.png|417]]

- **A**rrivals
- **B**usy time
- **C**ompleted jobs

![[Pasted image 20260913203659.png|437]]

# Utilisation law
- Assumption: Job flow balance 
	- Job arrivals = Completions (**A** = **C**) / (**Xi = λi**)
- ![[Pasted image 20260913204006.png|189]]
- Device with highest utilisation = **bottleneck device**

# Visit ratio
- Jobs can visit a device multiple times
	- Each makes **Vi** requests for the i'th device
- Number of jobs Ci visiting i'th device
	- Ci = C * Vi
	- Vi = Ci / C
		- Vi = ratio of visits to the i'th device and outside link (jobs completed)
		- For one job, Vi = Ci/1 = number of visits per job to device i
	- where **C** is number of job completed

# Forced Flow Law
- Xi = X * Vi
	- X is system throughput
		- Open network - number of jobs leaving system per unit time
		- Closed network - number of jobs completing service
- **C** is number of jobs leaving system in T
	- X = C/T
- When job flow is balanced at device i, 
	- Vi = Ci/C (above, visit ratio)
- Then throughput of device is
	- Xi = Ci/T * C/C
	- Xi = X * Vi

![[Pasted image 20260913204925.png|348]]
![[Pasted image 20260913205005.png|563]]

# Little's law
- Assume job flow balance (**Xi = λi**)
- To get **device response time R** from **Queue length Q**
- If Qi is average queue size (**including jobs in service**) and Ri is average customer response time at queue i,
	- Qi = λiRi = XiRi
		- get Xi from Forced Flow Law


# General response time law
- ![[Pasted image 20260913210114.png|107]]


# Interactive vs Batch system
![[Pasted image 20260913210237.png|410]]

# Interactive Response Time Law
- User generates request, waits for response
	- On getting response, wait (think time), generate new request
- R = system response time
- Z = average think time
- For N users,
	- Number of jobs completed = N * T / (R + Z)
	- System throughput X = N/(R+Z)

![[Pasted image 20260913210615.png|391]]

# Summary
![[Pasted image 20260913210642.png|606]]

# Performance bounds
- Asymptotic bounds (optimistic and pessimistic)
- Balanced bounds (when no bottleneck)
- Note
	- No bounds for open system, beyond saturaion point, R continues to get worse as load increases

## Asymptotic bounds
- Assumption:
	- Service demand of customer does not depend on how many other customers are in system, or at which service center they are located
- Forced Flow Law
	- Device utilisation proportional to total service demand
		- Ui = XiSi = λiSi = XDi
	- Device with highest total service demand Di has highest utilisation and is the bottleneck device
	- D is total sum of service demands on all devices except terminals


![[Pasted image 20260913211332.png|509]]
![[Pasted image 20260913211521.png|512]]

# Balanced system bounds
![[Pasted image 20260913211750.png|536]]