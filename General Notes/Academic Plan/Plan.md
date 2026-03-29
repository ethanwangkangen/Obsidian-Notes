## **Phase 1: C++ Fundamentals + OSTEP (Mar 29 – May 3)**

| **Week** | **Dates**       | **Primary Focus**                                  | **OSTEP Focus**                      |
| -------- | --------------- | -------------------------------------------------- | ------------------------------------ |
| **W1**   | Mar 29 – Apr 5  | Finish LearnCPP.com: Move Semantics & Templates    | Ch 1-6 (CPU Virtualisation)          |
| **W2**   | Apr 6 – Apr 12  | Consolidate C++ Notes, Internal Logic Docs         | Ch 7-11 (Scheduling)                 |
| **W3**   | Apr 13 – Apr 19 | A Tour of C++ Ch 1–7 (speed run)                   | Ch 12-17 (Address Spaces/Memory API) |
| **W4**   | Apr 20 – Apr 26 | A Tour of C++ Ch 8–15, Library & Concurrency intro | Ch 18-24 (Paging & TLBs)             |
| **W5**   | Apr 27 – May 3  | Exam Break. LeetCode Hard grind (3/day)            | Finish OSTEP & write summary notes   |

> **Notes:** Concurrency theory is lightly introduced in W4; deeper concurrency waits until projects.

---

## **Phase 2: STL + Modern C++ (May 4 – May 31)**

| **Week** | **Dates**       | **Effective Modern C++ Focus**            | **STL Implementation / Concurrency Prep**                            |
| -------- | --------------- | ----------------------------------------- | -------------------------------------------------------------------- |
| **W6**   | May 4 – May 10  | Items 1-10: Type Deduction & `auto`       | `std::vector`: emplace_back & exponential growth                     |
| **W7**   | May 11 – May 17 | Items 11-17: Modern C++ Features          | `std::unique_ptr`: move-only & custom deleters                       |
| **W8**   | May 18 – May 24 | Items 18-30: Smart Pointers / Rvalue Refs | `std::shared_ptr`: atomic reference counting (introduces atomic ops) |
| **W9**   | May 25 – May 31 | Items 31-42: Lambdas & Concurrency        | `std::function` implementation; prep for threaded Project 2          |

> **Notes:** W8–W9 begin laying foundations for concurrency: atomics, memory order, shared_ptr atomicity.

---

## **Phase 3: Projects + Concurrency (Jun 1 – Jul 12)**

| **Week** | **Dates**       | **Focus**                              | **OSTEP Chapters**                                          | **C++ Concurrency / Projects / Daily C++ Interview**                                                                                  |
| -------- | --------------- | -------------------------------------- | ----------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| **W10**  | Jun 1 – Jun 7   | Project 1: Single-threaded LOB         | -                                                           | Daily C++ Interview Ch 1–15; light threading theory (Williams, intro threads)                                                         |
| **W11**  | Jun 8 – Jun 14  | Project 2 Start: SPSC Ring Buffer      | Ch 29–30 (Threads, Thread API)                              | Implement SPSC with `std::atomic`; Daily C++ Interview Ch 16–30                                                                       |
| **W12**  | Jun 15 – Jun 21 | Project 2 Finish: MPMC Queue           | Ch 31–32 (Mutexes, Synchronisation, Concurrency Challenges) | Stress-test MPMC, false sharing, memory order; Daily C++ Interview Ch 31–45; Williams: mutexes, atomics                               |
| **W13**  | Jun 22 – Jun 28 | Project 3 Start: UDP Binary Parser     | Ch 33 (Advanced Concurrency)                                | Multi-threaded pipeline (ingest → parse → queue); Daily C++ Interview Ch 46–60; Williams: condition variables, thread safety patterns |
| **W14**  | Jun 29 – Jul 5  | Project 3 Finish: Zero-copy & SIMD     | -                                                           | Optimise threading, benchmarks, lock-free integration; Daily C++ Interview Ch 61–75; Williams: atomics, memory fences, task division  |
| **W15**  | Jul 6 – Jul 12  | Profile all projects                   | -                                                           | Concurrency review & optimisations; finish Daily C++ Interview Ch 76–90; stress-test thread pools and atomic structures               |
| **W16**  | Jul 13 – Jul 20 | APPLY: Resume polish, HFT applications | -                                                           | Showcase concurrency projects; optional final Daily C++ Interview Ch 91–100                                                           |
| **W17**  | Jul 20 – Jul 27 | Mental Reset + Review                  | -                                                           | Revisit threading notes & concurrency projects; small concurrency exercises                                                           |

> **Notes:**
> 
> - Projects now **explicitly include concurrency**: Project 2 is atomic/lock-free, Project 3 is multi-threaded.
> - Daily C++ Interview reading is integrated weekly; by end of W15 you finish page coverage ~90 pages, leaving the last sections for July.

---

## **Phase 4: Internship Prep + Mental Reset (Jul 13 – Jul 27)**

|**Week**|**Dates**|**Focus**|**Action**|
|---|---|---|---|
|**W16**|Jul 13 – Jul 20|START APPLYING: polish resume, projects|Apply to HFT firms; showcase concurrency projects|
|**W17**|Jul 20 – Jul 27|RESERVIST: review all notes & projects|Mental reset; last Daily C++ Interview pages 91–100|