---
title: General Advice
description: Personal reference guide of tips and tricks needed to solve binary exploitation challenges.
---

# Stack Exploitation Reference

### The `__x86.get_pc_thunk` Function (32-bit PIC)

This is a function you will frequently encounter in **32-bit (x86)** executables compiled with Position Independent Code (PIC/PIE). It exists because 32-bit x86 architecture lacks native Instruction Pointer (`EIP`) relative addressing. 

!!! note "IP Relative Addressing"
    A method in assembly where a memory address is calculated by adding a fixed offset to the current Instruction Pointer. It allows code to run correctly anywhere in memory (ASLR) without needing hardcoded absolute addresses. **64-bit (x86_64) architectures support this natively**, so they do not need this thunk.

To calculate addresses relative to the current execution point, 32-bit binaries use this function:
```asm
__x86.get_pc_thunk.bx:
    mov ebx, dword ptr [esp] ; Move the return address (current execution point) into EBX
    ret                      ; Return to the caller
```

*How it works*: When the program calls this function (call __x86.get_pc_thunk.bx), the call instruction pushes the address of the next instruction onto the stack. The thunk simply reads that value directly from the top of the stack ([esp]) into a register (usually EBX or CX), effectively giving the program its own current location in memory.

### SIGSEGV Fault Due to movaps (Stack Alignment)

In 64-bit systems (like modern Ubuntu), glibc heavily uses SIMD vector instructions to optimize performance in functions like printf or system.

The movaps instruction moves 16 bytes of data at a time into special XMM registers. However, there is a strict hardware rule: movaps requires the memory address to be a multiple of 16 bytes (the address must end with 0 in hex). If the Stack Pointer (RSP) is misaligned (e.g., ends in 8), the CPU throws a General Protection Fault, resulting in a segmentation fault (SIGSEGV).

When you execute a standard Buffer Overflow / Ret2Win, you skip the legitimate call instruction and use ret to jump directly into the target function. Because a call normally pushes 8 bytes to the stack, skipping it leaves your stack misaligned by exactly 8 bytes.

The Fix: Add a simple ret gadget to your ROP chain right before the target function address. This forces a "dummy" return, advancing the stack pointer by 8 bytes and bringing it back into perfect 16-byte alignment.

### Calling Conventions (x86 vs x64)

When building ROP chains or shellcode to call functions (like system("/bin/sh") or execve), you must format the arguments according to the architecture's calling convention.

    32-bit (cdecl): Arguments are passed entirely on the stack, pushed in reverse order (right-to-left).

        Payload structure: [Function Addr] + [Return Addr] + [Arg 1] + [Arg 2]

    64-bit (System V AMD64 ABI): The first six integer/pointer arguments are passed via registers. Any additional arguments go on the stack.

        Register Order: RDI, RSI, RDX, RCX, R8, R9

        Payload structure: You need pop rdi; ret gadgets to load your arguments into the registers before calling the function.

### Stack Pivoting (leave; ret)

If you have a buffer overflow but the vulnerable buffer is too small to fit a full ROP chain, you can move (pivot) the stack to another location in memory that you control (like a global variable in the .bss segment or the heap).

This is achieved by overwriting the saved Base Pointer (RBP) and utilizing the leave instruction, which translates to:

```
mov rsp, rbp  ; Sets the Stack Pointer to wherever RBP is pointing
pop rbp       ; Restores RBP
```

By overwriting the saved RBP with your fake address and ensuring the function ends with leave; ret (main), you trick the program into treating your fake memory region as the real stack.

### Intel CET and the Modern PLT: `.plt`, `.plt.sec`, and `endbr64`

In modern x86_64 binaries, the Procedure Linkage Table (PLT) is often split into two sections: `.plt` and `.plt.sec`. Additionally, you will see the `endbr64` instruction everywhere. This new structure is designed to support **Intel CET** (Control-flow Enforcement Technology), a hardware defense mechanism against ROP and JOP exploits.

#### `endbr64`
Intel CET introduces **IBT** (Indirect Branch Tracking). IBT enforces a strict rule to prevent attackers from hijacking execution flow: **every indirect jump or call must land on an `endbr64` instruction**. 

If an indirect jump (like `jmp *rax` or a jump via a GOT entry) lands on anything else, the CPU triggers a hardware exception and terminates the process.

#### The Legacy PLT Problem
The traditional PLT handled both execution and lazy binding in a single, strictly 16-byte aligned block. However, with CET active:
1. Calls via function pointers (indirect calls) targeting the PLT require an `endbr64` at the start.
2. The lazy binding "bounce" from the GOT back to the PLT is an indirect jump, requiring a second `endbr64`.

Squeezing two 4-byte `endbr64` instructions into a rigid 16-byte block would destroy performance and alignment. The solution was to split the PLT in two.

#### The 2-PLT Architecture
Modern linkers separate the PLT into a "fast path" for execution and a "slow path" for resolution.

#### `.plt.sec`
This is the section your code actually calls (e.g., `call printf@plt.sec`). It is streamlined, CET-compliant, and handles the actual jump to the GOT.

```assembly
endbr64          # Safe landing pad for function pointers
jmp *GOT[func]   # Indirect jump to the real address (or back to .plt)
nop              # Alignment padding
```

#### .plt (The Slow Path / Lazy Binding)

This section is only used during the very first call to a function, when the GOT still contains the fallback address instead of the real library address.

```
endbr64          # Safe landing pad for the GOT bounce-back
push id          # Push the relocation index
jmp plt0         # Jump to the dynamic resolver trampoline
```

I wrote this because I was stuck on a challenge where I used a Use-After-Free (UAF) vulnerability to overwrite the GOT, specifically trying to replace free@got with system. Due to alignment constraints, my write had to start at GOT[2] (the dynamic resolver pointer) just to reach the free@got entry. I had managed to force the resolution of system earlier in the program, but whenever I tried to redirect execution to system@plt, it resulted in a segmentation fault. My exploit kept failing because I didn't understand the architectural split between .plt and .plt.sec. Once I realized I needed to point to system@plt.sec since the function was already resolved the exploit worked.