# Parallelising 
```cpp
#pragma omp parallel for
for (int i = 0; i < n; i++)
    work(i);
```
- `#pragma omp parallel`
	- Divides outer loop (the immediately following loop) across threads
- `#pragma omp for`
	- Inside a parallel region, divides the loop
- Loop requirements
	- Integer/pointer counter
	- Comparison against a loop-invariant inboud
	- Increment by loop-invariant step
	- **No `break`, no early `return`**

# Sharing data
- Variables declared **outside** the region are **shared**, **inside** the region are **private**

| Clause            | Meaning                                                              |
| ----------------- | -------------------------------------------------------------------- |
| `shared(x)`       | one copy, all threads see it. Writes need protection.                |
| `private(x)`      | per-thread copy, **uninitialized** on entry, value discarded on exit |
| `firstprivate(x)` | private, initialized from the value before the region                |
| `lastprivate(x)`  | private, value from the sequentially-last iteration copied out       |
| `default(none)`   | forces you to name every variable. Use it.                           |
eg. 
```cpp
#pragma omp parallel for default(none) shared(a, b, result) private(j, k)
```
- Outer loop counter is implicitly private but inner loop counters are **not**
	- List them in `private(...)`

# Reduction
```cpp
#pragma omp parallel for reduction(+:sum)
for (int i = 0; i < n; i++)
    sum += a[i];
```
- Each thread gets **private accumulator** initialised to operator's identity
	- Combines them once at the end
- Operators: `+ - * & | ^ && || min max`
- Always **prefer this** over **atomic or critical** in a loop

# Scheduling
- `schedule(kind [, chunk])` on the `for`
- 
