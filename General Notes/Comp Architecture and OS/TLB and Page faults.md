
# Memory address translation protocol
- Core hands virtual address to MMu
	- MMU checks TLB
	- TLB miss -> hardware walks page tables
	- If leaf PTE has present bit clear (page is not present) or access violates permission bits (write to read only page etc.) MMU raises a page fault exception
- Page fault
	- CPU switches to kernel mode, page handler decides what to do

# Page Fault
- Kernel first checks if address falls inside valid VMA (region the process has actually mapped)
	- No -> SIGSEGV
	- Yes -> fault is legit, valid address but not in memory
- 
