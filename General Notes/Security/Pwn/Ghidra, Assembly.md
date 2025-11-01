64 bit architecture:
- arguments passed to functions through registers

| Order | Register |
| ----- | -------- |
| 1     | rdi      |
| 2     | rsi      |
| 3     | rdx      |
| 4     | rcx      |
| 5     | r8       |
| 6     | r9       |
|       |          |
Return values:
- Integer/pointer -> rax
- Float/double -> xmm0

That's how we can identify entry point (ie. main) address even in a stripped binary.
- Program starts with call to \__libc_start_main, ie entry function
- Address of main should be loaded into the rdi register
	- Because it is the first argument passed to libc start main
	- `` `int __libc_start_main`(int (*_main_) (int, char **, char **), int _argc_, char ** _ubp_av_, void (*_init_) (void), void (*_fini_) (void), void (*_rtld_fini_) (void), void (*_stack_end_)); ``


# Assembly Instructions
## Data Movement

mov     dst, src        ; Copy src to dst
lea     dst, [addr]     ; Load effective address
xchg    a, b            ; Exchange a and b

## Stack Operations

push    reg             ; Push register onto stack
pop     reg             ; Pop from stack into register
leave                  ; mov rsp, rbp; pop rbp (function epilogue)
ret                    ; Return from function (pop RIP from stack)
ret n                  ; Return and clean up n bytes of arguments

## Control Flow

call    addr            ; Call function (push RIP, jmp)
jmp     addr            ; Unconditional jump
jmp     reg             ; Jump to address in register (useful in ROP)

je/jz    addr           ; Jump if equal / zero
jne/jnz  addr           ; Jump if not equal / not zero
jg/jnle  addr           ; Jump if greater (signed)
jl/jnge  addr           ; Jump if less (signed)
ja/jnbe  addr           ; Jump if above (unsigned)
jb/jnae  addr           ; Jump if below (unsigned)

## Arithmetic / Logic

add     dst, src        ; dst += src
sub     dst, src        ; dst -= src
inc     dst             ; dst += 1
dec     dst             ; dst -= 1
imul    dst, src        ; Signed multiplication
idiv    src             ; Signed division (uses rax/rdx)
neg     dst             ; dst = -dst

and     dst, src        ; Bitwise AND
or      dst, src        ; Bitwise OR
xor     dst, src        ; Bitwise XOR
not     dst             ; Bitwise NOT

## Comparison & Test

cmp     a, b            ; Compare a - b (sets flags)
test    a, b            ; Bitwise AND and set flags (does not store)

## Shifts & Rotates

shl     dst, n          ; Shift left (multiply by 2^n)
shr     dst, n          ; Logical shift right (unsigned)
sar     dst, n          ; Arithmetic shift right (signed)
rol     dst, n          ; Rotate left
ror     dst, n          ; Rotate right

## Flags & Conditional Set

setnz/setne reg         ; Set if not zero
setz/sete   reg         ; Set if zero
setg/setnle reg         ; Set if greater
setl/setnge reg         ; Set if less

# Return Oriented Programming
- pop rsi: 
	- reads 8 bytes from top of stack (ie. Stack Pointer/RSP address.)
	- loads that 8-byte value into the register rsi.
	- increments RSP by 8, ie popping the value from the stack

- ret
	- reads 8 bytes from top of stack (ie. Stack Pointer/RSP address.)
	- loads that 8-byte value into the rip register (will go to that address later) 
	- increments RSP by 8, ie popping the value from the stack


## What happens at function exit?

```c
(the previous stack is here)

↑ higher addresses
+------------------------+
| Return address to main |  ← b
+------------------------+
| Old RBP (from main)    |  ← a
+------------------------+
| Local variables        |
|                        |
|                        |
| ...                    |  ← rsp now (after `sub rsp, N`)

```

At the end of the function (epilogue), this happens:
```c
mov rsp, rbp 
pop rbp
ret
```

- RSP set to where current RBP is at.
	- RSP now at point a
- RBP holds address of previous RBP, and pop RBP sets RBP back to the previous RBP.
	- RSP now at point b (since pop will increment it )
- ret will increment RSP one last time
	- RSP now points to the top of the previous stack

## How to implement gadgets in ROP:
- Gadgets are all placed on the stack

The original stack looks like
```c
↑ Higher Addresses
───────────────────────────────────────────────
0x7fffffffeff8*       →  0x00401234     ; Return address to main (or caller)
0x7fffffffeff0  →  0x[saved RBP]  ; Previous base pointer (from main)
0x7fffffffefe0  →  [local buffer start]
↓ Lower Addresses
```

Place the gadget contents at the return address, after the rbp.
```c
↑ Higher Addresses
───────────────────────────────────────────────
0x7fffffffeff8 + 32  →  0x0000000000402000    ; Target function address (e.g., system)
0x7fffffffeff8 + 24  →  0xcafebabecafebabe    ; Argument for rsi
0x7fffffffeff8 + 16  →  0x0000000000401020    ; Gadget 2: pop rsi; ret
0x7fffffffeff8 + 8   →  0xdeadbeefdeadbeef    ; Argument for rdi
0x7fffffffeff8*      →  0x0000000000401010    ; Gadget 1: pop rdi; ret  ← ^Overwritten eturn address
0x7fffffffeff0       →  0x[bogus RBP or overwritten if needed]
0x7fffffffefe0       →  [overflowed buffer data begins here]
↓ Lower Addresses
```

At the end of execution of the original function,
- mov rsp, rbp -> rsp and rbp now at 0x7fffffffeff0
- pop rbp -> not that important. rsp moved up to 0x7fffffffeff8
- ret -> normally, this would set the RIP to 0x00401234, and increment rsp again.
	- However, we have modified this address to address of the first Gadget.

So once ret is called,
- rsp set to 0x7fffffffeff8 + 8
- rip set to 0x401010 (first gadget address)

First gadget executed
- pop rdi
	- 0xdeafbeefdeadbeef put into rdi, rsp set to 0x7fffffffeff8 + 16
- ret
	- rip set to 0x401020, rsp set to  0x7fffffffeff8 + 24

Second gadget executes
- pop rsi
	- 0xcafebabecafebabe put  into rsi, rsp set to 0x7fffffffeff8 + 32
- ret
	- rip set to 0x40200, rsp set to 0x7fffffffeff8 + 40


**Summary**
- Rewrite original return address to address of first gadget
- Then continue with value to put in register, then next gadget and so on