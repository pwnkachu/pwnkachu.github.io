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