---
title: x86-64 Assembly Learning Guide
description: Personal reference guide for x86-64 assembly instructions, memory layout, and architecture concepts.
---

# x86-64 Assembly Learning Guide

## Introduction

This document captures my learning journey through **OpenSecurityTraining 1001: x86-64 Assembly**. This guide serves as my personal reference for key concepts, instructions, and practical applications of x86-64 assembly language.

---

## Architecture Concepts

### C Data Sizes & Types

In x86-64 assembly, data types are directly associated with specific hardware word sizes:

* **Char:** 1 Byte
* **Short / Word:** 2 Bytes
* **Int / Long / Float $\rightarrow$ DWord (Double Word):** 4 Bytes
* **Double / Long Long $\rightarrow$ QWord (Quad Word):** 8 Bytes
* **Pointers (int *ptr):** 4 Bytes on 32-bit systems, 8 Bytes on 64-bit systems.

!!! info "Union Memory Layout"
    Inside a `union`, every member shares the exact same memory space. The total size allocated to the union corresponds to the size of its largest member type.

### Endianness

Endianness defines how bytes (not bits) are ordered and stored within system memory. Consider the 4-byte value **0x12345678**:

#### Big Endian (Standard for Network Traffic)
The most significant byte is stored first (at the lowest memory address).
* Memory Layout: `0x12`, `0x34`, `0x56`, `0x78`

#### Little Endian (Standard for Intel/AMD x86)
The least significant byte is stored first (at the lowest memory address).
* Memory Layout: `0x78`, `0x56`, `0x34`, `0x12`

!!! note "Registers vs RAM"
    Endianness **only** applies when data is written to or read from RAM. Inside CPU registers, values are always represented in standard mathematical layout, meaning the most significant digits are always on the left. You must account for this discrepancy when interpreting raw bytes inside reverse engineering tools like Ghidra or IDA Pro.

### Computer Memory Hierarchy

Data access speeds vary drastically across the system layout. The memory hierarchy scales from fastest/smallest to slowest/largest:

$$\text{Processor Registers} \rightarrow \text{Processor Cache (L1/L2/L3)} \rightarrow \text{Random Access Memory (RAM)} \rightarrow \text{Flash/USB Memory} \rightarrow \text{Hard Drives (SSD/HDD)} \rightarrow \text{Tape Backup}$$

When writing or reversing assembly, the vast majority of our low-level tracking happens directly within the **Processor Registers**.

---

## CPU Registers

Registers are high-speed, volatile storage cells built directly into the processor core. 

Intel architectures provide **16 General Purpose Registers (GPRs)** plus a specialized **Instruction Pointer (RIP)**, which continuously points to the *next* instruction queued for execution.

* **x86-32 (32-bit):** Registers are 32 bits wide.
* **x86-64 (64-bit):** Registers are extended to 64 bits wide.

### Register Breakdown & Backward Compatibility

To maintain backward compatibility with older 16-bit and 32-bit processors, 64-bit registers can be accessed in smaller subdivisions.

Taking the primary accumulator register **RAX** as an example:

* **RAX:** The full 64-bit register.
* **EAX:** The lower 32 bits of RAX. (Writing to EAX automatically zero-extends into the upper 32 bits of RAX).
* **AX:** The lower 16 bits of RAX.
* **AH / AL:** The high and low 8-bit slices of the 16-bit `AX` register.

#### The Architecture Extension Legacy
The structural split exists because early architectures only had 8-bit or 16-bit registers (like `AX`). As computing shifted to 32-bit (`EAX`) and later 64-bit (`RAX`), the hardware designers simply wrapped the newer, wider registers around the legacy ones. 

For the newer registers introduced in x64 (`R8` through `R15`), a cleaner sizing convention is used:
* **R8:** 64-bit (Quad Word)
* **R8D:** 32-bit (Double Word)
* **R8W:** 16-bit (Word)
* **R8B:** 8-bit (Byte)

### General Purpose Registers (GPR) Cheat Sheet

While modern compilers use registers interchangeably for optimizations, standard software conventions give them dedicated roles:

* **RAX:** Stores function return values.
* **RBX:** Base pointer for data sections.
* **RCX:** Loop counter for string and iteration operations.
* **RDX:** I/O pointer and secondary data register.
* **RSI:** Source Index pointer for string/stream operations.
* **RDI:** Destination Index pointer for string/stream operations.
* **RSP:** Stack Pointer (Points to the top item on the current stack frame).
* **RBP:** Base Pointer (Points to the base of the current stack frame).
* **RIP:** Instruction Pointer (Contains the memory address of the next execution step).

---

## Process Creation & Memory Management

When a static executable file (like an ELF binary on Linux or a PE `.exe` on Windows) is executed, the Operating System reads it from storage and maps its contents into system RAM. This mapping provides the CPU with high-speed access to the application instructions and structural runtime sections.

### Virtual Memory & Isolation

To maintain application stability and security, the OS never exposes real *physical* RAM addresses directly to an application. Instead, it creates an abstraction layer called **Virtual Address Space**.

This space provides each running process with the illusion that it has an independent, uninterrupted block of memory all to itself.

* **The Page Table:** The OS manages a specialized translation directory called a Page Table. The CPU's hardware Memory Management Unit (MMU) uses this table to map continuous virtual addresses requested by the app into fragmented, discontinuous physical RAM locations on the fly. This avoids wasting memory resources.

### Process Memory Segments

During process instantiation, the Virtual Memory Space is organized into standard logical segments, sorted here from lowest memory addresses to highest memory addresses:

```
[LOW ADDRESSES]
├── Text Segment (Code)  --> Read-Only, contains compiled machine code instructions.
├── Data Segment         --> Read/Write, contains initialized global/static variables.
├── BSS Segment          --> Read/Write, contains uninitialized global/static variables (Zeroed out).
├── Heap                 --> Grows UPWARD towards higher addresses. Allocated at runtime.
│
▼

▲

├── Stack                --> Grows DOWNWARD towards lower addresses. Stores local frames.
[HIGH ADDRESSES]
```

#### 1. Text Segment (.text)
Contains the raw machine code instructions executed by the CPU. It is explicitly marked as **Read-Only** to prevent code injection attacks and can be shared among multiple instances of the same program to conserve RAM.

#### 2. Data Segment (.data)
Contains global and static variables that have been explicitly initialized with a value in the source code.
* Example: `int counter = 10;` (declared outside of `main`)

#### 3. BSS Segment (.bss)
Contains global and static variables that are declared but not initialized to a value. 
* Example: `int buffer[100];` (declared outside of `main`)
* The OS automatically wipes this entire region with zeroes upon process boot.

#### 4. Heap
A dynamic, developer-managed memory pool allocated at runtime (via `malloc()` or `new`). The Heap boundary expands **upward** toward higher memory addresses.

#### 5. Stack
A highly ordered, hardware-managed memory pool used for tracking function execution blocks. The Stack boundary expands **downward** toward lower memory addresses.

---

## The Stack Internals

The Stack operates as a strict **Last In, First Out (LIFO)** data structure. Data is added via a `PUSH` instruction and retrieved via a `POP` instruction. 

The stack space manages:
1. **Return Addresses:** Saved pointers allowing execution to jump back to a calling function once the current function finishes.
2. **Local Variables:** Storage for data arrays and variables scoped inside a function block.
3. **Saved CPU States:** Register data preserved across function calls.

!!! key "Stack Growth Direction"
    Because the stack grows downward toward lower memory addresses:
    * Executing a `PUSH` **decrements** the Stack Pointer (`RSP`).
    * Executing a `POP` **increments** the Stack Pointer (`RSP`).

### Stack Frames

A **Stack Frame** is a distinct block of memory on the stack allocated exclusively for a single function call. Its boundaries are traditionally managed by two registers: `RBP` (Base Pointer, marking the fixed start of the frame) and `RSP` (Stack Pointer, marking the fluid top of the frame).

---

## Calling Conventions

Since accessing RAM is significantly slower than executing operations within the CPU, 64-bit systems rely primarily on registers to pass arguments into functions quickly.

### System-V AMD64 ABI (Linux, macOS, BSD)
The first 6 arguments passed into a function are assigned to the following registers in order:

$$\text{1st: RDI} \quad \rightarrow \quad \text{2nd: RSI} \quad \rightarrow \quad \text{3rd: RDX} \quad \rightarrow \quad \text{4th: RCX} \quad \rightarrow \quad \text{5th: R8} \quad \rightarrow \quad \text{6th: R9}$$

If a function requires more than 6 arguments, the remaining parameters are pushed onto the stack in **reverse order** (right-to-left) right before the `CALL` instruction is executed.

### 32-Bit x86 Legacy Calling Conventions
On older 32-bit systems, registers are rarely used for parameter passing. Instead, **all** arguments are written directly onto the stack.
* **cdecl:** The calling function is responsible for cleaning up arguments from the stack after the function finishes.
* **stdcall (Common in Windows API):** The called function (the callee) is responsible for cleaning up the stack before returning.

### Stack Frame Linkage (Prologue & Epilogue)

Every time a function is called, it executes a standard structural sequence to link its new stack frame with the caller's frame:

#### The Function Prologue
Sets up the environment for the new function.
```
// push rbp          ; Save the old caller frame base pointer
// mov rbp, rsp      ; Set the new base pointer to the current top of the stack
// sub rsp, 0X20     ; Allocate local variable space (decrements RSP)
// [CODE BLOCK END]

#### The Function Epilogue
Restores the caller's environment right before exiting.
//
// mov rsp, rbp      ; Collapse the local stack space
// pop rbp           ; Restore the caller's base pointer
// ret               ; Pop the saved return address back into RIP
```

---

## Core Assembly Instruction Set

### NOP (No Operation)

An instruction that tells the CPU to do nothing for one clock cycle. It is represented by the opcode byte **0x90**.
* NOP is technically an architectural alias for `XCHG EAX, EAX`.
* It is commonly used for code alignment padding or to build robust "NOP sleds" within software exploit payloads.

### Push & Pop

* **PUSH:** Decrements `RSP` (by 8 bytes in 64-bit mode), then writes the target operand value to the memory location now pointed to by `RSP`.
* **POP:** Reads the value currently pointed to by `RSP`, writes it into the destination register, and then increments `RSP` (by 8 bytes in 64-bit mode). 

!!! warning "Pop Data Behavior"
    Executing a `POP` instruction **does not wipe or delete** the data from physical RAM. It merely shifts the `RSP` pointer upward, meaning the old value remains in dead memory until it is overwritten by a future stack operation.

### Memory Addressing Forms (`r/mX`)

In Intel syntax manuals, `r/mX` indicates that an operand can be either a raw **register** or a direct **memory address**. Memory dereferencing is denoted using square brackets `[]`, behaving exactly like standard pointer dereferencing in C.

There are four primary structural forms for memory addressing:

1. **Register:** Direct target tracking.
   * `rbx`
2. **Direct Memory Pointer:** Fetches the data stored at the memory address residing inside the register.
   * `[rbx]`
3. **Base + Index $\times$ Scale:** Ideal for standard array tracking.
   * Formula: $\text{Address} = \text{Base} + (\text{Index} \times \text{Scale})$
   * `[rbx + rcx * 4]` (Accesses a 4-byte integer array where `rbx` is the base pointer and `rcx` is the element index).
4. **Base + Index $\times$ Scale + Displacement:** Ideal for tracking arrays of data structures.
   * Formula: $\text{Address} = \text{Base} + (\text{Index} \times \text{Scale}) + \text{Displacement}$
   * `[rbx + rcx * 8 + 0x20]` (Accesses an object member variable located at offset `0x20` within an array of 8-byte structures).

### Data Sizing Qualifiers
When moving immediate values into raw memory pointers, you must explicitly specify the intended data size so the compiler knows how many bytes to write:
```
// mov qword ptr [rsp], 0xdeadbeef   ; Writes a 8-byte Quad Word
// mov dword ptr [rsp], 0x12345678   ; Writes a 4-byte Double Word
```

### Call & Ret

* **CALL:** Executes a functional jump. It first pushes the address of the *next* instruction (the return address) onto the stack, then changes the Instruction Pointer (`RIP`) to the target function's entry address.
* **RET:** Exits a function. It pops the top value off the stack directly back into `RIP`, returning execution control back to the caller.

### Syntax Standards: Intel vs AT&T

| Feature | Intel Syntax (NasM / Microsoft) | AT&T Syntax (GNU / GCC / Linux Default) |
| :--- | :--- | :--- |
| **Operand Order** | `Opcode Destination, Source` | `Opcode Source, Destination` |
| **Registers** | No Prefix (`rax`) | Requires `%` Prefix (`%rax`) |
| **Immediate Values** | No Prefix (`42`, `0x20`) | Requires `$` Prefix (`$42`, `$0x20`) |
| **Size Suffixes** | Explicit descriptors (`qword ptr`) | Appended opcode character (`movq`, `movl`) |

#### Practical Syntax Comparison
```
// ; Intel Syntax Representation
// mov rax, 42
// mov dword ptr [rbx], 0x10
//
// ; AT&T Syntax Representation
// movq $42, %rax
// movl $0x10, (%rbx)
```

### Data Arithmetic & Logic Instructions

#### MOV (Move)
Copies data from a source operand to a destination operand.
* **Crucial Architecture Rule:** x86-64 hardware **does not allow** direct memory-to-memory transfers within a single `MOV` instruction. To move data from one RAM location to another, you must route it through an intermediate CPU register.

#### ADD & SUB
Performs binary addition or subtraction, updating destination registers and setting processor status flags (`ZF`, `SF`, `CF`, `OF`).

#### IMUL (Signed Multiplication)
Multiplies signed integers. It features three operational formats:
1. `imul r/m64` $\rightarrow$ Multiplies `RAX` by the operand. Stores the 128-bit result split across `RDX:RAX`.
2. `imul r64, r/m64` $\rightarrow$ Multiplies the destination register by the source operand in place.
3. `imul r64, r/m64, imm8` $\rightarrow$ Multiplies the second operand by an immediate value, writing the result to the target register.

#### Data Extension Instructions
When moving smaller data types into larger registers, you must manage the remaining high-order bits:
* **Sign Extension (`MOVSX`):** Copies the original sign bit (the highest bit) across all the newly added upper bits. This preserves the accurate value of negative signed numbers.
* **Zero Extension (`MOVZX`):** Unconditionally fills all upper bits with `0`. This is used when casting unsigned data types.

#### LEA (Load Effective Address)
Though `LEA` uses square brackets (`[]`), it **never dereferences memory**. Instead, it calculates the raw mathematical address inside the brackets and saves that address directly to the destination register.
* Example: `lea rax, [rdx + rbx * 8 + 5]` reads as: $\text{rax} = \text{rdx} + (\text{rbx} \times 8) + 5$.
* It is frequently leveraged by optimizing compilers to run rapid, single-cycle algebraic math equations without altering CPU status flags.

#### TEST & CMP
* **CMP:** Subtracts the second operand from the first operand (identical to `SUB`), evaluates the outcome to alter CPU status flags, and then **discards the calculation result**.
* **TEST:** Performs a bitwise `AND` calculation between two operands, modifies status flags (`ZF`, `SF`), and **discards the calculation result**. Commonly used to check if a register is empty or null via `test rax, rax`.

#### Bitwise Shifts (`SHL`, `SHR`, `SAR`)
* `SHL` / `SHR` shift bits left or right, padding empty spaces with `0`. Optimizing compilers utilize these as fast alternatives for multiplying or dividing unsigned values by powers of two ($2^n$).
* `SAR` (Shift Arithmetic Right) shifts bits right but **retains the original sign bit** in the highest position, preserving negative values during division optimization.

#### String Operations (`REP STOS`, `REP MOVS`)
Instructions utilizing the `REP` prefix use the `RCX` register as an automated countdown loop mechanism.
* **REP STOS:** Fills a block of memory with a constant value (the assembly equivalent of `memset`). It writes data from `RAX` into the destination address pointed to by `RDI`, decrementing `RCX` on each iteration.
* **REP MOVS:** Copies a block of memory from a source to a destination (the assembly equivalent of `memcpy`). It reads data from `[RSI]` and writes it to `[RDI]`, looping for `RCX` iterations.

---

## CPU Status Flags & Control Flow

The **RFLAGS** (64-bit) or **EFLAGS** (32-bit) register contains individual bit status indicators updated automatically after arithmetic execution. Conditional jump instructions inspect these flags to branch execution.

* **Zero Flag (`ZF`):** Set to `1` if the output of an operation is exactly zero.
* **Sign Flag (`SF`):** Set to `1` if the output of an operation is negative (highest bit is a 1).
* **Carry Flag (`CF`):** Set to `1` if an unsigned calculation required a carry or borrow operation.
* **Overflow Flag (`OF`):** Set to `1` if a signed calculation output exceeded the register's limits.

### Conditional Jumps (`Jcc`)

Conditional jumps evaluate different combinations of flags based on whether you are analyzing **signed** or **unsigned** variables:

| Context | Jump Instructions | Meaning |
| :--- | :--- | :--- |
| **Generic** | `JE` / `JNE` | Jump if Equal (`ZF=1`) / Not Equal (`ZF=0`) |
| **Signed** | `JG` / `JL` | Jump if **Greater** / Jump if **Less** |
| **Unsigned** | `JA` / `JB` | Jump if **Above** / Jump if **Below** |

---

## Advanced Architecture Technicalities

### 32-Bit Register Zero-Extension Legacy
For structural legacy reasons, when you execute a 32-bit instruction that modifies a 32-bit register slice (like `EAX`), the CPU automatically **wipes the upper 32 bits of the parent 64-bit register (`RAX`) with zeroes**. 

Crucially, this automatic zero-extension **does not happen** when writing to 8-bit (`AL`) or 16-bit (`AX`) register slices. Security engineers leverage this quirk to find or construct gadget payloads during Return-Oriented Programming (ROP) exploitation.

### Stack Realignment Rules
According to the x86-64 Application Binary Interface (ABI) standard, the Stack Pointer (`RSP`) must always be **16-byte aligned** right before a `CALL` instruction is executed. If you observe a compiler injecting seemingly arbitrary addition or subtraction padding on `RSP` that doesn't relate to local variables, it is executing a mandatory stack pointer alignment.

### Windows x64 Calling Convention: The Shadow Store
Unlike Unix platforms, the Windows x64 calling convention allocates a mandatory **32-byte (4 Quad-Words) "Shadow Store"** on the stack directly preceding any function call. 

The caller must explicitly drop `RSP` to reserve this space even if the target function takes zero parameters. The called function utilizes this reserved shadow space to temporarily save the four primary incoming argument registers (`RCX`, `RDX`, `R8`, `R9`) if it needs to loop through variable argument structures (like `printf`).

---

## High-Level Code Translation Examples

### Array Construction & Access Optimization

Consider this standard C code snippet:
```
// short a;
// int b[6];
// a = 0xBABE;
// b[1] = a;
```

An optimizing compiler translates the indexing calculation ($\text{Offset} = \text{Index} \times \text{Sizeof(Type)}$) into a combination of sign extensions and direct displacements:

```
// mov eax, 0xFFFFBABE        ; Loads the immediate value into EAX (Sign-extended short)
// mov [rbp - 0x10 + 4], eax  ; Writes directly to array address b[1] (Offset 4 bytes from base)
```

### Struct Static Offset Mapping

Data structures are mapped linearly within memory based on the size of their elements. Take this C structure:

```
// struct User {
//     int id;         // Offset +0x0 (Size: 4 bytes)
//     int role;       // Offset +0x4 (Size: 4 bytes)
//     char pass[8];   // Offset +0x8 (Size: 8 bytes)
// };
```

When interacting with an instance of this structure via a base pointer inside `RBX`, the disassembler reveals direct static displacements mapping to each member variable layout:

```
// mov eax, [rbx]        ; Accesses User.id (Offset 0)
// mov ecx, [rbx + 0x4]  ; Accesses User.role (Offset 4)
// mov rdx, [rbx + 0x8]  ; Accesses User.pass (Offset 8)
```