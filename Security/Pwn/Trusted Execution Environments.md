- Want to ensure that root of trust is our VMM (safe), and not something like attacker VMM

TEE
- hardware root of trust
- can assume software is alicious
- why hardware
	- tamper resistant from software malware
- example
	- sensitive application and CPU trusted
	- malware, OS, RAM, all untrusted

Recap on crypto primitives
- Confidentiality
- Integrity 
- Authenticity

# TEE primitive 1: remote attestation (PCR)
Trusted hardware 
- has extra registers (PCR)
- software cannot write there, only cpu can
- bios loaded -> cpu compute hash (PCRn||BIOS), store in PCRn
- Then do the same for boot loader, then os, then vmm, etc. store all the hashes
- cpu sign the hash digitally with private key (AIK refers to the key pair)
- verifier - check the hash with public key
	- can know that 1. THIS cpu signed the hash
	- 2. contents of packet did not change the hashes(?)

# TEE primitive 2: trusted boot
- no need for verifier, just check locally
- store the hashes in a secure storage
- no signing keys needed

# TEE primitive 3: sealing data (Secure Storage)
- Suppose we want to protect a disk encryption key (like in Windows BitLocker).
- The TPM can **seal** the key:
    - Store it in secure hardware, _bound_ to specific PCR values (the measured software stack).
    - In other words: “Only release this key if the boot state matches what I saw when sealing.”
- At boot time:
    - PCRs are recomputed as BIOS, bootloader, OS load.
    - If they match the original values, the TPM releases the disk key.
    - If not, the key stays locked (disk stays encrypted).

- **Remote Attestation:** Convince _someone else_ you’re in a trusted state.    
- **Trusted Boot:** Convince _yourself_ you’re in a trusted state.
- **Sealing:** Protect secrets so they’re only available in a trusted state.

Note that the protection from first 2 primitives is for measured software stack, not hardware
- software stack: bios, bootloader, os, vmm..
3rd primitives protects data..