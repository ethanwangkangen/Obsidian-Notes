Virtual Machine Monitor
- Software that manage the virtual machines, give them illusion that they are the only one running on the machine

Goal
- Isolation of Code, data and resources between
	- Guest VM and host VMM
		- VMM is in TCB of system, but not get compromised
	- Between VMs
Assumptions
- Bug free TCB: Host OS, VMM
- Malware can affect guest OS and apps

Application
- Virtual Machine isolation (different VMs)
	- Dynamic analysis/containment of malware
- Virtual Machine Introspection
	- eg. run anti virus in VMM

Security VMM goals
- Complete mediation
	- If guest VM tries to change anything privileged, VMM supervises this
- Trap on all MMU, DMA, I/O accesses
- Transparency
	- VM thinks it's running on real machine, and not through VMM

Compatibility Issues
- Page table with accessibility bits
- OS will think that it has privileged access to hardware
	- But the VMM sits between the Guest OS and the hardware
	- So the Guest OS now has to sit at a less privileged level, so the accesses to hardware will fail
	- Solution: Trap to Host OS(VMM) when wanting to access main memory

What about privileged instructions that don't even trap?
- Silent failure
- Solution: Dynamic binary translation
- Break instructions into blocks, then when find things that should trap, replace the instruction