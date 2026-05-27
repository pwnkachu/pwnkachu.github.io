---
title: Understanding the ELF (Executable and Linkable Format)
description: A deep dive into the structure of ELF binaries for binary exploitation.
---

The **ELF (Executable and Linkable Format)** is the universally accepted standard file format for executables, object code, shared libraries, and core dumps in Unix-like operating systems across multiple architectures (x86, x64, ARM, MIPS, etc.). Its comprehension is vital for the security exploitation of programs in the Unix environment. To give a direct comparison, the `.exe` format is the equivalent of ELF in Windows.

A modern executable is not formed merely by a raw block of machine language code. Instead, the file is structurally divided into multiple parts used to store metadata, OS mapping instructions, relocation data, and dynamic linking information. 

Executables are not the only type of ELF files you will encounter. Other common types include:

- **Shared Object files (`.so`)**: Dynamically loaded libraries.

- **Relocatable files (`.o`)**: Compiled object code waiting to be linked.

- **Core dump files**: Snapshots of a program's memory and system state at the time of a crash.

---

## Structure of an ELF File

An ELF binary is divided into four main macro-components. The **Segments** contain information needed for runtime execution, while **Sections** usually contain specific information needed for linking and relocation.

| Component | Main Role |
| :--- | :--- |
| **ELF Header** | Contains the primary metadata and blueprint of the file. |
| **Program Header Table** | Tells the kernel *how* to map the file into memory. Essential for execution. |
| **Sections / Segments** | The actual payload (code, data, variables, symbols). |
| **Section Header Table** | Tells the linker where to find specific logical sections (e.g., `.text`, `.data`). Mostly used for static analysis and debugging. |

??? info "Reference Link"
    [Visualizing ELF Structure](https://santhosh-duraipandiyan.medium.com/elf-executable-and-linkable-format-1448baa49afb)

---

### ELF Header
To visualize the main header of an ELF file, we can execute the command `readelf -h <elf_name>`, which outputs metadata similar to this:

```text
ELF Header:
  Magic:   7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00 
  Class:                             ELF64
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0
  Type:                              EXEC (Executable file)
  Machine:                           Advanced Micro Devices X86-64
  Entry point address:               0x400920
  Start of program headers:          64 (bytes into file)
  Start of section headers:          11744 (bytes into file)
  ...
```

!!! success "Key Takeaway"
This header provides valuable information for analysis tools. For instance, the Type field shows EXEC, telling us that the ELF is compiled without PIE (Position Independent Executable). If PIE were enabled, the Type header would display DYN (Shared object file).

### Program Headers (Segments)

Program headers can be obtained with the command ``` readelf -l <elf_name>```. This output represents the execution view of the binary:

```text
Elf file type is EXEC (Executable file)
Entry point 0x400920
There are 9 program headers, starting at offset 64

Program Headers:
  Type           Offset             VirtAddr           PhysAddr
                 FileSiz            MemSiz              Flags  Align
  ...
  LOAD           0x0000000000000000 0x0000000000400000 0x0000000000400000
                 0x00000000000012d4 0x00000000000012d4  R E    0x200000
  LOAD           0x0000000000001e10 0x0000000000601e10 0x0000000000601e10
                 0x0000000000000298 0x0000000000000320  RW     0x200000
  ...
  GNU_STACK      0x0000000000000000 0x0000000000000000 0x0000000000000000
                 0x0000000000000000 0x0000000000000000  RW     0x10

 Section to Segment mapping:
  Segment Sections...
   ...
   02     .interp .note.ABI-tag .gnu.hash .dynsym .dynstr .init .plt .text .rodata ...
   03     .init_array .dynamic .got .data .bss
```

The top part shows the segments the OS checks to map memory. For each segment, we can see its memory permissions (Readable, Writable, Executable).

    
- The LOAD segments are the actual chunks copied into memory.

- The mapping section at the bottom shows how the linker grouped sections into these segments. For example, .text (which contains machine code) is mapped into the 02 segment with Read/Execute (R E) flags.

- The GNU_STACK segment is flagged as RW (lacking the E). This confirms that the NX (Non-Executable) security protection is enabled.

### Section Headers

The section headers can be printed using ```readelf -S <elf_name>```. This provides detailed information regarding the logical divisions of the program:

```text
Section Headers:
[Nr] Name              Type             Address           Offset
    Size              EntSize          Flags  Link  Info  Align
[12] .plt              PROGBITS         0000000000400800  00000800
    0000000000000110  0000000000000010  AX       0     0     16
[14] .text             PROGBITS         0000000000400920  00000920
    0000000000000662  0000000000000000  AX       0     0     16
...

```

- Address: The virtual address where the section will be loaded during execution. Notice that the .text Address frequently matches the binary's Entry Point.

- Offset: The exact physical byte location of the section's data from the start of the file on the disk.

??? info "Note on Memory Virtualization"
    The OS abstracts physical RAM addresses into a virtual address space, giving the process the illusion that it has access to a dedicated, contiguous block of memory. The CPU's Memory Management Unit (MMU) translates these virtual addresses to physical RAM addresses in real-time using page tables.

### Crucial Sections for Pwn

While Segments describe the runtime memory layout, Sections are the logical representations of the binary to analyze inside tools like Ghidra, IDA, or Radare2. Here are the most important ones for binary exploitation:

#### .text (The Code)

Contains the actual CPU Assembly instructions. It is mapped with Read and Execute (R-X) permissions.

#### .plt/.plt.sec & .got (Symbol Resolution)

These two sections work together to handle dynamic linking and are fundamental for techniques like Ret2Libc:

!!! info "How PLT and GOT Interact"
* .plt (Procedure Linkage Table): A table of "jump trampolines." When the code calls an external library function (like printf), it jumps to the PLT first.
* .got (Global Offset Table): A table of absolute addresses. It holds the actual resolved pointers to dynamic library functions (inside libc) at runtime. Overwriting a pointer in this section (GOT Overwrite) allows an attacker to hijack the control flow.

#### .data & .bss (Variables)

- .data: Contains initialized global and static variables (e.g., int a = 5;).

- .bss: Contains uninitialized global variables. This section has Read/Write (RW-) permissions and is highly useful in standard pwn challenges as a stable memory region to inject strings (like /bin/sh) via an Arbitrary Write primitive.

#### Metadata Sections

These are highly useful if the binary has not been stripped:

- .symtab (Symbol Table): Stores the names, addresses, and sizes of all functions and variables.

- .strtab (String Table): Stores the raw text strings used to name the symbols in the program.

- .debug_*: Contains raw debugging information generated by the compiler.

??? info "Stripped Binary"
    A stripped binary is a file which non essential information has been removed, typically debugging data, symbol tables, relocation informations and metadata.