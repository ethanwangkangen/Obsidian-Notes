# OS Interview Master List — C++ / HFT-Adjacent Graduate Role

Each answer is calibrated to ~20–30 seconds spoken (60–90 words). HFT-relevant hooks are marked **[HFT]** — dropping one of these naturally is how you signal you know _why_ the OS matters for latency, not just what it does.

---

## 1. Processes & Lifecycle

**Q1.1 — What does `fork()` do, and what does it return?** `fork()` creates a child process that is a near-copy of the parent: same code, a copy of the address space, copies of the file descriptor table. It returns twice — **0 in the child, the child's PID in the parent, -1 on failure** — and you branch on that return value to run different code in each process. The copy is lazy: pages are shared copy-on-write, so fork is cheap even for a huge process.
```cpp
pid_t pid = fork();
if (pid < 0)      { /* error */ }
else if (pid == 0){ /* child code */ }
else              { /* parent code, pid = child's PID */ }
```

**Q1.2 — What does `exec()` do? Why is it separate from `fork()`?** `exec` replaces the current process image with a new program — same PID, same open FDs by default, but new code, data, heap, and stack. On success it never returns. Unix splits creation (`fork`) from loading (`exec`) so the child can be _configured between the two_: redirect file descriptors, wire up pipes, drop privileges, change directory — then exec. That gap is exactly how shells implement redirection and pipelines.

**Q1.3 — What do `wait()` / `waitpid()` do?** The parent blocks until a child changes state, and collects the child's exit status. Crucially, this is also how the kernel gets permission to free the child's process-table entry — until someone waits, the dead child stays around as a zombie. `waitpid` lets you target a specific child and pass `WNOHANG` to poll without blocking. `WIFEXITED`/`WEXITSTATUS` decode the status.

**Q1.4 — What is a zombie process? An orphan?** A **zombie** is a child that has exited but whose parent hasn't called `wait` yet — the kernel keeps its PID and exit status in the process table so the parent can still collect them. It consumes a process-table slot, not real memory. Fix: reap it (`wait`, or a `SIGCHLD` handler). An **orphan** is a live child whose parent died first; it's re-parented to `init`/`systemd`, which reaps it on exit — so orphans don't become permanent zombies.
Reason - must hang around so exit status can be found.g

**Q1.5 — What is copy-on-write?** After fork, parent and child share the same physical pages, all marked read-only in both page tables. When either process writes, the CPU faults, and the kernel copies just that one page and makes it writable. So fork copies page tables, not memory — a multi-GB process forks in microseconds, and pages that are never written are never duplicated.

**Q1.6 — How many lines does `fork(); fork(); printf("hi\n");` print?** Four. Each `fork` doubles the process count, so _n_ sequential forks give 2ⁿ processes, each printing once. Same logic for a loop: `for (int i = 0; i < 3; i++) fork();` ends with 8 processes.

**Q1.7 — Classic gotcha: `printf("A"); fork(); printf("B");` — what can go wrong with the output?** If stdout is **fully buffered** (e.g., redirected to a file), "A" may still be sitting in the userspace buffer at fork time. The buffer is part of the address space, so it's duplicated — both processes flush it at exit, and "A" prints **twice**. On a terminal, stdout is line-buffered and the `\n`-less "A" can still be buffered. Defense in prediction problems: check for `\n` or `fflush` before the fork.

**Q1.8 — Predict the output:**

```c
int x = 5;
pid_t pid = fork();
if (pid == 0) { x += 5; }
else          { x -= 5; wait(NULL); }
printf("%d\n", x);
```

Two lines: `10` (child) and `0` (parent), order not guaranteed unless the parent's `wait` happens before its own printf — here it does wait first, so the child's `10` prints before the parent's `0`. The key point to say out loud: **after fork the variables are independent copies** — the child's `+=` never affects the parent's `x`.


---
Learning points
- Each process has a page table, mapping virtual memory to main memory
	- The actual memory is stored in the **pages** themselves.
	- The page table just stores the mappings
- When fork()
	- New process created = new page table
	- The page table entries are then **copied**. The **pages** themselves are **not copied**.
		- Only copy pages when the new process **writes**
		- And of course, code pages are never written to so always shared.
			- eg. there's only every one copy of libc
- Demand paging
	- `malloc(1GB)` doesnt actually put anyting in the page table.
	- The first time you touch a byte there - CPU walks page table, finds no entry - page fault
	- Kernel checks records - is this address in a region I promised? Yes -> allocate one physical page, zero it, install page table entry and continue
		- One byte written, own one 4kb page
- The key point
	- A process' memory is a promise, not a possession. The address space is a description of what the process is entitled to see; physical pages are attached lazily, shared covertly etc.
- VIRT vs RSS
	- VIRT = sum of promises
	- RSS = materialized physical pages
	- free() doesn't drop RSS
		- 2 layers of promises - free() returns chunks to userspace allocator's pool for reuse, allocator rarely returns pages to kernel
	- 




## 2. Threads vs Processes

**Q2.1 — What do threads share, and what is per-thread?** Threads of one process **share** the address space — code, globals, heap — plus the file descriptor table, signal dispositions, and current directory. **Private** to each thread: its stack, register state and program counter, thread-local storage, errno, and its signal mask. Processes share nothing by default and must communicate through explicit IPC.

**Q2.2 — Why is creating a thread cheaper than creating a process?** A new process needs its own address space: page tables to set up, COW bookkeeping on every mapped region. A new thread needs only a stack and a kernel task struct — the address space already exists. Rough orders: thread creation is ~10µs; fork scales with the size of the mappings.

**Q2.3 — Why is a thread context switch cheaper than a process context switch?** Both save and restore registers. But a process switch also swaps the page-table base register (CR3 on x86), which **invalidates TLB entries** — every memory access after the switch pays page-walk costs until the TLB rewarms, and caches are cold too. Threads of the same process keep the same page tables, so no TLB flush. **[HFT]** The direct switch cost is ~1–2µs; the indirect cache/TLB pollution usually costs more — which is why latency-critical threads get pinned to isolated cores so they're never switched out at all.

**Q2.4 — When would you pick processes over threads?** When you want **isolation**: a segfault or heap corruption in one process can't corrupt another; separate address spaces limit blast radius and improve security; and separate processes are easier to restart independently or move across machines. Threads win when you need cheap, fast shared state. **[HFT]** Common trading-system shape: separate processes per component (gateway, strategy, risk) connected by shared-memory queues — isolation of processes, speed of shared memory.

**Q2.5 — What actually happens during a context switch?** Kernel saves the current task's registers, PC, and stack pointer into its task struct; picks the next task via the scheduler; restores that task's state; for a different process, also loads its page-table base. Triggered by the timer interrupt, the task blocking (I/O, lock), or a higher-priority task waking.

---
Learning points
- Process switch - new page table, require TLB flush 
- Thread switch - no need TLB flush since same address space
	- Thread has its own stack - just carves out a portion of the address space for its own stack. That means there's a limit to the number of threads.
	- But on 64 bit systems, the constraint is actually the kernel stack
		- Kernel stack is unpromiseable, needs to actaully be there (unlike user stack, which is demand-paged and growable)
- Can you have a pointer into another thread's stack?
	- Yes, as long as that stack's frame is alive
- So process vs thread switch
	- Both: save state into task's kernel stack/thread_struct, run scheduler, restore task's next state. swap page table root
	- For proess: historically, TLB flushed (but now due to ASID tagging, TLB entries are stamped with address space's ID so switching page tables just means new process' translations hit its own tagged entries (could still be warm))



## 3. Scheduling

**Q3.1 — Preemptive vs cooperative scheduling?** Preemptive: the kernel can take the CPU away from a running task — on a timer interrupt, or when a higher-priority task becomes runnable. Cooperative: tasks keep the CPU until they voluntarily yield. Modern general-purpose OSes are preemptive so one spinning task can't hang the machine.

**Q3.2 — Linux CFS in one or two lines?** CFS — the Completely Fair Scheduler — tracks each task's **virtual runtime** and always runs the runnable task that has had the least, keeping tasks sorted in a red-black tree keyed on vruntime. `nice` values scale how fast vruntime accumulates, so higher priority means your clock ticks slower and you get more CPU. (Bonus point: since kernel 6.6 it's been replaced by EEVDF, a latency-aware refinement of the same fair-share idea.)

**Q3.3 — What are priorities / nice values?** Normal tasks have `nice` from -20 (favored) to +19 (deprioritized), which weights their CPU share under CFS. Above all normal tasks sit the **real-time policies** `SCHED_FIFO` and `SCHED_RR`, which always preempt normal tasks. **[HFT]** Standard tuning: hot threads run `SCHED_FIFO` on cores isolated from the scheduler (`isolcpus`/cpusets), pinned with `taskset`/affinity, so nothing ever preempts the hot path.

**Q3.4 — What events cause the scheduler to run?** The timer tick expiring a timeslice; the running task blocking on I/O, a lock, or sleep; a higher-priority task waking up; or the task exiting/yielding. On return from any interrupt or syscall the kernel checks a "need resched" flag and switches if set.

---

## 4. Syscalls & Privilege

**Q4.1 — What's the difference between user mode and kernel mode?** They're hardware privilege levels. In user mode the CPU refuses privileged instructions — you can't touch page tables, device registers, or interrupt state. Kernel mode can do everything. This is what makes process isolation _enforceable_: a user program physically cannot bypass the kernel, and the only way in is through controlled entry points — syscalls, exceptions, interrupts.

**Q4.2 — Walk me through what happens when you call `read()`.** The libc wrapper puts the syscall number and arguments into registers and executes the `syscall` instruction. The CPU switches to kernel mode and jumps to a fixed kernel entry point — user code can't choose where it lands. The kernel validates the arguments (is that fd open? is the buffer pointer valid user memory?), runs the handler, puts the result in a register, and executes the return instruction back to user mode.

**Q4.3 — Why are syscalls expensive?** Three layers. The mode switch itself — saving state, entering the kernel — costs on the order of 100+ ns versus ~1 ns for a normal call. Then indirect cost: the kernel's code and data pollute your caches, TLB, and branch predictors. Then mitigations — post-Meltdown KPTI switches page tables on entry, adding TLB cost. **[HFT]** This is why hot paths batch or avoid syscalls entirely: kernel-bypass networking (DPDK, Onload) reads NIC ring buffers from user space, and `io_uring` batches I/O submissions.

**Q4.4 — Trap vs interrupt vs exception?** An **interrupt** is asynchronous, from hardware — a NIC, the timer. An **exception** is synchronous, caused by the current instruction — page fault, divide-by-zero. A **trap** is a deliberate synchronous entry into the kernel — the syscall instruction. All three enter the kernel through its vector table; the difference is what triggered them and whether execution resumes at the same or the next instruction.

**Q4.5 — Why is `clock_gettime()` so much faster than other syscalls?** Because of the **vDSO**: the kernel maps a small shared library plus a data page into every process, so time queries read a kernel-updated timestamp entirely in user space — no mode switch at all. **[HFT]** Relevant because timestamping is everywhere on a trading path; the vDSO (or raw `rdtsc`) is why it's affordable.

---

## 5. Virtual Memory

**Q5.1 — Why does virtual memory exist?** Four reasons. **Isolation** — each process gets its own address space and physically can't read another's. **Abstraction** — every process sees a clean contiguous space regardless of how fragmented physical RAM is. **Overcommit** — you can map more than physical RAM and fill it lazily via demand paging. **Sharing** — the same physical page can appear in many processes (shared libraries, COW after fork), with per-page permissions.

**Q5.2 — Why multi-level page tables instead of one flat table?** A flat table for a 48-bit space at 4KB pages would need billions of entries per process — almost all for unmapped addresses. A multi-level tree (4 levels on x86-64, optionally 5) allocates interior nodes only for regions actually in use, so a sparse address space costs almost nothing. The price is that a translation walks 4 levels — 4 memory accesses — which is why the TLB exists.

**Q5.3 — What is the TLB?** A small, very fast cache inside the MMU holding recent virtual-to-physical translations. Hit: translation is essentially free. Miss: hardware walks the page tables — ~4 dependent memory accesses. **[HFT]** Two standard tunings: **huge pages** (2MB/1GB) so each TLB entry covers far more memory, and avoiding process context switches, which invalidate TLB entries.

**Q5.4 — What happens on a page fault?** The MMU can't translate the address (or permissions fail), so the CPU raises an exception into the kernel. The kernel checks whether the address falls in a valid mapping. If yes, it repairs it — allocate and zero a fresh page (first touch), read it back from disk (swapped/file-backed), or copy it (COW write) — then restarts the faulting instruction transparently. If the address is invalid, the process gets **SIGSEGV**. Terminology: a **minor** fault needs no disk I/O; a **major** fault does.

**Q5.5 — What is demand paging?** Mapping memory lazily. `mmap` or a growing heap only reserves address space; physical frames are allocated on first touch, via a page fault per page. So `malloc(1GB)` is nearly instant and RAM is only consumed as pages are actually used. **[HFT]** The flip side: first touch on the hot path costs a fault — so latency-critical processes pre-fault their memory at startup and `mlock` it so it can never be paged out.

**Q5.6 — What is swap?** Disk space the kernel uses to evict cold pages when RAM is short; touching a swapped-out page triggers a major fault that reads it back — milliseconds instead of nanoseconds. **[HFT]** Production trading boxes disable swap or `mlockall`, because an unpredictable millisecond stall is worse than an OOM you can see.

---

## 6. Process Memory Layout

**Q6.1 — Describe the memory layout of a process.** From low addresses up: the **text** segment (machine code, read-only + execute, includes `.rodata` for string literals and constants); **data** (initialized globals/statics); **bss** (zero-initialized globals — occupies no space in the binary); the **heap**, growing upward; the **mmap region** in the middle (shared libraries, thread stacks, large allocations); and the **main stack** near the top, growing downward. Kernel address space sits above all of it, inaccessible from user mode.

**Q6.2 — Stack vs heap: trade-offs?** Stack allocation is a pointer bump — effectively free — with automatic lifetime tied to scope, and it's cache-hot because you keep reusing the same lines. But it's small (default ~8MB) and an object can't outlive its function. Heap gives arbitrary lifetime and size, at the cost of allocator work, possible syscalls and page faults, fragmentation over time, and manual/RAII lifetime management. **[HFT]** Hot-path rule of thumb: stack and preallocated pools; no heap allocation per message.

**Q6.3 — What is a stack overflow, mechanically?** The stack grows into the **guard page** below its limit — an unmapped page placed there deliberately — so the overflowing access page-faults with no valid mapping and the process gets SIGSEGV. Typical causes: unbounded recursion or huge local arrays. That's also why "stack overflow" usually presents as a segfault.

**Q6.4 — Where do these live: string literal, `static int` inside a function, `const int` global, uninitialized global?** String literal → `.rodata` (writing to it is UB, and it's mapped read-only, so it also segfaults). Function-`static` → data if initialized, bss if zero. `const` global → `.rodata`. Uninitialized global → bss, and it _is_ zero-initialized — "uninitialized" only means no explicit initializer.

---

## 7. Heap Internals

**Q7.1 — What does `malloc` actually do? Is it a syscall?** Not a syscall — it's library code (glibc's ptmalloc, or jemalloc/tcmalloc). The allocator holds arenas of memory it already got from the kernel and serves small requests from free lists binned by size, carving chunks and storing size headers next to them. Only when it runs out does it call the kernel — `brk` to grow the heap or `mmap` for a new region. `free` returns the chunk to a bin and coalesces with free neighbors. So most mallocs never enter the kernel.

**Q7.2 — brk vs mmap?** `brk`/`sbrk` moves the end of the single contiguous heap segment — glibc uses it for the main arena. `mmap` creates an independent mapping anywhere in the address space — glibc uses it for large allocations (default threshold 128KB). The practical difference: an mmap'd block is returned to the OS the moment you free it, while brk memory can only shrink from the top — one long-lived allocation at the top of the heap pins everything below it in the process.

**Q7.3 — Internal vs external fragmentation?** **Internal**: the allocator rounds your request up to a size class, wasting the slack _inside_ the allocation. **External**: after many allocs and frees, free memory is scattered in small chunks — plenty of total free bytes, but no contiguous run big enough for a large request. **[HFT]** Long-running processes fight external fragmentation with fixed-size object pools and arena allocators — same-size objects can't fragment.

**Q7.4 — Why is `malloc` on the hot path a problem for HFT?** Nondeterministic latency. A malloc might be a 20ns free-list pop — or a lock contention stall in a multithreaded allocator, or a syscall for fresh memory, followed by page faults on first touch of the new pages. The median is fine; the tail is not, and HFT is a tail-latency business. Standard answer: preallocate and pre-fault everything at startup, use object pools and ring buffers, zero allocations per message.

---

## 8. Synchronization

**Q8.1 — Mutex vs semaphore?** A **mutex** is mutual exclusion with ownership: the thread that locks must unlock, and it protects a critical section. A **semaphore** is a counter with `wait` (decrement-or-block) and `post` (increment) and **no ownership** — any thread can post. Use a counting semaphore to track N available resources, and a binary semaphore for _signaling_ between threads — one thread waits, another posts — which a mutex can't express because the poster isn't the locker.

**Q8.2 — What is a condition variable and how do you use it correctly?** It lets a thread sleep until some predicate over shared state becomes true. Correct usage is a fixed pattern: lock the mutex, then `wait` **in a while loop** re-checking the predicate; the wait atomically releases the mutex while sleeping and re-acquires it on wake. The loop is mandatory because of **spurious wakeups** and because another thread can consume the condition between the notify and your wake. The notifier locks, changes state, then calls `notify_one`/`notify_all`.

**Q8.3 — What is a spinlock, and when is it the right choice?** A lock that busy-waits in a CAS/test-and-set loop instead of sleeping. Right when the critical section is tiny — tens of nanoseconds — and the holder runs on another core, because sleeping and waking through the kernel costs microseconds, orders of magnitude more than the wait itself. Wrong on a single core (you spin against a holder that can't run) and for long holds (burning a core). **[HFT]** Latency-critical code prefers spinning generally — spinlocks, busy-polling queues — because a sleeping thread pays wakeup latency exactly when the market moves.

**Q8.4 — Summary: when do you reach for each primitive?** Mutex: default for protecting shared data. Condition variable: waiting for a state change without burning CPU. Semaphore: counting resources or cross-thread signaling. Spinlock: sub-microsecond critical sections on multicore where wake latency is unacceptable. Atomics: single-word counters and flags, and lock-free structures when you can justify the complexity.

---

## 9. Deadlock

**Q9.1 — What are the four conditions for deadlock?** All four must hold simultaneously: **mutual exclusion** — resources can't be shared; **hold and wait** — a thread holds one resource while waiting for another; **no preemption** — resources can't be forcibly taken; **circular wait** — a cycle of threads each waiting on the next. Break any one and deadlock is impossible — which is exactly how prevention strategies are organized.

**Q9.2 — How does lock ordering prevent deadlock?** Impose a global total order on all locks and require every thread to acquire in that order. Then a cycle can't form: a cycle needs some thread holding a higher-ranked lock while waiting for a lower-ranked one, which the discipline forbids — circular wait is structurally eliminated. In C++, when you must take two locks whose order varies by call site, `std::scoped_lock(m1, m2)` (or `std::lock`) acquires both with a deadlock-avoidance algorithm.

**Q9.3 — Classic example of deadlock in two lines?** Thread A locks `m1` then wants `m2`; thread B locks `m2` then wants `m1`. Each holds what the other needs; both wait forever. Fix by ordering (both take `m1` before `m2`) or by `std::scoped_lock(m1, m2)` at both sites.

**Q9.4 — Deadlock vs livelock vs starvation?** Deadlock: threads blocked forever in a wait cycle — no one runs. Livelock: threads keep _running_ but make no progress — e.g., both back off and retry in lockstep forever. Starvation: the system progresses but some thread never gets its turn — e.g., a low-priority thread under constant high-priority load.

---

## 10. Data Races vs Race Conditions

**Q10.1 — What's the difference between a data race and a race condition?** A **data race** is precisely defined: two threads access the same memory location concurrently, at least one is a write, and there's no synchronization between them — and in C++ that is **undefined behavior**, so the compiler may reorder, cache in registers, or miscompile around it. A **race condition** is a _logic_ bug: correctness depends on timing or interleaving. They're independent — you can have a race condition in fully synchronized, data-race-free code, like a check-then-act split across two separately-locked calls.

**Q10.2 — Give an example of each.** Data race: two threads doing `counter++` on a plain `int` — read-modify-write with no sync; UB, and in practice lost updates. Race condition without a data race: `if (!map.contains(k)) map.insert(k, v)` where each call locks internally — no data race anywhere, but another thread can insert between your check and your act. The fix is different too: atomics fix the first; _widening the critical section_ to cover the whole check-then-act fixes the second.

**Q10.3 — How do atomics fit in?** `std::atomic` operations are exempt from the data-race rule and add **memory ordering**: `seq_cst` by default, or acquire/release for pay-only-what-you-need ordering — a release store makes all prior writes visible to the thread whose acquire load sees it. Atomics eliminate data races on that variable, but they do **not** fix race conditions: an atomic counter can still be part of a broken check-then-act. For a compound operation, you need a CAS loop or a lock.

**Q10.4 — Why is a data race UB rather than just "you get a stale value"?** Because the compiler and CPU optimize assuming no unsynchronized concurrent access: the compiler may hoist a read out of a loop (so you _never_ see the other thread's write), keep values in registers, or tear a wide store. The sanitizer answer: TSan exists precisely because you cannot reason about a racy program's behavior from the source.

---

## 11. IPC

**Q11.1 — How do pipes work, and what's the fork+close pattern?** `pipe(fd)` gives a unidirectional kernel buffer: `fd[0]` reads, `fd[1]` writes. You fork, so **both** processes hold both ends — then _each side closes the end it doesn't use_. That close discipline is load-bearing: a reader gets EOF only when **every** write end in every process is closed, so if the reader forgets to close its copy of the write end, it holds the pipe open against itself and its final `read` blocks forever. Writing to a pipe with no readers raises SIGPIPE.

**Q11.2 — How does a shell implement `ls | wc`?** Create the pipe, fork twice. First child: `dup2(fd[1], STDOUT_FILENO)`, close both originals, `exec ls`. Second child: `dup2(fd[0], STDIN_FILENO)`, close both, `exec wc`. Parent closes both ends and waits. The programs themselves are oblivious — they just use stdin/stdout; that's the fork/exec gap plus everything-is-a-file paying off.

**Q11.3 — What is a named pipe (FIFO)?** A pipe with a filesystem name (`mkfifo`), so **unrelated** processes — no common ancestor — can rendezvous by opening the same path. Same semantics as a pipe once open; opening blocks until both a reader and writer exist.

**Q11.4 — Shared memory — why is it the fastest IPC?** `shm_open` + `mmap` maps the same physical pages into both processes' address spaces. After setup, data transfer involves **no syscalls and no copies** — a write on one side is just a memory store the other side can load. Cost: the kernel no longer serializes access, so you must synchronize yourself — process-shared mutexes, semaphores, or atomics inside the region. **[HFT]** This is the standard intra-box transport: single-producer single-consumer ring buffers in shared memory, e.g. market data fanned out from the feed handler to strategy processes.

**Q11.5 — Sockets vs the others?** Sockets are bidirectional and, uniquely, work **across machines** (TCP/UDP). Locally, **Unix domain sockets** skip the network stack — faster than TCP loopback — and can pass file descriptors and credentials between processes. General rule: sockets for anything that may cross a machine boundary; shared memory when both ends are on the same box and latency matters.

**Q11.6 — Signals as IPC?** Asynchronous notification with essentially no payload — just the signal number (a bit more with `sigqueue`). Good for control-plane events: terminate, reload config, child died (SIGCHLD). Bad for data: not queued (standard signals coalesce), and handlers are minefields — so real programs turn signals into FD events via `signalfd` or the self-pipe trick and handle them in the main loop.

---

## 12. File Descriptors & I/O

**Q12.1 — What exactly is a file descriptor?** A small non-negative integer indexing the process's **FD table**. Each entry points to a kernel **open file description** — which holds the offset and status flags — which points to the underlying object: file, pipe, socket, device. Two levels matter: `dup` gives two FDs sharing one description (shared offset), while two separate `open`s of the same file give independent offsets. FDs 0/1/2 are stdin/stdout/stderr; FDs survive fork and, by default, exec — `O_CLOEXEC` opts out.

**Q12.2 — What does `dup2(oldfd, newfd)` do, and what's it for?** It makes `newfd` refer to the same open file description as `oldfd`, silently closing `newfd` first if it was open — atomically. Canonical use is redirection before exec: `dup2(pipe_write, STDOUT_FILENO)` makes the exec'd program's printf go into the pipe without the program knowing. Both descriptors then share one offset and flags.

**Q12.3 — What does "everything is a file" mean and why is it powerful?** One uniform interface — open/read/write/close on an FD — over wildly different resources: files, pipes, sockets, terminals, devices, and modern kernel objects like `timerfd`, `eventfd`, `signalfd`, epoll instances. The payoff is **composition**: shell redirection, `select`/`epoll`, and generic I/O code work over any of them without knowing what's underneath. It's why an event loop can wait on a socket, a timer, and a signal in one `epoll_wait`.

**Q12.4 — Semantics of `read()` people get wrong?** `read` may return **fewer bytes than requested** — a short read — perfectly legally, especially on sockets and pipes; correct code loops until it has what it needs. Return of **0 means EOF**; -1 means error, with `EINTR` (interrupted by signal — just retry) and `EAGAIN` (non-blocking, nothing available) being the two you must handle, not treat as fatal. Same short-write caveat applies to `write`.

---

## 13. I/O Multiplexing

**Q13.1 — select vs poll vs epoll?** All three wait on many FDs at once. `select`: FDs in fixed bitmasks capped at 1024, sets are destroyed each call so you rebuild them, and the kernel scans every FD — O(n) per call. `poll`: an array of pollfds, no 1024 cap, but still O(n) scanning every call. `epoll`: register interest **once** with `epoll_ctl`; the kernel maintains a ready list as events arrive, and `epoll_wait` returns only ready FDs — O(ready). With 100k mostly-idle connections, that's the whole game; epoll is why Linux servers scale.

**Q13.2 — Level-triggered vs edge-triggered?** **Level**: `epoll_wait` reports an FD as long as the condition holds — readable data still buffered means you get told again next call. Forgiving. **Edge**: you're notified only on the _transition_ — new data arriving. So with edge you must set the FD non-blocking and **drain it completely** — read until `EAGAIN` — or remaining data sits there and you never get another notification: a silent hang. Edge means fewer wakeups and is what high-performance servers use, paired with that drain discipline.

**Q13.3 — What is non-blocking I/O and how does it relate?** `O_NONBLOCK` makes read/write return `EAGAIN` immediately instead of sleeping when they can't proceed. It's the foundation of event loops: epoll tells you _which_ FDs are actionable, non-blocking ops let you service them without ever getting stuck on one — mandatory under edge triggering, and wise even under level (readiness can be spurious).

**Q13.4 — [HFT] Why do trading systems often not use epoll on the hot path?** Because `epoll_wait` sleeping in the kernel means a wakeup — scheduler latency, mode switch — exactly when a packet arrives. Latency-critical receive paths **busy-poll** instead: a pinned thread spins on the NIC's ring buffers in user space via kernel-bypass (DPDK, Solarflare/Onload), never sleeping, never syscalling. epoll remains the right answer for the thousands of non-critical connections.

---

## 14. Signals

**Q14.1 — SIGSEGV?** Raised synchronously when the process makes an invalid memory access — unmapped address, writing read-only memory, stack overflow into the guard page. Default action: terminate with a core dump. It _can_ be caught — JITs and GCs do this deliberately — but after a genuine bug, returning from the handler re-executes the faulting instruction, so for normal programs it's for logging-then-dying, not recovery.

**Q14.2 — SIGKILL vs SIGTERM?** `SIGTERM` is the polite request — default is termination, but it's **catchable**, so processes install handlers to flush, release resources, and exit cleanly; it's what plain `kill` sends. `SIGKILL` cannot be caught, blocked, or ignored — the kernel just destroys the process, no cleanup, no atexit, buffered data lost. So the discipline is TERM first, wait, KILL only as a last resort. (SIGSTOP is similarly uncatchable, for suspension.)

**Q14.3 — SIGINT?** What the terminal sends to the **foreground process group** on Ctrl-C. Default terminates; interactive programs catch it to cancel the current operation instead of dying. Being group-delivered is why Ctrl-C kills an entire pipeline at once.

**Q14.4 — What can you safely do inside a signal handler?** Almost nothing: a handler interrupts your code at an arbitrary instruction — possibly mid-`malloc` — so it may only call **async-signal-safe** functions. `printf` and `malloc` are not safe (they take locks; you self-deadlock). The two sanctioned patterns: set a `volatile sig_atomic_t` flag the main loop polls, or `write` a byte to a self-pipe/eventfd so the event loop sees it as a normal FD event — which is what `signalfd` formalizes.

---

## 15. Linking & Loading

**Q15.1 — Static vs dynamic linking?** **Static**: the linker copies needed library code into the executable at build time — self-contained binary, no runtime dependencies, dead-code elimination and better inlining, but bigger, no page sharing across processes, and a library fix means relinking every binary. **Dynamic**: the executable records needed `.so` names; the runtime loader maps them at startup and resolves symbols — shared physical pages across all users of libc, independently updatable libraries, but startup relocation cost and dependency-version risk. **[HFT]** Trading binaries are often statically linked for deployment determinism — the binary you tested is exactly what runs.

**Q15.2 — What does `ldd` show?** The shared libraries a binary depends on — direct and transitive — and the path where the loader resolves each one right now, or "not found" for missing ones. It answers "what will actually be loaded on _this_ machine," which makes it the first tool for dependency-hell debugging. (Caveat worth dropping: `ldd` works by invoking the loader, so don't run it on untrusted binaries.)

**Q15.3 — What happens between `exec` and `main`?** The kernel maps the executable's segments and, for a dynamic binary, maps the interpreter — `ld.so` — and starts _there_. The loader maps each needed `.so`, performs relocations, resolves symbols, runs initializers (constructors of globals — this is where C++ static init order problems live), then jumps to `_start`, which sets up the C runtime and calls `main`.

**Q15.4 — What are the PLT and GOT, in one breath?** Calls into shared libraries go through the **Procedure Linkage Table**, which jumps via an address slot in the **Global Offset Table**. By default that slot is resolved lazily on first call; `LD_BIND_NOW`/`-z now` resolves everything at startup instead. **[HFT]** Lazy binding means the _first_ call to a function eats a symbol-resolution stall — one more reason latency-critical systems bind at startup or link statically, and warm every path before go-live.

---

## Quick-Fire Round (one-liners to have loaded)

- **fork returns twice** — 0 in child, PID in parent.
- **Zombie** = dead, unreaped. **Orphan** = live, re-parented to init.
- **n forks → 2ⁿ processes.** Unflushed stdout buffers duplicate across fork.
- **Threads share the address space and FD table; stacks and registers are private.**
- **Process switch flushes TLB state; thread switch doesn't** — that's the cost gap.
- **CFS**: run the task with the lowest vruntime, from a red-black tree.
- **Syscall ≈ 100s of ns; function call ≈ 1 ns** — hence batching and kernel bypass.
- **TLB miss = a 4-level page-table walk.** Huge pages stretch TLB reach.
- **Minor fault: no disk. Major fault: disk.** Pre-fault + mlock kills both.
- **malloc is a library, not a syscall**; brk grows the heap, mmap for ≥128KB.
- **Mutex has an owner; semaphore doesn't** — that's why semaphores can signal.
- **CV waits go in a while loop** — spurious wakeups, stolen conditions.
- **Deadlock needs all four conditions; lock ordering kills circular wait.**
- **Data race = UB, fixed by atomics/locks. Race condition = logic bug, fixed by widening the critical section.**
- **Pipe readers see EOF only when _all_ write ends are closed** — close what you don't use.
- **dup2 before exec = redirection.** Shared open-file description = shared offset.
- **epoll is O(ready); select/poll are O(n).** Edge-triggered ⇒ non-blocking + drain to EAGAIN.
- **SIGKILL/SIGSTOP are uncatchable; everything else is negotiable.**
- **Handlers: async-signal-safe only** — set a flag or write to a self-pipe.
- **ldd = what the loader will actually map, resolved for this machine.**age numbers