---
title: PwnDbg Commands
description: Useful pwndbg commands for binary exploitation and debugging.
---

# PwnDbg Reference Guide

### Execution & Breakpoints
Control the flow of the program and stop execution at critical points.

* **`b func` / `b *0xdeadbeef`**: Sets a breakpoint at a specific function name or an exact memory address (use `*` for addresses).
* **`finish`**: Continues execution and stops right after the current function returns.
* **`watch *addr`**: Sets a hardware watchpoint. The debugger will pause execution the moment the data at this specific address is modified.
* **`aslr on/off`**: Toggles Address Space Layout Randomization for the current debugging session.

### Context & Code Inspection
Analyze the state of registers, variables, and assembly instructions.

* **`disass`**: Disassembles the current function or a specified address.
* **`nearpc n`**: Displays `n` assembly instructions around the current Program Counter (PC), providing immediate execution context.
* **`print var`**: Evaluates and prints the value of a specific variable, register, or expression.
* **`set $reg = value`**: Modifies the value of a CPU register or memory address on the fly (e.g., `set $rax = 0`).

### Memory & Stack Inspection
Explore memory layout, stack contents, and locate specific data.

* **`telescope addr/reg num`** (e.g., `tele $rbp-160 41`): A smart memory dump that automatically dereferences pointers to show strings, known addresses, or other pointers.
* **`x/20wx $esp`**: The standard GDB command to examine memory. This specific example prints 20 words in hex starting from the stack pointer.
* **`search -P "/bin/sh"`**: Searches the entire memory space for a specific string or pattern.
* **`search -T dword 0xdeadbeef`**: Searches memory for a specific data type and value.

### Binary & Security Analysis
Inspect binary mitigations, memory mappings, and imported libraries.

* **`checksec`**: Displays the security mitigations compiled into the binary (Canary, NX, PIE, RELRO).
* **`canary`**: Automatically finds and prints the current value of the stack canary.
* **`got` / `plt`**: Displays the Global Offset Table or Procedure Linkage Table, useful for ret2libc or GOT overwrite attacks.
* **`vmmap` / `mappings`**: Shows the memory map of the current process, including segments (like libc) and their permissions (Read/Write/Execute).
* **`info proc`**: Displays general information about the current process, including its PID.
* **`libc`**: Prints the base memory address where the C standard library (`libc`) is currently loaded.
* **find_fake_fast**: Displays memory addressed that have a suitable fake size for a fastbin poison attack.

### Exploit Development Toolkit
Tools specifically designed to calculate offsets and build payloads.

* **`cyclic n`**: Generates a De Bruijn sequence (a non-repeating pattern) of length `n` to help find buffer overflow offsets.
* **`cyclic -l value`**: Calculates the exact offset by finding the location of a specific `value` (or faulting address) within the generated pattern.
* **`retaddr`**: Scans the current stack and prints all the return addresses it can find.

### Heap Exploitation
Analyze dynamically allocated memory chunks and metadata.

* **`heap`**: Displays general information about the heap arenas and currently allocated chunks.
* **`bins`**: Shows the current state of the heap bins (tcache, fastbins, unsorted, small, large), crucial for Use-After-Free or Double Free attacks.
* **`vis`**: Visually maps out the heap layout, showing chunk boundaries, sizes, and metadata in an easy-to-read format.


## Examining Memory in GDB/pwndbg

The `x` command is the standard tool for inspecting raw memory without relying on source code data types.

### Command Syntax
`x/nfu <address>`

* **n (Number):** How many units to display.
* **f (Format):** Display format (`x` for hex, `d` for decimal, `s` for string, `i` for instruction).
* **u (Unit Size):** Size of each unit (`b` for 1 byte, `h` for 2 bytes, `w` for 4 bytes, `g` for 8 bytes).

### Example Breakdown
`x/32xb 0x602110`

This translates to: "Examine 32 units, formatted as hexadecimal, where each unit is 1 byte, starting at `0x602110`".

**Output structure:**
`0x602110 <u+16>:    0x75    0x73    0x65    0x72`
* `0x602110`: The starting memory address.
* `<u+16>`: Symbol resolution (an offset of 16 bytes inside a known variable `u`).
* `0x75 0x73...`: The raw bytes stored at those addresses, read sequentially from left to right.

Because x86-64 is little-endian, a sequence of bytes is read backwards when evaluated as a larger integer. 

For example, the byte sequence:
`0x30  0x35  0x32  0x32  0x37  0x00  0x00  0x00`
is read by the CPU (and `glibc`) as the 64-bit integer **`0x3732323530`**, not `0x30`. 
To pass a fastbin check for `0x30` (in case of size check), the byte sequence must be:
`0x30  0x00  0x00  0x00  0x00  0x00  0x00  0x00`
    modify_note(r, 1, p64(USERNAME_ADDR))